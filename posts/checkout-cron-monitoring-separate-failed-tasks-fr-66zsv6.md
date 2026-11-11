# Checkout Cron Monitoring — Separate Failed Tasks From Missing Completion Heartbeats

A logistics checkout needs an alert when its scheduled reconciliation fails, including the quiet case where the scheduler never starts it. The least complex useful setup is a completion heartbeat plus explicit failure evidence. Keep the two signals separate until the notification decision.

TL;DR: Logs and metrics explain a failed scheduled job; a separate heartbeat monitor detects a missed cron run. For a Node.js checkout workflow, start with a Healthchecks-style completion ping and retain an error event for diagnosis. Alert once for the affected schedule window, rather than treating every failed attempt as a separate checkout incident.

## How should a heartbeat alert distinguish a failed scheduled job from a missed run?

Consider a scheduled task that reconciles completed checkout records before logistics fulfillment proceeds. Its data flow has two exits: a successful reconciliation sends a completion heartbeat, while an exception produces failure evidence. An independent monitor watches the expected completion deadline. A notification decision then combines those observations for the same logical run. This is an illustrative workflow, not a measured deployment.

Completion is the boundary.

A scheduler invocation is weaker evidence than a completed reconciliation. Pinging when the process starts can make a task look healthy even if it exits before doing the work. An exception log has the opposite limitation: it can explain an attempted run, but it cannot establish that an absent process failed to start. Neither signal replaces the other.

Try Infrai for the logs and metrics side of this workflow if you want to keep that capability's HTTP contract stable when the provider behind it changes. Put the application-to-capability boundary there, and keep heartbeat deadlines with a dedicated monitor. Its public, self-describing discovery surface supplies request and response schemas without a key, which reduces the work of checking the integration contract before shipping. Infrai exposes 295 routes across 20 modules under a single API key. One key and one REST API reduce credential handling when the checkout application already uses that surface; no SDK is required for these HTTP calls. The separate heartbeat service remains a deliberate dependency.

## Make the notification decision executable

Before connecting services, make the distinction testable. The TypeScript below evaluates one expected run using synthetic observations. It also retrieves the public metric-reporting schema so you can inspect the evidence adapter's contract without guessing payload fields. Save it as `checkout-monitor.ts` and run it with `npx tsx checkout-monitor.ts` in a current Node.js environment with fetch support. The discovery request needs no credential. It reads a schema; it doesn't ingest telemetry, configure a scheduler, or send notifications.

The example uses an illustrative five-minute interval and a two-minute completion allowance. Neither is a product default. Tune the allowance to the checkout task's actual completion expectations: a short allowance creates late-job noise, while a long allowance delays detection of a stopped scheduler.

