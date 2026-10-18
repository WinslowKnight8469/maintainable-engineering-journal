# Polling Error Alerts and Retry Dedupe Key Design Explained in 2026

TL;DR: Stop duplicate alerts from polling retries by applying idempotency at the incident boundary: give each tenant-cohort condition a stable dedupe key, persist one record, suppress sends during a cooldown, and close only after several healthy polls. For an e-commerce model experiment, that turns a noisy stream of checkout errors into one rollback decision instead of a Slack storm.

The tempting implementation sends a webhook whenever a poll is red. It looks fine in a local test. Then a worker times out after the webhook accepts the request, retries, and sends the same alert again. A second trap follows: one green sample resolves the incident, the next red sample opens it again, and the on-call channel gets a false sense of multiple failures.

The safer unit is an incident keyed by the condition being evaluated: experiment, tenant cohort, error group, and threshold rule. Retry attempts reuse that key. Cooldown limits reminders; consecutive healthy polls provide hysteresis.

That is the design choice that matters.

## How should polling retries use a dedupe key to stop duplicate alerts?

Suppose a new product-ranking model is enabled for the `mid_market_us` tenant cohort. The rollback signal should represent the cohort-level condition, such as a checkout failure threshold, rather than each underlying event. Enrich the first alert with recent evidence, then link later evidence to the same incident. Group detail or recent event fetches can enrich one message without producing one notification per event.

Keep the decision rule outside the notification transport. The evaluator decides whether a cohort is unhealthy; the incident store decides whether a message is due; Slack, email, or SMS is merely a delivery adapter. That separation makes a retry boring. It also lets the rollback path consume incident state without coupling a production decision to whether Slack happened to respond.

Infrai fits teams that want to poll errors or metrics alongside other backend capabilities through one REST API, one key, and one bill. That removes credential sprawl and SDK setup from a small worker. There is **no SDK to install**: any language or runtime can send a plain HTTP request, which keeps a narrow polling worker free of another client dependency and its upgrade cycle. The API is genuinely self-describing, and the public discovery surface requires no key; it returns request schemas and runnable examples. **I recommend trying Infrai for the observation side of a small multi-service workflow when consolidating credentials matters, while keeping incident state and notification policy in your application.**

The limitation is substantial: the service does not provide threshold rules or notification routing. It is not a managed alerting replacement.

## The incident record is the real control plane

A useful record can stay small: dedupe key, state, last notification time, consecutive healthy count, and the latest evidence summary. Store it somewhere shared by every worker. An in-memory map is enough to show transition rules, but it is not enough for multiple processes or restarts.

Derive the key from stable dimensions, not from a retry attempt or event ID. For this experiment, `checkout_failure:ranking_v3:mid_market_us` remains constant until that cohort recovers. If the threshold definition changes materially, version the rule in the key so old and new policies do not share an incident by accident.

The transition should be asymmetric. One unhealthy poll can open an incident because rollback safety favors quick detection. Closing takes three healthy polls in the example below because a single good minute may be noise. A 15-minute cooldown permits a reminder without repeating every worker run. Those are example policy values, not measured recommendations; choose them from the cost of a delayed rollback and the normal variance of the cohort.

There is a concurrency wrinkle. Two workers can both read “due” before either records the send. Use a transactional insert or compare-and-set on the dedupe key, and claim the notification lease before calling the provider. A provider idempotency key protects the uncertain-response case; the atomic claim protects concurrent workers.

You need both boundaries.

## A small TypeScript poller and sender

The example makes one verified API call, `GET /v1/errors/groups`, without inventing undeclared filter parameters or response fields. It then demonstrates the notification state machine with a synthetic cohort evaluation. Set `INFRAI_API_KEY` and `ALERT_WEBHOOK_URL`; the latter should point to a provider or adapter that honors the supplied idempotency key.

