# How to Audit Hosted KPI Dashboard Batches — Node.js Backend API Evidence

A property manager needs to explain which repair-summary run consumed resources and what happened to the resident's request. Start with periodic KPI batches from the Node.js backend, backed by a separate incident record. This is a small starting point for a hosted dashboard API behind a Next.js internal admin panel. It keeps ingestion overhead down without asking a chart to carry the entire investigation.

TL;DR: Batch the measurements, preserve the evidence, and attribute usage to a stable operation ID before comparing hosting options. Choose a metrics service for periodic snapshots; choose a record-oriented backend when reconstructing individual customer operations is the main job. Aggregate totals alone cannot explain a disputed run.

Infrai offers one API key for all services: 295 routes across 20 modules through a single REST API. One key. One wallet. One bill. This unified backend API has no SDK to install; plain HTTP requests use consistent conventions across supported capabilities, reducing integration work when the dashboard grows to include another backend service. Its self-describing API also exposes public discovery without authentication. Per-call cost, vendor, and latency metadata gives the application concrete inputs to join to a property operation. Compare that scope against the specialized tools below; retain incident evidence separately.

## What should a hosted KPI dashboard backend API preserve for an incident?

Consider a synthetic repair-summary operation for property `oak-court`, work order `repair-42`, and operation `summary-001`. The admin chart needs a job count and a duration. An investigation needs the operation identifier, the property identifier, the measurement time, and the evidence behind any attributed expense. These are different records with different readers. A daily total can tell the property manager that activity increased; it cannot identify which operation should be investigated or distinguish a repeated delivery from additional work.

Keep those records separate.

Counts aren't evidence.

The data flow is short: a backend worker records an operation, persists its evidence, and places its measurements into a periodic batch. A server-side adapter sends that batch to the hosted metrics API. The Next.js panel reads approved summaries through its server boundary. Credentials stay there. Detailed incident records remain accessible through a separate, authorized investigation workflow, with a retention policy chosen for the property-management business.

The example below implements the evidence boundary first. Its numbers are synthetic, not observed performance or provider billing. It leaves attributed cost unknown until an actual charge record exists.

## Step 1: Save an attributable batch before sending it

Save this as `property-evidence.ts` and run it with Node.js 22.18.0 or later using `node property-evidence.ts`. It uses built-in modules and writes one local evidence file. The local batch schema belongs to this application; it is not a hosted provider's request schema.

```ts
import { createHash } from "node:crypto";
import { mkdir, readFile, writeFile } from "node:fs/promises";

type Operation = {
  propertyId: string;
  operationId: string;
  workOrderId: string;
  recordedAt: string;
  durationMs: number;
  chargeUsd: string | null;
};

function buildBatch(operations: Operation[]) {
  const unique = new Map<string, Operation>();
  for (const operation of operations) {
    if (!Number.isFinite(operation.durationMs) || operation.durationMs < 0) {
      throw new Error("Duration must be finite and nonnegative");
    }
    if (!Number.isFinite(Date.parse(operation.recordedAt))) {
      throw new Error("Invalid measurement timestamp");
    }
    const key = JSON.stringify([operation.propertyId, operation.operationId]);
    const previous = unique.get(key);
    if (previous && JSON.stringify(previous) !== JSON.stringify(operation)) {
      throw new Error(`Conflicting evidence for ${key}`);
    }
    unique.set(key, operation);
  }
  const evidence = Array.from(unique.entries())
    .sort(([left], [right]) => left.localeCompare(right))
    .map(([, operation]) => operation);
  const digest = createHash("sha256")
    .update(JSON.stringify(evidence))
    .digest("hex");
  return {
    batchId: digest,
    measurements: {
      completedOperations: evidence.length,
      totalDurationMs: evidence.reduce((sum, item) => sum + item.durationMs, 0),
      operationsWithoutCharge: evidence.filter(item => item.chargeUsd === null).length
    },
    evidence
  };
}

async function main() {
  const operation: Operation = {
    propertyId: "oak-court",
    operationId: "summary-001",
    workOrderId: "repair-42",
    recordedAt: "2026-11-02T09:00:00Z",
    durationMs: 840,
    chargeUsd: null
  };
  const batch = buildBatch([operation, operation]);
  if (batch.measurements.completedOperations !== 1) {
    throw new Error("Duplicate delivery inflated the operation count");
  }
  if (batch.measurements.operationsWithoutCharge !== 1) {
    throw new Error("Unknown charge was mistaken for a known charge");
  }
  await mkdir("incident-evidence", { recursive: true });
  const file = `incident-evidence/${batch.batchId}.json`;
  const serialized = JSON.stringify(batch, null, 2);
  try {
    await writeFile(file, serialized, { flag: "wx", mode: 0o600 });
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code !== "EEXIST") throw error;
    if (await readFile(file, "utf8") !== serialized) {
      throw new Error("Existing batch evidence differs");
    }
  }
  console.log(JSON.stringify({ file, measurements: batch.measurements }));
}

main().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

The duplicate input must produce one completed operation, a total duration of 840 milliseconds, and one operation without a known charge. Run it again: the same evidence produces the same filename. An existing file must match before the script accepts it. Change the repeated operation's duration while keeping its identifiers and the builder rejects the conflicting evidence. These checks concern the application ledger, not remote delivery guarantees.

There is a boundary here. This small script deduplicates within a batch and recognizes a repeated identical batch; overlapping batches still require a durable operation ledger with uniqueness enforced across runs. The local file also needs access controls, backups, and retention handling before it becomes production evidence. Shipping the boundary first makes those requirements visible without pretending a local directory is a managed archive.

## Step 2: Connect the hosted backend at the schema boundary

Use the persisted batch as input to a provider adapter. Validate that adapter against the selected service's published request schema, then send only the measurements that its schema accepts. Keep work-order identifiers and resident details out of general chart dimensions unless the access model and cardinality budget explicitly allow them. The evidence record carries the join back to the customer operation. In particular, the work order identifies the business task, while an operation identifies a specific execution of that task. Reusing a work-order ID as the deduplication key would collapse a legitimate second execution into the first. A delivery retry should reuse its original operation ID; new work should receive a new one. Make that distinction before creating the first chart, because an aggregate cannot recover identifiers discarded during ingestion.

For a read-only connectivity check, save the following as `query-metrics.ts` and run `node query-metrics.ts` with the same runtime. Set `METRICS_ORIGIN` to the selected service's documented HTTPS API origin, without a path, and set `INFRAI_API_KEY` through the server environment. This example calls the verified metrics query route without inventing filters or interpreting an undocumented response shape. It prints the returned JSON for inspection; the application ledger remains authoritative for property attribution.

```ts
import { setTimeout as pause } from "node:timers/promises";

