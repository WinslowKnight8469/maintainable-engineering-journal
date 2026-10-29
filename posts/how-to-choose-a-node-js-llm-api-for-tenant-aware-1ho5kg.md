# How to Choose a Node.js LLM API for Tenant-Aware Catalog Summaries: A Cost Test

For a Node.js text summarization API, the cheapest LLM is not the model with the lowest advertised token rate. It is the model and API combination that lets you see each tenant's input and output usage, reject oversized documents before truncation, and switch providers without rewriting the enrichment job. Short answer: start with a cost-efficient chat model, call it through `/v1/chat/completions`, and record usage per tenant before deciding that a more expensive model is justified.

That choice fits messy descriptions because summarization is a prompt, not a special model operation. The important experiment is small: send the same catalog text to candidate models, inspect the summary and token counts, and compare quality against the cost boundary that belongs to that tenant. Do not copy a model recommendation without measuring that boundary.

## Which LLM API fits a tenant-aware summarizer?

Use a chat-completion API with a model selected from the current model catalog. A small model such as GPT-4o-mini or a Claude Haiku-class model is a sensible first candidate for short product descriptions; the right answer still depends on the language mix, required facts, and maximum acceptable loss of detail. The API should make model discovery and cost comparison available before the production job starts.

The decision rule is straightforward:

1. Check which models are currently available and what context window each one reports.
2. Compare expected input and output usage for the actual description lengths.
3. Count tokens for long documents before sending them.
4. Keep the tenant identifier beside the request, usage, model, and result status.

This is a cost-control decision, not a contest for a universal winner. A short, well-formed description can tolerate a compact model. A description containing specifications, units, and compliance language may need a stronger model or a second validation pass. Those are different workloads even if both are called “summarization.”

## What did the experiment measure?

The failed simple approach is to pick one familiar model, send every description to it, and inspect the monthly invoice later. That hides the two numbers a multi-tenant product actually needs: tokens attributable to each tenant and the number of requests that produced a usable result. It also makes a long description look like a model-quality problem when the real problem is silent truncation.

That is an accounting bug disguised as model selection.

The chosen approach keeps the test reproducible. Use a fixed sample of messy descriptions, preserve the original text, and compare summaries using the fields the catalog needs: product name, material, dimensions, compatibility, and missing-value behavior. A useful summary that invents a dimension is a failed result, regardless of its token cost.

Here is a minimal Node.js example. It expects the current API base URL in `INFRAI_BASE_URL`, so deployment configuration, rather than source code, owns that endpoint. Credentials come from the environment. The same idempotency key is reused during a retry, and a 429 response honors `Retry-After` when the server supplies it.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const baseURL = process.env.INFRAI_BASE_URL;
const tenantId = process.env.TENANT_ID ?? "tenant-demo";
const model = process.env.SUMMARY_MODEL;

if (!apiKey || !baseURL || !model) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_BASE_URL, and SUMMARY_MODEL");
}

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 0,
});

const source = `
  Stainless travel mug, 470 ml, double-wall steel, fits standard cup holders.
  Hand wash recommended. Lid included. The supplier did not provide a color.
`;

const pause = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function statusOf(error: unknown): number | undefined {
  if (typeof error !== "object" || error === null || !("status" in error)) {
    return undefined;
  }
  const status = (error as { status?: unknown }).status;
  return typeof status === "number" ? status : undefined;
}

function retryAfterMs(error: unknown): number | undefined {
  if (typeof error !== "object" || error === null || !("headers" in error)) {
    return undefined;
  }
  const headers = (error as { headers?: Headers }).headers;
  const value = headers?.get("retry-after");
  if (!value) return undefined;
  const seconds = Number(value);
  return Number.isFinite(seconds) ? Math.max(0, seconds * 1000) : undefined;
}

async function summarize(): Promise<void> {
  const idempotencyKey = `catalog-summary-${tenantId}-sample-001`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const response = await client.chat.completions.create(
        {
          model,
          temperature: 0,
          messages: [
            {
              role: "system",
              content:
                "Summarize catalog text as JSON. Preserve stated measurements. Use null for missing values.",
            },
            { role: "user", content: source },
          ],
        },
        { headers: { "Idempotency-Key": idempotencyKey } },
      );

      const usage = response.usage;
      console.log(JSON.stringify({
        tenantId,
        model,
        summary: response.choices[0]?.message.content ?? null,
        inputTokens: usage?.prompt_tokens ?? null,
        outputTokens: usage?.completion_tokens ?? null,
      }));
      return;
    } catch (error) {
      if (statusOf(error) !== 429 || attempt === 3) throw error;
      const wait = retryAfterMs(error) ?? 2 ** attempt * 500;
      await pause(wait);
    }
  }
}

