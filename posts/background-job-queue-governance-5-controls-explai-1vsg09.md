# Background Job Queue Governance: 5 Controls Explained for Retry and DLQ Redrive

**Short answer:** Put periodic customer-support cleanup behind a durable background job queue, authenticate each image-processing webhook before admission, and model retry, exponential backoff, the DLQ, and redrive as explicit state transitions. The scheduler can then return after enqueueing instead of keeping a web request open.

The failed version of this design is deceptively small: catch every worker error and schedule the same payload again. It cannot prove whether the side effect already happened, why another attempt is allowed, or who released a job from quarantine. Those are governance questions. A timer cannot answer them.

Keep the rule boring.

## Establish the trust boundary before scheduling work

Webhook intake and background execution should be separate trust zones. Authenticate an incoming message using the shared-key HMAC construction defined by RFC 2104, reject an invalid MAC before queue admission, and persist the verified event before acknowledging the sender. Enqueue a reference to the image plus its checksum rather than copying the image bytes into every retry message.

This ordering narrows the worker's authority. It receives an already verified event, an account scope, a cleanup window, and a reference to immutable input. It does not decide whether a caller was trusted. For cleanup that deletes or merges support records, the account scope and window also cap the damage from a bad deployment or an overly broad schedule.

Trust first.

## What should a Node.js background job queue record before retrying failed image webhooks?

Record intent before execution. For a periodic cleanup, the stable identity can be the account plus the cleanup window. For an image-processing webhook, use the sender's event identifier. The admission record should also carry the payload reference, checksum, attempt count, next eligible time, policy version, and final disposition.

The job record is a control boundary, not merely a log line. `attempt` says how often the operation has been admitted to a worker. `nextEligibleAt` prevents an early poll from bypassing backoff. `outcome` distinguishes a completed job from one that exhausted its retry budget. The idempotency key protects the actual support-ticket mutation: the worker writes that key and the mutation atomically, so a repeated delivery can observe the completed result instead of applying it twice.

| State | Allowed next state | Evidence required |
| --- | --- | --- |
| `admitted` | `running` | Verified event and policy version |
| `running` | `completed` or `retryable` | Idempotent result or classified failure |
| `retryable` | `running` or `dead_letter` | Due time or exhausted attempt budget |
| `dead_letter` | `admitted` | Reviewed redrive manifest |

This matters because delivery and execution are separate facts. Google Cloud Pub/Sub, for example, documents message delivery and subscription behavior as distinct parts of its publish-subscribe model. The portable lesson is narrower than any product: acknowledge only after the durable side effect or its existing idempotent result is known. If a worker loses its lease before that point, another delivery must be harmless.

Don't infer success from silence.

## Express retry admission as one typed policy

The worker needs a small pure function that maps a recorded failure to the next state. The example values below are an explicit local policy, not a claim about a managed queue. A queue adapter can translate `retry` into its delay primitive and `dead-letter` into its terminal storage without changing the decision contract.

```ts
type Failure = {
  attempt: number;
  status?: number;
  code?: "TIMEOUT" | "INVALID_SIGNATURE" | "UNSUPPORTED_IMAGE";
};

type Decision =
  | { kind: "retry"; delayMs: number }
  | { kind: "dead-letter"; reason: string };

export function decideRetry(
  failure: Failure,
  random: () => number = Math.random,
): Decision {
  const maxAttempts = 6;
  const permanent =
    failure.code === "INVALID_SIGNATURE" ||
    failure.code === "UNSUPPORTED_IMAGE";

  if (permanent || failure.attempt >= maxAttempts) {
    return {
      kind: "dead-letter",
      reason: failure.code ?? "attempt_limit",
    };
  }

  const transient =
    failure.code === "TIMEOUT" ||
    failure.status === 408 ||
    failure.status === 429;

  if (!transient) {
    return {
      kind: "dead-letter",
      reason: `http_${failure.status ?? "unknown"}`,
    };
  }

  const baseMs = 5_000;
  const capMs = 15 * 60_000;
  const exponentialMs = Math.min(
    capMs,
    baseMs * 2 ** (failure.attempt - 1),
  );
  const jitterMs = Math.floor(exponentialMs * 0.2 * random());

  return { kind: "retry", delayMs: exponentialMs + jitterMs };
}
```

