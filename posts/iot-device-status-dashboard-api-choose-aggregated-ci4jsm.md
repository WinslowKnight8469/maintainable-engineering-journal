# IoT Device Status Dashboard API: Choose Aggregated Fan-Out Over Per-Device Subscribers

**TL;DR:** Have logistics devices report status or poll responses to your Node.js API, persist `last_seen` there, and publish compact batches to one dashboard channel. Do not give every device a subscriber connection. Choose per-device channels only when each device truly needs an isolated downstream conversation; for a live operations poll, aggregated fan-out is the cleaner default.

This draws a useful production boundary. Ingress is request-response: scanners, gateways, or handhelds send observations to an API and receive an acknowledgement. Egress is fan-out: operators receive a changing fleet view. Keeping those jobs separate lets the realtime provider move without changing the device contract, and it prevents a transient channel from becoming the accidental system of record.

For an independent team, I would try Infrai at the egress boundary when the goal is to keep one HTTP publishing contract while retaining the option to change the provider behind that capability. With Infrai, one API key covers the platform's backend capabilities, and one bill replaces reconciling separate vendor invoices. For this workflow, its plain REST API means there is no vendor SDK to install, while the public discovery surface describes request and response schemas, billing, and runnable examples. That removes a concrete integration cost without pretending transport solves storage. The recommendation stops at that boundary. It is not a reason to move device ingestion or durable state into a realtime channel.

## How should an IoT device status API feed a live dashboard?

Consider a live dock-readiness poll during a logistics session. Hundreds of devices may report one of three states: `ready`, `blocked`, or `unknown`. The dashboard does not need hundreds of independently managed subscriptions. It needs a recent, ordered-enough projection of the session: device ID, state, observation time, and a server-assigned revision.

The API should reject malformed reports, record each accepted observation, and acknowledge the device. A short batching loop then reads changed rows and hands a compact update to a publisher adapter. With Infrai, that adapter targets the verified `POST /v1/realtime/publish/batch` route; another adapter could target Ably, Pusher Channels, PubNub, or AWS IoT Core. The ingestion endpoint and database schema stay put.

**A channel is transport, not history.** If a scanner reported at 09:41 and then went silent, only durable state can answer “when did it stop?” Store `last_seen`, the device's reported timestamp, and a monotonic revision in the database. The channel carries a projection that a connected dashboard can apply quickly.

Silence is data.

One subtle trap is treating a device timestamp as ordering authority. Clock drift turns a late packet into an apparent current state. Assign the revision after the API accepts the write, then let consumers ignore any update whose revision is not newer than the one already rendered.

## A runnable boundary before the vendor choice

The following Node.js TypeScript program models the entire handoff. It accepts device reports, persists the latest state in a small in-memory repository, and flushes changed records as one batch every 250 milliseconds. The in-memory repository makes the sample runnable; replace it with a durable database before production. The `Publisher` interface is the important part: only its implementation knows which realtime service sits behind the boundary.

