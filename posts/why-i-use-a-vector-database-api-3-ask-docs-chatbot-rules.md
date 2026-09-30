# Why I Use a Vector Database API: 3 Ask-Docs Chatbot Rules

**TL;DR:** For a no-infrastructure property-management chatbot, I would start with a hosted vector API behind a tiny adapter, but I would not choose it from a feature matrix. I would choose the least complex service that can filter before retrieval, accept stable document IDs, and return the stored text plus metadata in one query. Then I would split leases, maintenance procedures, and building notices into separate retrieval classes. That boundary is more useful than chasing one supposedly perfect chunk size: it protects answer quality without putting a reranker or a second database in the request path.

This is a retrieval decision, not a database beauty contest. The chatbot has to find the right clause for the right property while keeping an interactive response time. A managed API removes servers from the first release, but the application must still own ingestion, authorization metadata, deletion, evaluation, and the interface to the index. I accept that small amount of code because it limits lock-in and makes latency visible.

## What should an ask-my-docs chatbot require from a vector database API?

A lease clause, an emergency maintenance procedure, and a temporary elevator notice do not fail in the same way. Lease language is dense and locally dependent: a sentence about a fee may only make sense beside its definition. A maintenance procedure often has short ordered steps. A notice may be only a paragraph, yet its building and effective dates decide whether it belongs in an answer.

Putting all three through one chunking rule creates a false simplification. Large chunks preserve context but add irrelevant text to the model input and can dilute the passage that matched. Small chunks are quick to transfer and precise when they work, but can separate a condition from the obligation it modifies. I therefore treat document class as an ingestion input, alongside the property and source revision, rather than as a label added after retrieval.

One rule loses.

My initial defaults would be deliberately boring: approximately 900 characters with 150 characters of overlap for leases, 600 with 80 for procedures, and one whole item for a short notice. Those are starting settings, not measured optima or universal facts. The evaluation set decides whether they survive.

The plain-language flow is short. A worker extracts text, assigns a stable source ID and revision, applies the class-specific chunker, obtains an embedding for each chunk, and upserts the resulting records. At question time, the server embeds the question, applies property and access filters, retrieves a small candidate set, and gives the answer model only those passages with their source labels. Retrieval-augmented generation follows this general separation between generated output and retrieved non-parametric memory [1].

## A narrow TypeScript boundary I can replace

I keep provider-specific SDK objects out of the application. The following example is intentionally an application boundary, not a wrapper around every possible database feature. Its two dependencies can call hosted APIs, a local test double, or a later replacement without changing chunk IDs or metadata.

```ts
type DocumentClass = "lease" | "procedure" | "notice";

type SourceDocument = {
  sourceId: string;
  revision: string;
  propertyId: string;
  accessGroups: string[];
  kind: DocumentClass;
  title: string;
  text: string;
};

type VectorRecord = {
  id: string;
  values: number[];
  text: string;
  metadata: {
    sourceId: string;
    revision: string;
    propertyId: string;
    accessGroups: string[];
    kind: DocumentClass;
    title: string;
    chunk: number;
  };
};

type Match = Pick<VectorRecord, "id" | "text" | "metadata"> & {
  score: number;
};

interface Embeddings {
  embed(texts: string[]): Promise<number[][]>;
}

interface VectorIndex {
  upsert(records: VectorRecord[]): Promise<void>;
  query(input: {
    vector: number[];
    limit: number;
    filter: {
      propertyId: string;
      accessGroup: string;
      kind?: DocumentClass;
    };
  }): Promise<Match[]>;
}

const chunkRules: Record<DocumentClass, { size: number; overlap: number }> = {
  lease: { size: 900, overlap: 150 },
  procedure: { size: 600, overlap: 80 },
  notice: { size: 1_800, overlap: 0 },
};

function chunkText(text: string, size: number, overlap: number): string[] {
  if (size <= 0 || overlap < 0 || overlap >= size) {
    throw new Error("Invalid chunk settings");
  }

  const normalized = text.replace(/\s+/g, " ").trim();
  if (!normalized) return [];

  const chunks: string[] = [];
  const step = size - overlap;
  for (let start = 0; start < normalized.length; start += step) {
    chunks.push(normalized.slice(start, start + size));
    if (start + size >= normalized.length) break;
  }
  return chunks;
}

function recordId(doc: SourceDocument, chunk: number): string {
  return `${doc.sourceId}:${doc.revision}:${chunk}`;
}

async function ingest(
  doc: SourceDocument,
  embeddings: Embeddings,
  index: VectorIndex,
): Promise<void> {
  const rule = chunkRules[doc.kind];
  const chunks = chunkText(doc.text, rule.size, rule.overlap);
  const vectors = await embeddings.embed(chunks);

  if (vectors.length !== chunks.length) {
    throw new Error("Embedding count did not match chunk count");
  }

  await index.upsert(
    chunks.map((text, chunk) => ({
      id: recordId(doc, chunk),
      values: vectors[chunk],
      text,
      metadata: {
        sourceId: doc.sourceId,
        revision: doc.revision,
        propertyId: doc.propertyId,
        accessGroups: doc.accessGroups,
        kind: doc.kind,
        title: doc.title,
        chunk,
      },
    })),
  );
}

async function retrieve(
  question: string,
  propertyId: string,
  accessGroup: string,
  embeddings: Embeddings,
  index: VectorIndex,
  kind?: DocumentClass,
): Promise<Match[]> {
  const [vector] = await embeddings.embed([question]);
  if (!vector) throw new Error("No query embedding returned");

  return index.query({
    vector,
    limit: 6,
    filter: { propertyId, accessGroup, kind },
  });
}
```