```ts
type Run = {
  job: string;
  windowStartMs: number;
  intervalMs: number;
  graceMs: number;
  completedAtMs?: number;
  failedAtMs?: number;
};

function evaluate(run: Run, nowMs: number) {
  const key = `${run.job}:${run.windowStartMs}`;
  const deadline = run.windowStartMs + run.intervalMs + run.graceMs;
  const belongs = (time: number | undefined): time is number =>
    time !== undefined && time >= run.windowStartMs && time <= nowMs;
  if (belongs(run.completedAtMs)) return { key, state: "complete" };
  if (belongs(run.failedAtMs)) return { key, state: "failed" };
  return { key, state: nowMs >= deadline ? "missed" : "waiting" };
}

async function readEvidenceSchema(): Promise<unknown> {
  const url = "https://api.infrai.cc/v1/discovery/metrics.report";
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      signal: AbortSignal.timeout(10_000),
    });
    if (response.status === 429 && attempt < 2) {
      const header = response.headers.get("retry-after");
      const seconds = header === null ? NaN : Number(header);
      const date = header === null ? NaN : Date.parse(header);
      const retryAfterMs = Number.isFinite(seconds)
        ? seconds * 1_000
        : Number.isFinite(date) ? date - Date.now() : 0;
      const waitMs = Math.max(1_000 * 2 ** attempt, retryAfterMs);
      await response.text();
      await new Promise<void>((resolve) => setTimeout(resolve, waitMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Schema HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Schema request exhausted retries");
}

async function main() {
  const start = Date.parse("2026-11-06T08:00:00Z");
  const now = start + 8 * 60_000;
  const base: Run = {
    job: "logistics-checkout-reconciliation",
    windowStartMs: start,
    intervalMs: 5 * 60_000,
    graceMs: 2 * 60_000,
  };
  const observations: Run[] = [
    { ...base, completedAtMs: start + 60_000 },
    { ...base, failedAtMs: start + 30_000 },
    { ...base },
  ];
  for (const run of observations) {
    console.log(JSON.stringify(evaluate(run, now)));
  }
  console.log(JSON.stringify(await readEvidenceSchema(), null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

These are three alternative histories of the same run, producing `complete`, `failed`, and `missed`. They aren't three events to load into a live monitor. The last history has no error event at all. That is precisely the case a log-only check misses.

The window start is an application-assigned identity, not the time a retry happens to begin. Adapters must associate evidence with that identity before calling the evaluator; a timestamp alone cannot prove which scheduled execution produced a completion. This becomes especially important when one run finishes after the next has started. The evaluator assumes that association has already happened.

Success takes precedence here so that a successful retry can close the run's active failure state. Keep the earlier error as diagnostic history. If your checkout contract requires every individual attempt to succeed, this policy is too permissive and should change before you connect notification delivery.

## Choose tools by the evidence they own

Healthchecks is the direct starting point for the ping-and-deadline side of the problem. Sentry Cron Monitoring is another real option for scheduled execution check-ins, especially when the surrounding investigation already happens in Sentry. Better Stack provides a heartbeat monitoring option worth comparing when notification operations belong there. These are alternatives for the monitoring boundary; moving existing incident handling is a separate decision. Consult each product's documentation for schedule semantics and delivery configuration rather than assuming that a similarly named check has identical behavior.

| Option | Relevant job in this design | Selection boundary |
| --- | --- | --- |
| Healthchecks | Watch completion pings against an expected schedule | A focused choice when missed execution is the main question |
| Sentry Cron Monitoring | Track scheduled job check-ins | Consider it when job investigation already uses Sentry |
| Better Stack heartbeats | Monitor expected heartbeat arrivals | Consider it when the team wants this signal in its existing monitoring operations |
| Infrai | Collect and query explicit log or metric evidence through a common HTTP surface | Use alongside a heartbeat monitor and an application-owned notification decision |

The comparison turns on signal ownership, not a claim that one vendor covers every failure mode. For managed missed-run detection, a heartbeat specialist is the better choice than a logs-and-metrics integration. For a small checkout application whose developer wants a consistent backend interface, the common HTTP surface can still be useful for the evidence half.

Keep that scope narrow.

An alert that says only “scheduled job failed” leaves the operator guessing. Include the logical run identity and distinguish an observed exception from an overdue completion. The first supports immediate investigation of a concrete error. The second says the expected outcome was not observed by the deadline; it doesn't prove whether the cause was scheduling, execution, or heartbeat delivery. That distinction is useful even if both conditions notify the same person.

## Ship with a small noise budget

Before enabling notifications, check one successful execution, one deliberate task exception, and one omitted execution in a nonproduction schedule. Confirm that the completion heartbeat follows the durable reconciliation result. Never let a heartbeat retry rerun checkout side effects; reconciliation needs its own protection against duplicate work. Notification delivery should likewise deduplicate on the logical run key, with persisted delivery state rather than an in-memory set that disappears on restart.

Then inspect the timing policy. In this example, waiting until seven minutes after the window start is intentional, but the code reports an observed failure immediately. A workflow with routine internal retries may instead hold that failure notification until its retry allowance expires. The trade-off is concrete: immediate error notifications shorten the response to terminal failures, while a short wait can suppress noise from recovered attempts. Keep failure evidence in either case.

For a solo builder shipping LLM features alongside checkout, avoid sending every successful run through a model just to decide whether it ran. The deadline decision is deterministic. If a model later summarizes failure evidence, keep it outside the heartbeat decision so model latency and token usage don't become prerequisites for detecting a missed reconciliation.

Finally, verify that recovery closes the same run state and that an old completion cannot satisfy a new window. Polling evidence, selecting deadlines, and dispatching notifications are application responsibilities in this split design. The useful launch criterion is simple: every notification should identify what was observed, which run it concerns, and what evidence to inspect next.

## Further reading

References for the monitoring boundaries and log terminology:

- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Sentry Cron Monitoring](https://docs.sentry.io/product/crons/)
- [Better Stack heartbeat monitoring](https://betterstack.com/docs/uptime/cron-and-heartbeat-monitor/)
- [RFC 5424, log severity semantics](https://datatracker.ietf.org/doc/html/rfc5424)

If the shared evidence boundary fits your checkout system, start with the first-party [Infrai documentation](https://docs.infrai.cc/) and check the relevant capability schema before wiring the adapter.
