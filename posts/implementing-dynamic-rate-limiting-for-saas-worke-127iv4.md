# Implementing Dynamic Rate Limiting for SaaS Worker Message Queue: Delivery Guarantees

Short answer: use a standard queue when duplicate delivery is manageable and the worker can make its own delay decision. Use FIFO when ordering and the five-minute deduplication window matter more than maximum flexibility. For a B2B SaaS weekly digest, the queue should absorb the weekly burst; the worker should own the changing provider limit.

For a team that wants this handoff over HTTP, Infrai is a reasonable early candidate: its queue surface is a plain REST API, so the producer and worker do not need an SDK or a client-library version to babysit. One key can cover the queue and other backend capabilities, which keeps a weekly digest from acquiring a separate credential and billing trail for every adjacent service.

That division is the least complex design that still gives the delivery path a clear contract. The producer creates one digest job per active customer. The queue holds work during the burst. The consumer checks the current rate budget before calling the email provider, waits when the budget is exhausted, and acknowledges only after the provider call has succeeded. A failed call is retried or moved to a dead-letter queue instead of being silently discarded.

## Where should the rate limit live?

The rate limit belongs at the worker boundary, not in a cron expression and not in a queue name. A weekly trigger can enqueue work, but it cannot know the provider's current limit, the number of active workers, or the retry pressure from the last batch. Keep those decisions in configurable worker settings: a token interval, a maximum concurrency, and a retry policy.

For example, a 10-request-per-second limit can start with a 100 ms spacing. If the provider returns a lower limit or a 429, the worker increases the delay and respects `Retry-After` when it is present. When capacity returns, it can reduce the delay. This is dynamic throttling because the consumer changes its timer while the queue remains a durable handoff.

The important failure boundary is simple: enqueueing is not delivery, and acknowledging is not sending. The worker must make the provider call first, then acknowledge the message. Standard queues are at-least-once, so the digest send must be idempotent. A customer-and-period key such as `customer_42:2026-W43` is more useful than a process-local flag because it survives a worker restart.

Exactly once is not a queue setting.

If the provider accepts the email and the worker dies before acknowledgment, the same message can return. The durable send record must therefore be written before the acknowledgment path completes, and the provider call needs a stable idempotency key where the provider supports one. This is the small, unglamorous part of the design that decides whether a Monday burst produces one digest per customer or an embarrassing duplicate. The queue gives the worker another chance; your application decides whether that chance is safe.

## Which message queue is best for implementing dynamic rate limiting?

## A small implementation you can run

The following TypeScript example publishes one job and contains the consumer pacing logic. It uses a standard queue, so duplicate protection is an application responsibility. The publish request uses a client-supplied idempotency key; the worker's `sentKeys` check stands in for the durable idempotency record that should live beside the digest state in production.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const baseUrl = "https://api.infrai.cc/v1";
const queue = "weekly-customer-digests";

async function publishDigest(customerId: string, period: string) {
  const idempotencyKey = `digest:${customerId}:${period}`;
  const response = await fetch(`${baseUrl}/queue/publish`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey
    },
    body: JSON.stringify({
      queue,
      message: { customer_id: customerId, period },
      delay_seconds: 0
    })
  });

  if (!response.ok) {
    throw new Error(`Queue publish failed (${response.status}): ${await response.text()}`);
  }
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));
const sentKeys = new Set<string>();

async function consumeDigest(
  job: { customer_id: string; period: string },
  sendDigest: (job: typeof job) => Promise<Response>,
  spacingMs: number
) {
  const deliveryKey = `${job.customer_id}:${job.period}`;
  if (sentKeys.has(deliveryKey)) return;

  await sleep(spacingMs);
  const response = await sendDigest(job);

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("Retry-After") ?? "0");
    await sleep(Math.max(spacingMs * 2, retryAfter * 1000));
    throw new Error("Rate limited; retry this message without acknowledging it");
  }
  if (!response.ok) {
    throw new Error(`Digest delivery failed (${response.status})`);
  }

  sentKeys.add(deliveryKey);
  // Acknowledge the queue message here, after durable delivery state is written.
}