The adapter has one non-negotiable semantic requirement: filters must be applied as part of candidate selection, not used to hide unauthorized matches after the query. Post-filtering can return too few useful passages, and retrieving forbidden text before discarding it expands the security boundary for no benefit.

Wrong scope.

Stable IDs matter too. Re-running the same revision should overwrite the same records rather than create duplicates. When a source changes, the revision changes; once the new records are available, the ingestion worker can remove the old revision. The exact transition mechanism belongs in the adapter because different APIs expose different write and deletion primitives.

## How do I trade retrieval quality for latency?

I measure the stages separately. Ingestion timing includes extraction, chunking, embedding, and upsert. Online timing includes query embedding, filtered lookup, optional reranking, and answer generation. One end-to-end number cannot tell me whether a slower answer came from retrieval or from the language model.

For quality, I begin with a small, reviewed question set drawn from the three document classes. Each item needs the property, access group, expected source revision, and the passage that supports the answer. I then record whether the expected passage appears in the first 1, 3, and 6 results. I also keep questions that should return no support, such as asking one building about a notice issued only for another.

No invented benchmark helps here. The useful comparison runs the same documents, embedding output, filters, and query set against every candidate API. It captures lookup latency distributions rather than an average alone, verifies that returned text and metadata require no follow-up read, and repeats after the index has reached its normal size. If a reranker improves difficult lease questions, I add it only after measuring its extra network hop. Ship the simpler path first.

The decision rule is concrete: choose the managed API whose filtered retrieval meets the application's reviewed quality threshold at an acceptable tail latency. Reject one that cannot express the access filter, even if its unfiltered demo is faster. Reject one that makes export or deterministic deletion unverifiable. Price is a constraint after correctness and latency, because a cheap wrong answer about a lease is still wrong.

A hosted vector database API also has a real limitation: it is not suitable when policy requires the index and document payloads to stay entirely inside an environment the service cannot enter. A self-hosted index is the clearer choice in that case, provided the team accepts responsibility for capacity, upgrades, backups, and query availability. The hosted route can also be the wrong trade-off when ingestion volume, filter behavior, or export semantics fall outside a candidate's verified limits. I would resolve those uncertainties with a representative import and deletion test before signing a long commitment, not with a feature-page claim.

There is no universal winner.

## Failure handling is part of the retrieval contract

Ingestion should be retryable by record ID. I would cap batch size in the adapter, apply bounded retries to transient calls, and put a failed source revision into a queue with its error category. A document is searchable only after every expected chunk has been written; tracking the expected count prevents a partially indexed lease from looking complete.

Queries need a different posture. If embedding or retrieval fails, the application should not ask the model to improvise a property answer. It should return a no-source response and log a correlation ID. If retrieval succeeds but no passage clears the evaluated acceptance rule, the result is still “I don't have supporting material,” not a guess.

Stop there.

Observability stays compact: duration and outcome for embedding, lookup, and generation; candidate count; selected source IDs and revisions; document class; and a hash or internal ID for the question rather than raw tenant text. Logs must follow the same access policy as the documents. I also want counters for zero-result queries, incomplete ingestions, stale revisions, and deletions that have not been verified.

Before release, I walk one lease update all the way through the system. The new revision must become searchable, the old one must disappear, a user from the wrong property must get no passage, and every generated claim must retain a source label. Then I run the reviewed question set under representative concurrency and inspect tail latency by stage. This is the operational checklist I trust because it exercises replacement, isolation, grounding, and speed together.

**My choice remains conditional:** use the simplest hosted vector API that passes those checks, keep it behind the small interface, and preserve the three document classes until evaluation shows that combining them is equally accurate and faster. The API is replaceable. The ingestion IDs, access rules, and reviewed retrieval cases are the durable parts.

## Further reading

- [1] Lewis et al., “Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks”: https://arxiv.org/abs/2005.11401
