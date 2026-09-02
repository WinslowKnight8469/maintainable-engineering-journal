# Cheapest Node.js Plan for Property Documents: AI Thumbnails, Object Storage, Signed URLs

Short answer: for a property-management app, put each signed document and its AI-generated thumbnail in private object storage, keep authorization and deletion deadlines in the application database, issue short-lived signed URLs only after an access check, and run an application cleanup worker even if the storage layer also has lifecycle rules. The cheapest design is the one that retains the fewest unnecessary bytes without weakening access control.

That decision separates two concerns that are easy to blur. Object storage holds immutable bytes. The database decides who may see them, which lease or property they belong to, and when they must disappear. A signing flow writes the final PDF first, creates a preview under a different key, then commits both keys and one explicit deletion deadline in a transaction. A read request checks the current user's relationship to the property before returning a signed URL. A deletion worker later removes both objects and records completion.

Keep the deadline explicit. “Temporary” isn't a retention policy.

## Implementation: model each property document as a state machine

Start with private objects and opaque, immutable keys. A practical record might contain `documentKey`, `thumbnailKey`, `contentType`, `sha256`, `deleteAfter`, and `deletionState`. Authorization belongs in ordinary application data: tenant, property, lease, role, and any legal hold. Do not encode those mutable facts into the object key and expect a prefix to become an access-control system.

The write sequence matters. Upload the signed PDF to a new key, generate and upload the thumbnail to another new key, and only then publish the database row that points to both. If thumbnail generation fails, the document can remain in a non-visible staging state until the job is retried or cleaned up. Never overwrite the bytes behind an already published key; a new signature event should produce a new object key and a new database version. This keeps caches and audit records from silently referring to changed content.

For a concrete example, imagine a lease packet signed at `2026-08-11T09:00:00Z`. The business rule in this example assigns `deleteAfter` as `2027-08-11T09:00:00Z`; that date is application data, not a claim about a universal legal retention period. The original might use `documents/01J.../signed.pdf`, while its preview uses `previews/01J.../page-1.webp`. Both keys share a database owner and deadline, but they sit under different prefixes so operations can measure derivative growth separately.

The thumbnail is a convenience copy. It should never outlive the signed source.

## Rollout: run the storage orchestration before debating trade-offs

The useful abstraction here is small: write bytes, create a signed read URL, and delete a key. The provider adapter implements those operations; the application owns authorization and retention. This example is intentionally strict about ordering and makes no assumptions about a commercial API route.

```ts
import { createHash, randomUUID } from "node:crypto";

type StoredDocument = {
  id: string;
  propertyId: string;
  documentKey: string;
  thumbnailKey: string;
  sha256: string;
  deleteAfter: Date;
  deletionState: "active" | "pending" | "deleted";
};

interface ObjectStore {
  put(key: string, body: Uint8Array, contentType: string): Promise<void>;
  signRead(key: string, expiresInSeconds: number): Promise<URL>;
  delete(key: string): Promise<void>;
}

interface DocumentRepository {
  insert(document: StoredDocument): Promise<void>;
  findAuthorized(documentId: string, userId: string): Promise<StoredDocument | null>;
  claimExpired(now: Date, limit: number): Promise<StoredDocument[]>;
  markDeleted(documentId: string): Promise<void>;
  releaseDeletion(documentId: string): Promise<void>;
}

export async function retainSignedDocument(
  store: ObjectStore,
  repository: DocumentRepository,
  input: {
    propertyId: string;
    pdf: Uint8Array;
    thumbnail: Uint8Array;
    deleteAfter: Date;
  },
): Promise<StoredDocument> {
  if (input.deleteAfter.getTime() <= Date.now()) {
    throw new Error("deleteAfter must be in the future");
  }

  const id = randomUUID();
  const documentKey = `documents/${id}/signed.pdf`;
  const thumbnailKey = `previews/${id}/page-1.webp`;

  await store.put(documentKey, input.pdf, "application/pdf");
  try {
    await store.put(thumbnailKey, input.thumbnail, "image/webp");
    const document: StoredDocument = {
      id,
      propertyId: input.propertyId,
      documentKey,
      thumbnailKey,
      sha256: createHash("sha256").update(input.pdf).digest("hex"),
      deleteAfter: input.deleteAfter,
      deletionState: "active",
    };
    await repository.insert(document);
    return document;
  } catch (error) {
    await Promise.allSettled([
      store.delete(documentKey),
      store.delete(thumbnailKey),
    ]);
    throw error;
  }
}

export async function getDocumentDownload(
  store: ObjectStore,
  repository: DocumentRepository,
  documentId: string,
  userId: string,
): Promise<URL> {
  const document = await repository.findAuthorized(documentId, userId);
  if (!document || document.deletionState !== "active") {
    throw new Error("Document not available");
  }
  return store.signRead(document.documentKey, 60);
}

export async function deleteExpiredBatch(
  store: ObjectStore,
  repository: DocumentRepository,
  now: Date,
): Promise<void> {
  const expired = await repository.claimExpired(now, 100);
  for (const document of expired) {
    try {
      await Promise.all([
        store.delete(document.documentKey),
        store.delete(document.thumbnailKey),
      ]);
      await repository.markDeleted(document.id);
    } catch (error) {
      await repository.releaseDeletion(document.id);
      throw error;
    }
  }
}
```

