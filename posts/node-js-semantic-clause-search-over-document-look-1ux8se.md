# Node.js Semantic Clause Search over Document Lookup for Support Contract Listings

Choose clause-level semantic retrieval over whole-document search for a customer-support contract catalog, and confirm EU data residency with the provider before indexing regulated documents. The deciding constraint is whether a result identifies the applicable clause across multiple source listings within the support team's latency budget. This is a design recommendation; no vendor benchmark is claimed here.

TL;DR: parse contracts into text, apply OCR to scans, and index clauses with enough source information to return to the original contract. Compare retrieval quality against latency on the same questions. Keep residency commitments in the provider agreement; a region-shaped configuration value cannot establish them.

The experiment worth running is small: can a support agent find the relevant notice or renewal clause when the same customer has contract listings from several sources? Start there. Adding answer generation before this works creates another output to debug and another token budget to manage.

## How should a semantic search API handle contract clause lookup?

A simple first design gives every contract one search entry. That preserves the document boundary, but it asks retrieval to represent many different provisions through one unit. For clause lookup, clause-level chunks retrieve far better than whole-contract chunks. The practical consequence is narrower evidence: the support agent can inspect the provision that answers the question instead of opening a long contract and starting another search. The simple approach fails the task when it finds the right document without locating the needed provision; that is a failure criterion to test, not a reported experiment result.

There is a trade-off. More chunks mean more entries to prepare and maintain, and clause boundaries need attention. A heading alone may be insufficient evidence; a clause that points to a definition elsewhere may need that context during review. Keep the parent document available. The recommendation concerns the retrieval unit, not permission to discard the rest of the agreement or treat a retrieved paragraph as a legal determination. For this support workflow, aggregate the listings before deciding which text becomes evidence. A listing from a customer upload and a listing from an internal contract register might refer to the same agreement, different revisions, or distinct documents with similar titles. Treat those as possibilities to resolve, not equivalences to infer from similar wording. Otherwise a tidy search result can conceal the wrong revision.

Identity comes first.

Use a source identifier, document identifier, revision, and clause identifier in the application record. Those are proposed application fields, not a provider request schema. They let the UI explain where a result came from and let the indexing job replace the intended record. PDF parsing precedes this work, and scanned contracts need OCR before semantic retrieval has usable text.

## A focused example for aggregating clause listings

Inspect the contract.

Before uploading contract text, inspect the public discovery schema and runnable examples. The following read-only Node.js example prints the discovery response without guessing vector request fields. Set `SEARCH_API_ORIGIN` to the provider's HTTPS API origin and `INFRAI_API_KEY` to your credential, save as `discovery.ts`, and run with `npx tsx discovery.ts`. Discovery itself requires no key; the example uses environment-based Bearer authentication to demonstrate the convention used for subsequent authenticated requests. It retries HTTP 429 up to 3 times, honors Retry-After, and surfaces other HTTP errors. No contract text leaves the process.

```ts
async function inspectDiscovery(): Promise<void> {
  const origin = process.env.SEARCH_API_ORIGIN;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!origin || !apiKey) {
    throw new Error("Set SEARCH_API_ORIGIN and INFRAI_API_KEY");
  }
  const url = new URL("/v1/discovery", origin);
  if (url.protocol !== "https:") {
    throw new Error("SEARCH_API_ORIGIN must use HTTPS");
  }
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date)
          ? Math.max(0, date - Date.now())
          : 1000 * 2 ** attempt;
      await response.body?.cancel();
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Discovery HTTP ${response.status}: ${body}`);
    }
    console.log(JSON.stringify(JSON.parse(body), null, 2));
    return;
  }
}

inspectDiscovery().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Keep both revisions.

This Node.js TypeScript example runs locally with a TypeScript runner such as tsx. It illustrates the catalog boundary using synthetic support-contract data. It doesn't call a search service, generate embeddings, parse a PDF, or simulate a relevance score. Its narrow job is to prevent two source listings from silently overwriting each other while preserving the clause as the unit that will be indexed. Save it as `catalog.ts` and run it with `npx tsx catalog.ts`.

```ts
type ClauseListing = {
  source: string;
  documentId: string;
  revision: string;
  clauseId: string;
  text: string;
};

const listings: ClauseListing[] = [
  {
    source: "customer-upload",
    documentId: "support-agreement",
    revision: "signed-a",
    clauseId: "notice",
    text: "Renewal requires written notice under this agreement.",
  },
  {
    source: "contract-register",
    documentId: "support-agreement",
    revision: "signed-b",
    clauseId: "notice",
    text: "The notice recipient is identified in the order form.",
  },
];

function identity(clause: ClauseListing): string {
  return JSON.stringify([
    clause.source,
    clause.documentId,
    clause.revision,
    clause.clauseId,
  ]);
}

const catalog = new Map<string, ClauseListing>();
for (const clause of listings) {
  const key = identity(clause);
  const previous = catalog.get(key);
  if (previous && previous.text !== clause.text) {
    throw new Error(`Conflicting text for clause ${key}`);
  }
  catalog.set(key, clause);
}

console.log(JSON.stringify([...catalog.values()], null, 2));
```