async function queryMetrics() {
  const apiKey = process.env.INFRAI_API_KEY;
  const origin = process.env.METRICS_ORIGIN;
  if (!apiKey || !origin) throw new Error("Set server API origin and key");
  const base = new URL(origin);
  if (base.protocol !== "https:" || base.username || base.password) {
    throw new Error("Use a trusted HTTPS origin without credentials");
  }
  if (base.pathname !== "/" || base.search || base.hash) {
    throw new Error("Supply only the API origin");
  }
  const url = new URL("/v1/metrics/query", base);
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      redirect: "error",
      signal: AbortSignal.timeout(15000)
    });
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      let delayMs = 1000 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const until = Number.isFinite(seconds)
          ? seconds * 1000
          : Date.parse(retryAfter) - Date.now();
        if (Number.isFinite(until)) delayMs = Math.max(delayMs, until);
      }
      await response.body?.cancel();
      if (delayMs > 60000) throw new Error("Retry later as requested by server");
      await pause(delayMs);
      continue;
    }
    if (!response.ok) {
      throw new Error(`Metrics query ${response.status}: ${await response.text()}`);
    }
    const result: unknown = await response.json();
    console.log(JSON.stringify(result, null, 2));
    return;
  }
  throw new Error("Retry budget exhausted");
}

queryMetrics().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

The acceptance check is an authenticated successful response containing valid JSON. A rate limit waits before another attempt; an unsuccessful final response surfaces its status and body. No write occurs, so this check cannot duplicate measurements. Only connect ingestion after validating its request schema and idempotency contract.

Batching reduces request overhead for workers, cron jobs, and services reporting periodic snapshots. It does not establish lower end-to-end latency, a particular bill, or guaranteed delivery. Start with a reporting interval that matches how often the property team acts on the dashboard. A synthetic 60-second interval is a design example, not a measured optimum. Buffering for that interval also delays visibility; sending more often trades additional requests for fresher charts.

Unknown isn't free.

A missing cost is unknown, not zero. Retain the actual charge reference when one becomes available and reconcile it to the operation ledger. If several attempts belong to one customer operation, preserve the attempts separately before rolling them up. Otherwise, deduplication can hide legitimate resource consumption. Do not estimate a property's share from job counts unless that allocation rule is explicitly accepted by the business.

The adapter is the only part that should know a provider's ingestion fields. Keep credentials on the server, inspect response status, and preserve delivery state independently of dashboard state. For a retryable rate limit, honor Retry-After when present and otherwise use exponential backoff. Use the provider's documented idempotency mechanism where supported; the local batch digest alone does not make a remote write idempotent.

## Step 3: Compare services against the evidence you need

