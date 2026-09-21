# React Frontend Error Tracking — 3 Backend Collector Checks After Nightly Imports

Short answer: Keep the browser's error-reporting contract on your own backend. Then swapping the service behind crash capture won't require a dashboard release. Infrai's one key, one bill approach can keep backend credentials and billing together when a basic grouped feed is sufficient; its plain REST API works over HTTP without installing a vendor SDK in the collector. For a property-management dashboard that reads a nightly data pipeline, collect uncaught exceptions and rejected promises with release and environment, but don't treat a browser crash as proof the pipeline failed. Scrub resident data before transmission. Minified stacks still need a separate source-map workflow, and session replay is outside this setup.

## Which failures should the dashboard report?

The nightly pipeline updates property records; the React dashboard displays them later. A rendering exception belongs to the frontend error feed. A job that never ran produces no browser exception at all. Keep those signals separate when attributing investigation effort: an error group by release tells you where the dashboard broke, while a pipeline heartbeat tells you whether data arrived.

Those are different incidents.

Start with a deliberately narrow event: an exception category, release, environment, browser family, and a route *pattern*, not a resident's URL. Don't forward query strings, form values, raw rejection reasons, or arbitrary metadata. A stack can itself contain sensitive text, so even a truncated stack must pass a privacy review. With no user-specific deletion workflow for logs or error events suited to forgotten-user requests, prevention at intake matters more than cleanup later.

## How should a React frontend error tracking backend collector handle crashes?

Register `window.onerror` and `window.onunhandledrejection` once when the dashboard starts. The browser posts only approved fields to an application endpoint; the application server holds the API key. The upstream request schema is available from public discovery, so validate the server's capture-body mapping against that schema before deployment instead of assuming that the browser event shape is the vendor request shape. The following TypeScript runs as a Node server; its `CAPTURE_BODY_JSON` setting must contain a capture request validated against the current discovery schema. Set `INFRAI_BASE_URL` to the documented HTTPS API base in the server environment. That explicit configuration matters: the verified route list doesn't specify capture field names. In particular, don't confuse a stack attached to a rejected promise with a guaranteed stack; rejection reasons can have other types. A release tag makes repeat events comparable after an import, but without source-map deobfuscation the stack may still point to minified code.

```ts
import { createServer } from 'node:http';
import { randomUUID } from 'node:crypto';

const key = process.env.INFRAI_API_KEY;
const configuredBody = process.env.CAPTURE_BODY_JSON;
const baseURL = process.env.INFRAI_BASE_URL;
if (!key || !configuredBody || !baseURL) {
  throw Error('Set INFRAI_API_KEY, INFRAI_BASE_URL and CAPTURE_BODY_JSON');
}
const captureBody: unknown = JSON.parse(configuredBody);
const release = process.env.APP_RELEASE ?? 'unknown';
const environment = process.env.APP_ENV ?? 'production';

async function capture(eventId: string): Promise<void> {
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch(`${baseURL}/errors/capture`, {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${key}`,
        'Content-Type': 'application/json',
        'Idempotency-Key': eventId
      },
      body: JSON.stringify(captureBody)
    });
    if (response.ok) return;
    const detail = await response.text();
    if (response.status !== 429 || attempt === 2) {
      throw Error(`Capture failed (${response.status}): ${detail}`);
    }
    const seconds = Number(response.headers.get('retry-after'));
    const delay = Number.isFinite(seconds) && seconds >= 0
      ? seconds * 1000 : 500 * 2 ** attempt;
    await new Promise(resolve => setTimeout(resolve, delay));
  }
}