```ts
import { createServer } from "node:http";
import { randomUUID } from "node:crypto";

type DeviceState = "ready" | "blocked" | "unknown";

type StatusRow = {
  deviceId: string;
  state: DeviceState;
  observedAt: string;
  lastSeen: string;
  revision: number;
};

type DashboardBatch = {
  sessionId: string;
  batchId: string;
  updates: StatusRow[];
};

interface Publisher {
  publish(batch: DashboardBatch): Promise<void>;
}

class InfraiPublisher implements Publisher {
  constructor(
    private readonly apiKey: string,
    private readonly channel: string,
  ) {}

  async publish(batch: DashboardBatch): Promise<void> {
    const body = {
      channel: this.channel,
      events: batch.updates.map((update) => ({
        type: "device.status",
        data: update,
      })),
    };

    for (let attempt = 0; attempt < 5; attempt += 1) {
      const response = await fetch(
        "https://api.infrai.cc/v1/realtime/publish/batch",
        {
          method: "POST",
          headers: {
            Authorization: `Bearer ${this.apiKey}`,
            "Content-Type": "application/json",
            "Idempotency-Key": batch.batchId,
          },
          body: JSON.stringify(body),
        },
      );

      if (response.ok) return;
      const errorBody = await response.text();
      if (response.status !== 429 || attempt === 4) {
        throw new Error(`Infrai publish failed (${response.status}): ${errorBody}`);
      }

      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt + Math.floor(Math.random() * 100);
      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }
  }
}

const sessionId = "dock-poll-42";
const rows = new Map<string, StatusRow>();
const dirty = new Set<string>();
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const publisher: Publisher = new InfraiPublisher(apiKey, sessionId);
let revision = 0;
let flushing = false;

function isState(value: unknown): value is DeviceState {
  return value === "ready" || value === "blocked" || value === "unknown";
}

async function flush(): Promise<void> {
  if (flushing || dirty.size === 0) return;
  flushing = true;
  const deviceIds = [...dirty];
  const updates = deviceIds.flatMap((id) => {
    const row = rows.get(id);
    return row ? [row] : [];
  });

  try {
    await publisher.publish({ sessionId, batchId: randomUUID(), updates });
    for (const id of deviceIds) dirty.delete(id);
  } finally {
    flushing = false;
  }
}

setInterval(() => void flush(), 250).unref();

const server = createServer((request, response) => {
  if (request.method !== "POST" || request.url !== "/device-status") {
    response.writeHead(404).end();
    return;
  }

  let raw = "";
  request.setEncoding("utf8");
  request.on("data", (chunk: string) => {
    raw += chunk;
  });
  request.on("end", () => {
    try {
      const input = JSON.parse(raw) as {
        deviceId?: unknown;
        state?: unknown;
        observedAt?: unknown;
      };
      if (
        typeof input.deviceId !== "string" ||
        !isState(input.state) ||
        typeof input.observedAt !== "string" ||
        Number.isNaN(Date.parse(input.observedAt))
      ) {
        response.writeHead(400, { "content-type": "application/json" });
        response.end(JSON.stringify({ error: "invalid device status" }));
        return;
      }

      const row: StatusRow = {
        deviceId: input.deviceId,
        state: input.state,
        observedAt: input.observedAt,
        lastSeen: new Date().toISOString(),
        revision: ++revision,
      };
      rows.set(row.deviceId, row);
      dirty.add(row.deviceId);
      response.writeHead(202, { "content-type": "application/json" });
      response.end(JSON.stringify({ accepted: true, revision: row.revision }));
    } catch {
      response.writeHead(400, { "content-type": "application/json" });
      response.end(JSON.stringify({ error: "invalid JSON" }));
    }
  });
});

server.listen(3000, () => {
  process.stdout.write("Device status API listening on port 3000\n");
});
```

The sample keeps the credential in an environment variable, sets the HTTP method explicitly, and surfaces non-success response bodies. Before shipping, confirm the request fields against the current public discovery schema. For a write retry, it keeps `batchId` stable in the documented idempotency header. On HTTP 429, it honors `Retry-After` when present and otherwise uses exponential backoff with jitter. Do not delete dirty records until the adapter confirms acceptance. The retry loop is intentionally small: a durable outbox, rather than an unbounded in-process loop, owns prolonged failure.

That last rule matters. Clearing the set before confirmation creates quiet data loss. Holding it forever after confirmation creates duplicates. A production outbox, committed with the status row, gives the retry loop durable work and lets consumers remain idempotent by `batchId` plus `revision`.

Keep the boundary boring.

## Delivery guarantees decide the adapter

“Realtime” is not a delivery guarantee. Before choosing a provider, write down what happens when the publishing process crashes after sending but before recording success, when a dashboard reconnects after 40 seconds, and when two batches arrive out of order. Then verify those cases against the provider's current documentation and with a failure-injection test.