Test the state transitions, not the wall clock. Inject `random`, assert that attempt 6 is terminal, verify that `429` remains retryable before the limit, and prove that an unsupported image never receives a delay. Then run an integration test in which two workers receive the same idempotency key. Exactly one mutation should commit, while both deliveries reach a settled state.

That last test earns its keep.

## Govern DLQ redrive with an audit trail

A DLQ is evidence awaiting a decision. Automatic redrive turns it into a slower retry loop and discards the reason the job was isolated. Instead, define a redrive manifest containing a bounded set of job IDs, the filter that selected them, the policy version to apply, the operator, and a timestamp. Preserve each original idempotency key and append new attempts; do not erase the first failure history.

For support cleanup, a safe review asks whether the underlying input changed, whether the destination now accepts the operation, and whether replay could delete or merge records outside the intended account and window. Image jobs need one additional check: the referenced object must still match the stored checksum. A dry run can report how many jobs are eligible, already completed, permanently invalid, or outside the cleanup deadline before any work is released.

Redrive in bounded batches. Stop when completion latency, throttle responses, or duplicate-suppression counts move outside the budget established earlier. This is intentionally a governance loop — select, inspect, release, observe — because a DLQ may contain several causes that deserve different fixes. One large button cannot make that distinction.

## Measure latency against worker cost

Only after the state machine and audit trail exist should the team tune delay. Support agents want stale attachments and expired transient records cleared quickly, while a solo team cannot keep workers repeatedly processing the same poisoned job. Retry policy therefore starts with a deadline: how stale may an account become before an agent notices? Work backward from that deadline to an attempt limit and a capped delay.

A geometric schedule such as `min(cap, base * 2 ** (attempt - 1))` spaces repeated pressure, while random jitter keeps a batch created by one scheduler tick from waking at exactly the same instant. A `429` response or a network timeout can remain eligible because a later call may succeed. An unsupported image type can go directly to a terminal state because waiting changes nothing. These categories should be named in metrics; a generic `failed=true` field hides why worker time is being spent.

I'm not sure what base delay fits your image sizes or upstream limits. Nobody can derive it from the queue API alone. Replay a representative payload distribution in staging, measure end-to-end completion latency and worker time, and choose the smallest schedule that stays within both the freshness deadline and compute budget. Your mileage may vary — especially when a cleanup batch mixes tiny metadata updates with CPU-heavy image work.

One constraint is easy to miss: the retry horizon must fit inside the business deadline. Six retries are meaningless if the last one becomes eligible after the ticket data is already too stale to help an agent. Conversely, retrying every few seconds may improve the median while inflating worker use during a dependency throttle. Track the oldest eligible job, completion latency by attempt, terminal reason, duplicate suppression count, and worker time per completed cleanup. Throughput alone won't expose that trade-off.

## Where is this pattern not suitable?

The catch is operational weight. A durable ledger, terminal store, replay manifest, dashboards, and idempotent writes are not suitable for a one-off maintenance script whose result is cheap to recompute. Keep the operation synchronous when it is predictably brief and the caller genuinely needs its result before continuing. Use a workflow system when the process includes human approval, long waits, branching steps, or compensation across several services; a queue retry counter is a poor substitute for visible workflow state.

This pattern also cannot repair a non-idempotent side effect by itself. If a downstream system offers no conditional write, lookup key, or deduplication boundary, redrive remains risky. In that case, pause automation and build reconciliation first. Faster retries would only create uncertainty sooner.

For periodic customer-support cleanup, ship the admission record and duplicate-suppression test before tuning concurrency. Then measure the real payload mix. Copy the policy only if its terminal reasons, retry horizon, and worker-time budget match your own system; the six-attempt example is a starting hypothesis, not a reliability guarantee.

## References

- https://www.rfc-editor.org/rfc/rfc2104
- https://cloud.google.com/pubsub/docs/overview