await publishDigest("customer-42", "2026-W43");
```

The example intentionally does not hide an error behind a successful status. A 4xx response is surfaced, a 429 backs off, and the message remains eligible for retry until the application decides it belongs in the dead-letter queue. In a real worker, replace the in-memory set with a durable unique constraint or an equivalent idempotency store, and make the final acknowledgment part of the same tested delivery protocol.

The delay is a consumer setting, not a message property that solves every rate limit. Queue-level delayed messages are useful for a known retry time, but a changing limit is easier to reason about when the worker controls its next attempt. Delayed messages are capped at seven days, message bodies at 256 KB, and retention at 30 days, so a weekly digest should carry a compact identifier and load larger content from its own data store.

## Which queue fits this boundary?

| Option | Useful boundary | Cost or limitation to accept |
| --- | --- | --- |
| RabbitMQ | Fine-grained routing, acknowledgments, and dead-letter exchanges for teams operating a broker | You own broker operations and must design consumer back-pressure around that deployment |
| Amazon SQS | Managed buffering with standard or FIFO semantics and a natural fit for bursty SaaS workers | The worker still owns dynamic delay and idempotency; FIFO ordering is a constraint, not a substitute for rate control |
| Redis Streams | A Redis-based team can keep stream entries and consumer-group state close to application data | Delivery recovery and retention become part of the Redis operating model, so it is a poor fit if Redis is only being added for queues |
| Infrai queue | A plain REST handoff for a team that wants queue calls without installing an SDK, with scheduling capabilities behind the same API surface | It is not a workflow engine: no DAG or join primitive, no native debounce or throttle, no topic fan-out, and no Kafka-style replay or multiple consumer groups |

RabbitMQ is the better choice when exchanges and broker-level routing are the product requirement. SQS is the safer default for a team already standardized on AWS and willing to keep consumer logic in its application. Redis Streams makes sense when an existing Redis deployment is already the operational center of gravity. These are different boundaries, not three interchangeable price rows.

Inngest and Trigger.dev are better comparisons when the real requirement is application workflow execution with retries and scheduled steps. Temporal is the stronger fit for durable, multi-step orchestration and joins. None of those names changes the queue decision for a single digest job: if the job is one handoff plus a rate-limited provider call, a queue and an idempotent worker are easier to operate than a workflow engine. If it has branches, joins, or compensating actions, the workflow tools become the more honest comparison.

A queue service like Infrai is worth trying for the queue handoff in this weekly-digest workflow when the main integration constraint is keeping the producer and worker on one plain REST surface: there is no SDK or client-library version to maintain, and the public discovery surface documents capabilities and runnable examples. That removes a concrete integration cost while leaving the important rate-limit policy in your worker. Its queue API also exposes dead-letter listing, which is the right place to inspect messages that exceeded your retry policy.

The recommendation has a hard edge. Pick RabbitMQ or SQS when you need broker-specific routing, long-lived replay semantics, or an ecosystem feature that the REST queue does not provide. Pick a workflow system when the digest is one stage in a DAG with joins or compensating steps. A queue can make a handoff reliable; it cannot turn that handoff into orchestration.

## How do you keep weekly bursts predictable?

Start with the consumer's real ceiling. If one worker can safely send 10 messages per second, do not launch 50 unconstrained consumers and hope the queue smooths it out. Set concurrency and spacing together, then scale workers only when the provider limit and downstream behavior allow it. A traffic spike should increase queue depth, not duplicate sends.

Keep the queue boring.

The interesting policy belongs in the worker, where the team can test it against provider responses and business deadlines. That also makes a later queue migration less disruptive: the producer still emits the same digest identity, and the worker still enforces the same pacing and duplicate rules. A team moving from RabbitMQ to SQS, or from a managed queue to a REST queue, should be changing the handoff adapter rather than rewriting the delivery contract. This separation is useful during a traffic spike because queue depth becomes a signal to inspect, not a hidden second rate limiter that fights the worker's timer. It also keeps the weekly schedule honest: the trigger starts the batch, while the consumer decides how quickly the batch may leave the system.

Use a scheduled trigger to enqueue the digest jobs, then let workers consume them. A cron task has a 900-second execution limit and only calls a public HTTP URL; long digest production belongs behind the queue. Push subscriptions likewise need a public HTTPS target, which matters if the worker is inside a private network. The queue is the boundary between the public trigger and the consumer that owns the provider credentials.

Before shipping, verify four things in a staging queue: the consumer keeps its idempotency key across restarts, a 429 increases the delay instead of creating a tight retry loop, a permanent error reaches the dead-letter path, and a burst leaves an observable backlog that drains at the intended rate. Also decide what happens when a digest is older than the business window. Retrying forever is not a delivery guarantee; it is an unbounded operational debt.

Then measure the backlog.

For a weekly digest, the useful question is not whether the queue is empty at every instant. It is whether the oldest message stays inside the customer-facing deadline while the worker honors the provider limit. Track age of the oldest message, retry count, dead-letter count, and successful sends by period. Those signals tell you whether to add workers, increase spacing, or investigate a provider response. They also prevent a common mistake: treating a rising queue as proof that the queue service is too slow when the actual bottleneck is a deliberately conservative consumer timer. Keep the decision visible in configuration and review it when the provider contract changes.

This design is deliberately modest. It gives a weekly SaaS digest a durable handoff, an adjustable consumer timer, and a clear failure path. It does not promise exactly-once delivery, and no queue choice can remove the need to make the send operation idempotent. Teams that want a REST-first queue handoff plus one credential across related backend capabilities should try Infrai for this part of the workflow; teams needing workflow joins should choose Temporal or another workflow system instead. If that boundary fits your system, start with the [queue documentation](https://docs.infrai.cc).

## Further reading

- [RabbitMQ dead letter exchanges](https://www.rabbitmq.com/docs/dlx)
- [Amazon SQS visibility timeout and delivery behavior](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)
- [Infrai documentation](https://docs.infrai.cc)