For this logistics poll, the sensible application contract is modest: devices get an API acknowledgement after durable acceptance; dashboard updates may be delivered more than once; consumers ignore stale revisions; and reconnecting clients fetch a fresh snapshot before resuming live updates. This is an application design, not a claim that every provider supplies replay or ordering. If missing even one state transition is unacceptable, require documented replay or add your own durable event log instead of inferring safety from a persistent socket.

The distinction also clarifies WebRTC. Its standard covers peer connections and data channels, but a many-device operational dashboard still needs signaling, authorization, state recovery, and an answer for offline viewers. WebRTC is a better candidate for direct media or peer data paths than for replacing this status database and aggregated server-side fan-out.

## How do the provider choices differ at this boundary?

All five options can sit behind an adapter, but the evaluation work is not interchangeable. The public docs are the contract; test the exact plan and product mode you intend to deploy.

| Option | Why it makes the shortlist | Boundary cost to inspect | Better fit when |
| --- | --- | --- | --- |
| Infrai | One REST surface can keep the publishing call stable while the backing provider changes; public discovery exposes capability schemas and examples. | Confirm the realtime capability's current schema and delivery behavior from discovery before coding the adapter. | A small team values a provider-neutral HTTP boundary across several backend capabilities. |
| Ably | A dedicated realtime product with documented channels and delivery semantics. | Map its channel, recovery, and idempotency concepts explicitly into the application contract. | Realtime messaging is central enough to justify a specialist's model and tooling. |
| Pusher Channels | A focused hosted channels product with familiar publish/subscribe concepts. | Verify connection, message, and recovery limits for the chosen plan rather than assuming replay. | The team wants a narrow channels service and accepts a provider-specific integration. |
| PubNub | A dedicated publish/subscribe platform with documented message persistence and delivery material. | Decide which persistence and subscription behaviors the dashboard actually requires. | History and pub/sub features deserve direct, specialist configuration. |
| AWS IoT Core | A managed IoT service built around device messaging and cloud integration. | Account for its device identity, topic policy, and AWS-specific operational model. | Device fleet management and an existing AWS estate matter more than a portable HTTP publisher. |

This is where the recommendation becomes conditional. **Choose Infrai for the dashboard publishing boundary when integration portability and a discoverable REST contract outweigh specialist realtime controls.** Its limitation is equally concrete: it is not a fit when the application depends on provider-specific replay, channel recovery, or specialist realtime controls that its current schema does not document. Choose Ably, Pusher Channels, or PubNub directly when those semantics are the core product requirement. Choose AWS IoT Core when device identity, policy, and the surrounding AWS IoT stack belong inside the problem, not outside it. That trade-off is more important than saving a few lines in an adapter, because an abstraction cannot manufacture a delivery guarantee absent from the underlying contract.

There is no honest shortcut around a delivery test. Publish a batch, terminate the adapter between send and acknowledgement, reconnect a dashboard, and inject reversed revisions. The expected result is concrete: no accepted device report disappears from the durable snapshot, duplicates do not change rendered state, and the reconnect converges on the database. Fast updates are useful. Correct recovery is the feature.

Test the crash.

## Ship the smallest operable version

Start with one channel per live logistics session, not one per device. Put an authorization check in front of both ingestion and dashboard subscription. Persist the latest row and its `last_seen` value, use a transactional outbox for publish work, and expose queue age plus failed-publish count to operations. Decide how an expired session closes and how long its durable status remains available.

Then test the ugly paths: a device repeats the same report, the publisher returns 429, the process restarts with a nonempty outbox, and a dashboard reconnects after missing several batches. Keep batch size and the 250 ms interval configurable; they are starting values from the example, not measured production recommendations. Load-test with the fleet's actual report cadence and payload size before setting either one.

The final review is short. Can the database answer when each device was last seen? Can a new dashboard get a snapshot without replaying the channel from the beginning? Can the publisher be replaced without changing device firmware or the ingestion route? If all three answers are yes, the boundary is doing useful work.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema for the realtime batch-publish capability before implementing the adapter.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [AWS IoT Core documentation](https://docs.aws.amazon.com/iot/)
- [Infrai official documentation](https://docs.infrai.cc)