The `ObjectStore` adapter is where credentials, retry policy, and provider-specific request shapes belong. Keep it server-side. The browser receives the resulting signed URL, never the storage credential. The 60-second expiry above is an example chosen for a single download handoff, not a magic standard; slow clients, large packets, or multi-page viewers may need a different value.

The cleanup claim needs database-level exclusion so two workers don't process the same row. Deletion should also be idempotent at the adapter boundary: a missing object counts as the desired final state. If the second object delete fails, the row returns to a retryable state, and the next pass attempts both keys again. That is why the database record remains until object deletion finishes.

Bytes linger.

## Can migration preserve Node.js object storage policy for AI thumbnails and signed URLs?

Yes, if the port exposes byte operations rather than a vendor's policy vocabulary. `put`, `signRead`, and `delete` are enough for this workflow; object keys, authorization rules, deadlines, and deletion states remain in application-owned data. Keep provider response types out of `StoredDocument`, translate adapter errors into a small internal set, and contract-test a replacement adapter against the same cases. The application then changes storage credentials and adapter construction without changing who can open a lease or when its preview expires.

Portability still has limits. Signed-URL semantics, conditional operations, lifecycle configuration, response-header overrides, and consistency behavior can differ across storage systems, so the interface must describe only the guarantees the application actually tests. A migration also has a data plane: copy bytes, verify hashes, switch database pointers in bounded batches, and retain a rollback window that does not violate deletion deadlines. Do not promise a provider swap as a configuration-only event. The narrow port keeps policy code stable; it doesn't make terabytes move for free.

## Cost model: count retained byte-days before comparing storage prices

The first cost question isn't which bucket has the smallest advertised number. It is how many originals and derivatives remain at the end of each day. A property packet can create a PDF, a full-page render, a list thumbnail, and a temporary signing preview; if every revision leaves all four behind, the retention model dominates the unit-price comparison. Build a small estimate from monthly packets, average bytes per object kind, revision count, and retained days. Keep request and delivery estimates on separate lines so a busy viewing workflow doesn't masquerade as storage growth.

Measure bytes by retention class and object kind rather than chasing a headline price. Storage cost, request cost, delivery cost, thumbnail compute, and operational labor can move independently. A smaller thumbnail saves storage and delivery bytes but may create visible artifacts; a longer signed-URL expiry reduces refresh calls but expands the exposure window. Your mileage may vary because document size, preview traffic, and regional delivery patterns decide which term dominates. Run the estimate with your own monthly counts and sizes, then add a budget alert on total bytes and their growth rate.

A lifecycle rule can be useful for a prefix that is unambiguously disposable, such as failed preview jobs. It is a poor cost-control plan for signed originals because age alone doesn't express renewal, erasure, or legal hold. The application deadline remains the source of the deletion job; provider lifecycle configuration is a delayed safety net for orphaned temporary bytes.

Cheap storage cannot repair an ambiguous deadline.

## Decision: choose a delivery path by its revocation window

Private storage plus signed URLs adds an authorization hop, but it keeps the decision current. A public URL is simpler to render and cache, yet anyone who obtains it may continue using it according to the delivery layer's behavior. In a property workflow, a tenant moving out, a manager changing roles, or a deletion request can all change access after the object was created. The app must evaluate those facts before minting a new capability URL.

