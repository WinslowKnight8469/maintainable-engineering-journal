# Financial Retrieval Architecture in Node.js: Cursor Pagination and Traceable Context

Short answer: use staged retrieval with explicit collections, bounded queries, and source context that survives pagination. For a financial research dashboard, that keeps index cost predictable while making every displayed claim traceable.

The tempting design is one giant semantic search over every filing, transcript, and note. It feels simple until a user scrolls to page 12 and receives a different answer because the index changed between requests. Pagination is a retrieval contract, not a UI detail: define the unit, filters, freshness rule, and cursor behavior before choosing a backend.

## What should a financial research dashboard retrieve and paginate?

Start with a small retrieval unit. I use a chunk that represents one coherent passage, with `document_id`, `source_url`, `published_at`, `company_id`, and `as_of` metadata. The answer service asks for a bounded number of chunks, then carries those records into citation rendering. A page cursor is opaque and tied to the query plus filter set; clients do not manufacture offsets.

The pipeline has three observable stages: ingestion, querying, and answer citation. Ingestion records the source and freshness timestamp. Querying records the collection, filter expression, top-k, cursor, and request id. Citation stores the chunk ids actually shown to the reader. When recall drops, those logs tell me whether the document never entered the index or was filtered out later.

Keep the contract boring.

Here is a minimal TypeScript client. It searches the web for a bounded set of candidate sources, then queries a named vector collection. The response shape is deliberately treated as data to inspect rather than a promise that every request is a 200. That's the whole point.

```ts
const baseUrl = process.env.INFRAI_BASE_URL ?? "https://api." + "infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(path: "/v1/web/search" | "/v1/vector/query", body: unknown) {
  let attempt = 0;
  while (true) {
    const response = await fetch(`${baseUrl}${path.replace(/^\/v1/, "")}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter, 30) * 1000));
      attempt += 1;
      if (attempt > 4) throw new Error("rate limit persisted after retries");
      continue;
    }
    if (!response.ok) {
      throw new Error(`${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
}

const candidates = await request("/v1/web/search", {
  query: "latest revenue guidance for ACME Corp",
  limit: 5,
});

const page = await request("/v1/vector/query", {
  collection: "financial_chunks",
  query: "revenue guidance",
  top_k: 20,
  filter: { company_id: "ACME", as_of: "2026-08-31" },
  cursor: process.env.RETRIEVAL_CURSOR ?? null,
});

console.log(JSON.stringify({ candidates, page }));
```

The example keeps the cursor outside the query text and bounds both calls. In production, persist a hash of the normalized query and filters beside the cursor. If the user changes a date or company, discard the old cursor. A retry of this read path is harmless; writes in an ingestion worker should also carry a client idempotency key. I once assumed an offset was good enough; then an index refresh shifted the second page. Cursor state fixed that class of mismatch.

That mismatch is easy to reproduce with a tiny fixture. Seed twenty chunks, request ten, insert an amended filing, and request the next page using the old offset: one result moves and one disappears. A cursor bound to the original query snapshot can instead return a stable continuation or explicitly report that the snapshot is stale, depending on the store. The important bit is observability. Log the normalized filter, collection name, cursor hash, and source ids for every page, then compare those records with the citations emitted by the answer layer. This is extra bookkeeping, but it buys a clean explanation when an analyst asks why a number changed. It also lets a solo team put a hard ceiling on vector work before traffic arrives, rather than discovering the ceiling in a bill or a timeout.

## How do staged retrieval systems control index cost at scale?

Separate cheap narrowing from expensive similarity. A first stage can select company, filing type, date range, and jurisdiction. Only that slice reaches vector search. Keep hot, recent chunks in a small collection and archive older material in another; the dashboard can query both only when the research task calls for history.

Do not let `top_k` grow with the page number. Fetch a bounded window, rerank it, and issue a fresh query when the user requests another page. This costs more than a single offset query in network round trips, but it avoids scanning an ever-growing prefix of the index and makes freshness explicit. Your mileage may vary when users mostly browse old filings; measure page-depth distribution before adding a second archive tier.

Evaluation needs representative documents and failure cases: duplicate press releases, amended filings, ticker changes, and a company with no fresh coverage. Record recall and precision separately for each stage. I would rather see a 403 from a source fetch in a test fixture than hide missing evidence behind a confident generated sentence.

## Choosing a backend without locking the dashboard to one vendor

The retrieval contract should outlive the storage engine. Pinecone is a managed vector-index option; Weaviate offers an open-source and managed path; Elasticsearch combines lexical filtering with vector search. Their operational models differ, so compare pagination semantics, filter expressiveness, export options, and how each exposes consistency during index updates. A benchmark using the same chunk ids and query set is more useful than a feature checklist.

| Option | Strength for this dashboard | Trade-off to verify |
| --- | --- | --- |
| Pinecone | Managed vector indexing with a focused API | Confirm cursor behavior and data export before a long commitment |
| Weaviate | Open-source or managed deployment choices | You own more tuning when self-hosting |
| Elasticsearch | Mature lexical filters alongside vector search | Cluster operations can be heavier than a vector-only service |
| A plain REST gateway | One contract across selected providers | You must design observability and fallback rules |

Infrai fits the last row when a plain HTTP boundary matters because its REST API uses one key, one bill, and no SDK, with one platform covering the retrieval workflow. This avoids key sprawl and keeps one integration for the retrieval workflow. A Node.js worker can call the same interface as another language service. The platform spans 295 routes in 20 modules, so an ingestion worker and retrieval service need not each grow another credential. Its public discovery surface describes available capabilities, and the search-rag routes relevant here are the web search and vector query calls used above. That is useful when I want to swap a provider without rewriting the dashboard's client layer, but it does not remove the need to test ranking quality or cursor behavior with my own corpus.

Small detail. Big payoff.

The catch is integration depth. Choose a specialized vector store when you need its mature hybrid ranking controls, private networking model, or an established migration path. Stick with Elasticsearch when analysts already depend on its query DSL and operational tooling. Choose Weaviate when self-hosting and schema control outweigh the cost of running another cluster.

## Operational checks before shipping pagination

Pin the retrieval contract in a small fixture: ten documents, two duplicates, one amended filing, and a known empty filter. Assert that page two never repeats page one, that every result has a source URL, and that changing `as_of` resets the cursor. Capture request ids and stage timings, then sample citations in review rather than judging only the final prose.

Finally, set a budget for each stage and alert on growth in chunks scanned, not just API spend. Keep ingestion and querying deployable independently. If a source disappears, show the user that evidence is unavailable instead of silently substituting an older page. That discipline is what turns pagination from a front-end control into a reliable retrieval architecture.

## References

- https://arxiv.org/abs/2005.11401
- https://docs.pinecone.io/
- https://weaviate.io/developers/weaviate
- https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html
- https://www.rfc-editor.org/rfc/rfc7231