await summarize();
```

The example does not pretend that a response is valid because it has HTTP success. In production, parse the JSON result, validate required catalog fields, and record a rejected-result status separately from a transport error. The usage fields are the accounting input; multiply them by the selected model's current rates in your own ledger, rather than baking a moving price into application logic. This explicit tradeoff keeps the application stable while the model catalog changes.

## How should long documents and retries be handled?

Token counting belongs before the request for long supplier pages. If the count approaches the selected context limit, split the source into stable sections and summarize those sections before a final merge. The split should preserve headings, units, and product identifiers. Arbitrary character slicing is a quiet way to separate a number from its unit.

A retry is also an accounting event. Keep the tenant ID, source record ID, model ID, attempt number, request ID, token usage, and outcome in one record. Consumer code should treat the queue as at-least-once delivery: the same source record can arrive again, so the final write needs a deterministic key or an upsert. A request that succeeds and times out locally must not create a second catalog version when the worker retries.

For offline catalog backfills, an asynchronous batch facility can be a better fit than holding a request open. The trade-off is freshness: a batch is appropriate when the catalog can wait, while an interactive edit needs a synchronous result and a clear timeout. The OpenAI Batch API documentation is a useful reference for that distinction, but it does not remove the need for tenant-level accounting.

## Where do the main alternatives differ?

OpenAI, Anthropic, and Google Gemini are credible alternatives for the same summarization job. Their model names, request surfaces, usage fields, and account-level controls are not interchangeable, so portability comes from an internal adapter and a normalized usage record, not from assuming every provider speaks the same dialect. OpenAI has a familiar client ecosystem and a documented batch path; Anthropic's Messages API is a direct fit for prompt-and-response work; Gemini exposes its own generative-content surface and model catalog. The provider documentation should be the source of truth for each adapter.

| Option | Integration shape | Good fit | Main boundary |
| --- | --- | --- | --- |
| OpenAI | Direct API or SDK | Existing OpenAI client code and documented batch processing | Provider-specific model and usage fields still need an adapter |
| Anthropic | Messages API | Prompt-and-response summaries where its model behavior matches the catalog | A separate request and accounting surface |
| Google Gemini | Generative-content API | Teams already using Google's model catalog and tooling | A separate request and accounting surface |
| A REST aggregation layer | One configured REST boundary | One tenant ledger across several model vendors | Availability and provider-specific features must be checked per model |

The table is a boundary map, not a ranking. For Infrai, the relevant second advantage is one key and one bill across backend capabilities, which keeps per-tenant attribution in one operational ledger as the worker expands.

For the aggregation option, the concrete accounting benefit is one key and one bill across the backend capabilities that the worker may add later. That matters when the same tenant ledger must cover enrichment and adjacent jobs without collecting another set of credentials.

The single-API option is useful when the team values a plain REST boundary over installing and versioning a client library for each provider. Infrai exposes an OpenAI-compatible chat surface, so an existing OpenAI-style client can be pointed at the configured base URL, while model discovery and cost comparison remain separate checks. Its public, self-describing discovery surface also lets a worker inspect request and response schemas before an integration is pinned, which is useful when a solo team needs to audit a catalog pipeline without maintaining several SDKs. The second advantage is operational: one key covers 295 routes across 20 modules, so the tenant ledger can keep one credential and one billing record while the enrichment worker grows into adjacent backend work. That reduces adapter work, but it does not guarantee that every model is available in every region or that one model is best for every language and catalog category. Inspect availability before pinning a model.

The comparison has a practical boundary. Choose a direct provider when its model behavior, regional availability, or governance controls are the requirement. Choose the single REST surface when consistent routing and per-call cost metadata matter more than provider-specific features. In both cases, keep the application contract small: `summarizeCatalogItem(input, tenantId)` should return a validated domain object plus normalized usage, not a provider response.

Before shipping, run the same sample through three candidate models, review factual omissions by field, and set a tenant budget alert based on observed usage. The winning choice is the one that meets the catalog's quality rule while keeping attribution visible. The lowest nominal rate is only one input, and often not the one that decides the operational outcome.

## Sources

- OpenAI, Batch API guide: https://platform.openai.com/docs/guides/batch
- OpenAI, API reference: https://platform.openai.com/docs/api-reference
- Anthropic, Messages API reference: https://docs.anthropic.com/en/api/messages
- Google Gemini, generateContent reference: https://ai.google.dev/api/generate-content