Treat a signed URL as a bearer secret. Don't put it in analytics events, support screenshots, persistent database fields, or routine request logs. A short expiry limits the window; it does not revoke a URL already issued, and it does not replace the authorization check that preceded issuance. Return a generic `404` for both “missing” and “not authorized” if revealing document existence would leak tenancy information. Test `403` behavior at the application boundary if that is the contract you choose, but keep the choice consistent.

`Content-Disposition` controls presentation, not permission. The response header can suggest `inline` display or `attachment` download and can carry a filename. For a signed lease packet, `attachment` is usually easier to reason about; for a thumbnail, `inline` fits the preview. Browser behavior and same-origin rules have details, so verify the actual response in every delivery path rather than assuming a query parameter changed the header. The header itself does not make a public object private.

There is a real catch. Signed URLs are not suitable when the client needs immediate revocation after issuance, extremely long offline access, or fine-grained per-page policy. Proxying downloads through the Node.js service can enforce a check for every request and shape response headers centrally, but then the service carries bandwidth, connection duration, and back-pressure. Stick with the proxy path when immediate policy enforcement matters more than delivery simplicity. Use short-lived direct delivery when authorization at issuance is sufficient and the application should stay out of the byte stream.

I would make that decision per document class, not once for the whole bucket. I'm not sure a future legal-hold workflow will tolerate the same delivery rules as ordinary tenant previews; counsel and a threat model should resolve that uncertainty before launch.

## Reliability: rehearse deadline failures before launch

A storage lifecycle rule and an application deletion worker solve different problems. A lifecycle rule can remove objects by age or prefix according to the chosen storage system's configuration. It cannot infer that a lease was renewed, a resident requested erasure, a legal exception applies, or a database pointer moved. The application worker understands those events because it reads business state. Use lifecycle cleanup as a backstop for clearly disposable prefixes, not as the sole implementation of a user-specific deadline.

GDPR Article 17 describes a right to erasure in specified circumstances and also lists exceptions. It is not a universal instruction to delete every document after the same interval. Record the policy source that produced each deadline, keep legal-hold state separate from ordinary retention, and have qualified counsel define the applicable rule. The system's job is less glamorous: represent the decision faithfully, execute it, and produce evidence without retaining the deleted document itself.

Run one detailed clock-boundary exercise before deployment. Create a packet with an original key and a preview key, set `deleteAfter` to a controlled instant, and pause one worker immediately after it claims the row. Start a second worker; it must find no claimable copy. A read one millisecond before the deadline may mint a short-lived URL under the chosen policy, while a read one millisecond after it must not. Resume the first worker, remove both objects, and interrupt it before `markDeleted`; after restart, it should repeat idempotent deletes and finish the row rather than release access. Then try a partial upload, retry the same generation job, and verify that unpublished keys are reclaimed. Confirm that logs contain object IDs rather than signed query strings, and that backup retention does not quietly contradict the product deadline. Finally, inspect the delivered response for `Content-Disposition`, including filenames with spaces and non-ASCII characters. This single exercise crosses authorization, time, concurrency, retries, header behavior, and evidence, which is exactly where a superficially cheap design tends to become expensive.

Test the clocks.

Keep the operational checklist in prose because order matters. Alert on cleanup backlog age, deletion retry count, orphan scans, bytes by prefix, and signed-URL issuance volume. Review a sample of deletion evidence. Reconcile database keys against object listings on a schedule, but don't let the reconciliation job invent business truth from filenames. Finally, rehearse credential rotation and adapter replacement; a narrow storage port makes that change possible without rewriting authorization or retention policy.

This design is deliberately boring — and bounded. It is not suitable for records that require immutable legal preservation, independently verified destruction, cross-region residency controls, or immediate URL revocation. Those requirements call for a storage and governance review before implementation, not another flag in the thumbnail worker.

## References

- MDN, `Content-Disposition`: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- GDPR Article 17, right to erasure: https://gdpr-info.eu/art-17-gdpr/

## Further reading

- Browser handling of `Content-Disposition`: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- The text and exceptions of GDPR Article 17: https://gdpr-info.eu/art-17-gdpr/