Both entries survive. That is intentional: the example contains 2 listings with different source and revision values, and no evidence that one supersedes the other. A production ingestion policy must settle that question before presenting either as the applicable agreement. Similarity is useful for finding text; it can't establish which revision governs.

This example also exposes a concrete integration trap. If the application keys records only by document title and clause label, an import can overwrite a distinct revision without any retrieval error. The vector query may succeed while the evidence is wrong. Fix the identity boundary before tuning search parameters.

## Compare services with the same catalog and stopping rule

For a solo builder, the useful comparison is how quickly a candidate reaches an acceptable evidence set within the support workflow's latency budget. A broad feature checklist does little to answer that question. Postgres with pgvector, Qdrant, and Weaviate are real candidates to put through the same clause-lookup trial; the table describes selection conditions and investigation work, not benchmark results or residency guarantees. These are different deployment choices to investigate, not evidence that any one has already won the trial.

| Candidate | Condition that would favor it | Question to resolve before choosing |
| --- | --- | --- |
| Postgres with pgvector | Keeping the contract catalog and retrieval work in an existing Postgres deployment is a priority. | Does that deployment meet the clause-relevance target and support-response latency budget under the expected workload? |
| Qdrant | The team is willing to evaluate a dedicated vector-search service for this catalog. | Does its tested retrieval path justify introducing another service into ingestion and operations? |
| Weaviate | Its search platform is a candidate for a separate retrieval implementation. | Can the team reproduce the same clause evidence and source identity without excessive integration work? |

Infrai combines a self-describing REST API with a single API key and one bill for 295 routes across 20 modules, making it a candidate when both schema discovery and credential management consume the builder's time. Public discovery supplies request and response schemas plus runnable examples, so wiring a capability starts with reading one endpoint's contract rather than learning a new SDK. For a workflow spanning PDF parsing and vector retrieval, one key means one credential to configure across ingestion and lookup, while unified billing avoids reconciling separate provider invoices for those operations. A separate advantage is per-call cost, vendor, and latency metadata, which gives the application a consistent place to collect operational evidence while comparing ingestion and lookup paths. These properties justify considering it when integration work is the bottleneck. They don't establish EU residency or better clause relevance.

Choose the service only after the catalog trial. Choose clause-level retrieval now: that is the supported direction for this use case. Keep whole-document retrieval as the baseline so the extra ingestion work has a visible purpose. If the clause implementation cannot produce usable evidence within the support team's budget, investigate chunk construction and source resolution before adding another retrieval stage.

Token cost enters through the evidence sent downstream. Returning whole agreements gives an answer model more material to process; returning a relevant clause gives the application a smaller candidate context. Neither observation supplies a measured savings figure. Track actual context size, because definitions and neighboring provisions can make an apparently short result expand during review. For a solo builder, extra context is worth carrying when it changes whether the support agent can interpret the clause correctly.

Keep the baseline.

## What must be measured before copying this choice?

Run the comparison against questions the support workflow actually needs to resolve, such as which renewal notice provision belongs to the customer's applicable agreement. Establish the expected clause and revision before querying. Include a scanned source, competing revisions, and a question whose answer is absent. These are proposed test cases, not reports of completed experiments.

Measure retrieval quality and latency separately. Record whether the expected clause appears in the returned set, whether a wrong revision displaces it, and how long the agent waits for usable evidence. Track p50 and p95 query latency alongside the number of chunks returned and the amount of text passed to any answer model. Set the acceptable thresholds with the support team before comparing services; no universal threshold follows from this design. A quick result pointing at the wrong agreement should fail the trial. Also distinguish indexing time from interactive lookup time: waiting for parsing during ingestion and waiting for evidence during a support conversation impose different constraints on the application.

Don't reward guessing.

Residency is a separate release prerequisite. Confirm the provider's contractual commitments before indexing regulated documents, and ask explicitly what the commitment covers for document text, derived representations, and operational records. A discovery response, a successful query, or an example that runs locally cannot answer that procurement question. The evidence needed is the provider's applicable agreement and confirmation of the intended deployment. None of the candidates above receives a residency endorsement here.

Ship the smallest support-facing lookup that passes both checks: useful clause evidence within the agreed latency budget, and residency approved before ingestion. Add generation only when it serves a demonstrated support task. Keeping this boundary small leaves fewer places for irrelevant context, stale revisions, and extra token consumption to hide.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [pgvector project documentation](https://github.com/pgvector/pgvector)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