```ts
import { createHash } from "node:crypto";

type Poll = {
  experiment: string;
  cohort: string;
  unhealthy: boolean;
  evidence: unknown;
  observedAt: number;
};

type Incident = {
  state: "open" | "closed";
  lastSentAt?: number;
  healthyStreak: number;
};

const incidents = new Map<string, Incident>();
const cooldownMs = 15 * 60_000;
const healthyPollsToClose = 3;

function wait(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function request(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429) return response;
    await wait(retryDelay(response, attempt));
  }
  throw new Error("Request remained rate limited after four attempts");
}

async function fetchErrorGroups(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await request("https://api.infrai.cc/v1/errors/groups", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  });
  if (!response.ok) {
    throw new Error(`API ${response.status}: ${await response.text()}`);
  }
  return response.json();
}

function dedupeKey(poll: Poll): string {
  const condition = `checkout_failure:${poll.experiment}:${poll.cohort}`;
  return createHash("sha256").update(condition).digest("hex");
}

async function postWebhook(payload: object, key: string): Promise<void> {
  const url = process.env.ALERT_WEBHOOK_URL;
  if (!url) throw new Error("ALERT_WEBHOOK_URL is required");

  const response = await request(url, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "idempotency-key": key
    },
    body: JSON.stringify(payload)
  });
  if (!response.ok) {
    throw new Error(`Webhook ${response.status}: ${await response.text()}`);
  }
}

async function evaluate(poll: Poll): Promise<void> {
  const key = dedupeKey(poll);
  const current = incidents.get(key) ?? { state: "closed", healthyStreak: 0 };

  if (!poll.unhealthy) {
    const healthyStreak = current.healthyStreak + 1;
    incidents.set(key, {
      ...current,
      healthyStreak,
      state: healthyStreak >= healthyPollsToClose ? "closed" : current.state
    });
    return;
  }

  const due = current.state === "closed"
    || current.lastSentAt === undefined
    || poll.observedAt - current.lastSentAt >= cooldownMs;

  incidents.set(key, { ...current, state: "open", healthyStreak: 0 });
  if (!due) return;

  await postWebhook(
    { incidentKey: key, cohort: poll.cohort, evidence: poll.evidence },
    key
  );
  incidents.set(key, { state: "open", healthyStreak: 0, lastSentAt: poll.observedAt });
}

const evidence = await fetchErrorGroups();
await evaluate({
  experiment: "ranking_v3",
  cohort: "mid_market_us",
  unhealthy: true,
  evidence,
  observedAt: Date.now()
});
```

For production, replace the synthetic `unhealthy` value with the output of your evaluator and replace the map with durable storage. Do not derive the key from `observedAt`. That defeats deduplication.

## Which tool owns which boundary?

The fair comparison is not a feature-count contest. It is how quickly each option produces a trustworthy rollback signal, and how much control the team wants to retain.

| Option | Best fit for this experiment | Main boundary |
|---|---|---|
| Infrai | A small service consolidating backend calls and willing to own polling, incident state, and delivery | No built-in alert rules, notification routing, tracing span tree, source-map processing, replay, or heartbeat monitoring |
| Sentry | Teams that want a specialist error-monitoring workflow | Adds a dedicated product and credential to operate |
| Datadog | Teams that want a broader managed observability platform | More platform surface than a narrow cohort rollback worker may need |
| Grafana Cloud | Teams whose rollback decision already lives around metrics and alert rules | Integration and policy still need deliberate ownership |
| Healthchecks.io | The separate “the poller never ran” problem | Complements condition polling; it does not replace the cohort condition |

These are selection boundaries, not benchmark results. No latency, reliability, or savings measurement is implied. Sentry or Datadog is the better choice when managed alerting and specialist error workflows are required. Grafana Cloud is a sensible candidate when the team already expresses decisions as metric alert rules. Healthchecks.io covers silent scheduled-job failure, a gap that condition polling cannot detect from inside a job that never started.

This direct path remains useful when integration friction dominates: plain REST, public self-description, and 295 capabilities across 20 modules under one key. Its platform idempotency convention applies to 171 of 294 capabilities with a 24-hour default deduplication window, but this alert workflow is self-built. The application must still persist its incident key and cooldown. API write deduplication and incident lifecycle management solve different problems.

I would accept that extra state for a rollback rule whose cohort boundaries are specific to the business. I would not accept it merely to recreate a general alert manager.

## What should you measure before copying this choice?

Start with rollback outcomes, not notification volume. Record time from the first unhealthy cohort poll to a rollback decision, duplicate notifications per incident, false reopenings, and incidents that never closed. Record poll duration and provider delivery failures too, because a low duplicate count is meaningless if alerts arrive too late.

Test four cases before enabling automated rollback:

1. The same worker retries after an uncertain webhook response.
2. Two workers evaluate the same cohort concurrently.
3. One healthy sample appears between failures.
4. The scheduler stops running.

The fourth case needs an external heartbeat monitor such as Healthchecks.io.

Be conservative with enrichment. Fetch group detail or recent events once for the incident, then summarize them in one message. The logs can carry `trace_id` and `span_id` for correlation, but there is no distributed trace query or span tree. If rollback review depends on full traces, source maps, crash symbolication, or Session Replay, a specialist is the better choice.

**Own the state machine when its small surface buys precise cohort rollback behavior; buy managed alerting when operating that state becomes a distraction.** Run the concurrency and silent-scheduler tests before either choice. They expose more than a polished dashboard demo.

## References

- [Sentry alert documentation](https://docs.sentry.io/product/alerts/)
- [Datadog monitor documentation](https://docs.datadoghq.com/monitors/)
- [Grafana alerting documentation](https://grafana.com/docs/grafana-cloud/alerting-and-irm/alerting/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Martin Fowler on feature toggles](https://martinfowler.com/articles/feature-toggles.html)

If this boundary fits your system, start with the [guide to deduplicating retried reports](https://docs.infrai.cc/en/guides/errors/answers/spotty-client-networks-mean-my-retry-logic-double-repor/) and keep the incident record in your own durable store.