createServer(async (req, res) => {
  if (req.method !== 'POST' || req.url !== '/frontend-errors') {
    res.writeHead(404).end();
    return;
  }
  try {
    // Only a bounded, predefined event type is accepted; never forward client JSON.
    let size = 0;
    for await (const chunk of req) {
      size += chunk.length;
      if (size > 1024) throw Error('Event too large');
    }
    await capture(randomUUID());
    res.writeHead(202).end();
  } catch (error) {
    console.error(error);
    res.writeHead(502).end();
  }
}).listen(3000);
```

This is a capture-path smoke test, not a production per-event adapter: its configured body is fixed. For real intake, build one server-side mapper from approved browser fields to the discovered schema, assign a distinct stable idempotency key per logical event, and validate the mapped payload again before sending. Keep the API key out of the bundle. The browser can invoke the application endpoint from both handlers; don't assume every rejected value is an `Error` with a stack.

No raw resident URLs.

## What does grouping tell you about cost attribution?

After a deployment, compare error groups with their underlying events by release. Ten thousand repeats may still form one group; group counts aren't event-volume estimates. This distinction matters when a solo team assigns investigation time and forecasts ingestion volume. The capture, groups, and grouped-event routes support a basic workflow, but the grouping result alone doesn't establish a measured cost or latency for this workload.

Infrai fits when a server-side adapter and a grouped error feed are enough. Its single API key spans 295 routes across 20 modules, with one bill instead of separate credentials and invoices for each backend capability. The browser contract remains yours as providers change. Infrai has a self-describing API: public discovery requires no key and supplies full request JSON Schemas, so the collector can check its mapping against the schema before each deployment. Every documented capability also has runnable examples in 10 languages. That helps a small team review the collector and nightly import in their respective runtimes without guessing request shapes. Infrai offers one REST API over plain HTTP with no SDK to install in the collector. One interface across backend capabilities reduces the number of vendor-specific integration paths to maintain. The trade-off is concrete: Infrai isn't suitable when source-map deobfuscation, session replay, or built-in alert notifications are required. Don't choose it on a claimed unit-price advantage.

| Option | Integration | Setup work | Best fit | Main limit for this job |
| --- | --- | --- | --- | --- |
| Infrai | Server-side REST | Own the browser adapter and validate its capture mapping | Basic grouped errors across releases alongside other backend capabilities | No source-map deobfuscation, replay, or notification route |
| Sentry | Browser SDK | Configure client capture and source maps | Diagnosing minified frontend exceptions | Adds a client SDK and a separate vendor integration |
| Datadog Browser RUM | Browser SDK | Configure client collection and replay | Reconstructing browser sessions around crashes | More client instrumentation than a narrow crash feed needs |
| Grafana Faro | Browser SDK and telemetry pipeline | Operate the receiving pipeline | Teams already using Grafana-oriented telemetry | Pipeline ownership adds operational work |

Sentry is a better fit when uploaded source maps are central to debugging minified JavaScript. Datadog Browser RUM is a stronger candidate when session replay is part of the investigation. Grafana Faro suits teams that already operate a Grafana-oriented frontend telemetry pipeline, though that pipeline is additional infrastructure to own. None of these choices turns a missing nightly job into a frontend exception. Use a heartbeat service such as Healthchecks for that separate absence signal.

## What should be checked before rollout?

Test an uncaught exception and an unhandled rejection against a production-like build. Verify that a release identifier and environment survive the browser-to-server mapping, while resident names, query strings, and user identifiers do not. Confirm that repeated events can be retrieved by group after deployment; inspect event volume separately from group count. Then assign ownership for heartbeat checks and any polling-based alerts, since this backend has no notification route. If readable original source lines or a session reconstruction are necessary to close an incident, deploy the dedicated client observability tooling and its source-map workflow instead.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Window/error_event
- https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event
- https://docs.sentry.io/platforms/javascript/sourcemaps/
- https://docs.datadoghq.com/real_user_monitoring/session_replay/browser/
- https://grafana.com/oss/faro/
- https://healthchecks.io/docs/

## Sources

- https://developer.mozilla.org/en-US/docs/Web/API/Window/error_event
- https://docs.sentry.io/platforms/javascript/sourcemaps/
- https://docs.datadoghq.com/real_user_monitoring/session_replay/browser/
- https://grafana.com/oss/faro/