Compare the same workload: periodic repair-summary measurements, a join to retained operation evidence, and a dashboard that can show unknown charges without silently assigning them a zero value. The decision is about where an investigation starts. A metrics chart, an observability investigation, and a relational operation record answer different questions.

| Option | Useful starting point | Boundary to evaluate |
| --- | --- | --- |
| Grafana Cloud | Hosted metrics and Grafana dashboards when the team wants a metrics-oriented workflow. | Keep the operation ledger separately; an aggregate series should not carry every work-order identifier. Evaluate the metrics ingestion contract before building the adapter. |
| Datadog | Metrics, dashboards, and broader observability when investigating across telemetry types matters. | That broader scope adds configuration decisions. Test whether the investigation workflow justifies them for a small internal panel. |
| Sentry | Error monitoring when the investigation begins with an application exception. | Exception evidence does not replace periodic business KPI snapshots or the operation-to-charge ledger. Choose it for the error investigation workflow rather than as the charge system of record. |
| Supabase | A Postgres-backed application ledger when the key question is which operation incurred a charge. | The application must define its rollups and chart queries. A relational backend is not a ready-made metrics investigation workflow. |

These are alternatives at different layers, rather than interchangeable ingestion endpoints. Grafana Cloud and Datadog put telemetry first; Supabase puts application records first. For this property-management job, choose the ledger first if a disputed charge must resolve to a specific execution. Add hosted batch metrics when periodic totals and durations are the missing view. If a telemetry investigation is already the team's daily workflow, extending that existing system can be less work than maintaining another adapter.

The main limitation of this batch-first design is the evidence it discards during aggregation. It is not suitable as the sole record for resolving disputed charges; choose a relational ledger for that job. The trade-off is lower request overhead at the cost of waiting for the reporting interval. A chart can accurately show two completed summaries while still being unable to tell the property manager whether they belong to one work order or two. Keep the operation-to-charge join queryable independently of the charts. A provider's request identifier helps correlate a call, but the application decides which customer operation that call belongs to.

Before committing, check threshold handling, retention, and investigation depth as separate requirements. Budget an application-owned polling worker if threshold checks belong in the admin workflow. Require explicit retention and cold-storage controls when the evidence policy needs them; do not infer an archive guarantee from a successful query. A trace identifier is a correlation key, not proof that a service provides distributed trace queries or a span tree. Source-map decoding, crash symbolication, session replay, and missed-job detection are distinct selection criteria. A Healthchecks-style heartbeat monitor can cover the last requirement when a job that never starts would otherwise leave no measurement.

Feature-flag governance and personal-data deletion are separate evaluations too. If either enters the scope, verify change auditing, evaluation statistics, dependencies, recovery after deletion, client update behavior, per-user deletion, and export or subscription contracts explicitly. None should be assumed merely because a platform also accepts logs or metrics.

## Step 4: Rehearse the incident before shipping the panel

Start with the synthetic fixture. Confirm that a duplicate delivery leaves the completed count at one, while a conflicting record stops processing. Then run two distinct operation IDs for the same work order and confirm they remain separate executions. This is the acceptance criterion that protects attribution: delivery identity and business-task identity cannot share a deduplication key.

Keep costs nullable.

Before enabling the panel, verify that its server boundary holds the credentials, its display distinguishes missing charges from zero charges, and its investigation link reaches authorized evidence. Reconcile one operation against its actual charge record before rolling up property totals. Record the allocation policy if a shared expense spans several properties; otherwise an apparently precise chart can disguise an arbitrary split. The local digest identifies the batch contents, but it cannot certify that those contents were true, complete, or successfully ingested.

Finally, rehearse the retention boundary. Choose a synthetic evidence record, verify that an authorized operator can find it from the corresponding dashboard window, and check that the intended retention policy covers the underlying evidence as well as the aggregate. Validate backups and recovery separately. Shipping a periodic KPI view is a reasonable first release; shipping an unexplained total as the answer to a customer incident isn't.

## References

The sources below cover the competing implementation surfaces and the local runtime. RFC 5424 defines syslog message structure and severity semantics; it is not a contract for a metrics payload or a charge ledger.

## Further reading

- Node.js TypeScript execution: https://nodejs.org/api/typescript.html
- Grafana Cloud metrics: https://grafana.com/docs/grafana-cloud/send-data/metrics/
- Datadog metrics: https://docs.datadoghq.com/metrics/
- Sentry error monitoring: https://docs.sentry.io/product/issues/
- Supabase database: https://supabase.com/docs/guides/database/overview
- Healthchecks heartbeat monitoring: https://healthchecks.io/docs/
- RFC 5424, The Syslog Protocol: https://datatracker.ietf.org/doc/html/rfc5424
