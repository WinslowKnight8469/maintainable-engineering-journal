# Low-Cost Compatible Image Storage for Startup Avatars, Thumbnails, and Receipts

Short answer: for a startup app, the simplest avatar image storage architecture keeps private objects in object storage, ownership and retention dates in the application database, and separate deletion paths for temporary uploads, reproducible thumbnails, and marketplace receipt originals.

The least complex design has one object namespace and two deletion classes. Avatar thumbnails are replaceable derivatives; receipt originals are audit evidence whose deletion date comes from a business or legal policy. Both can share an adapter, but they must not share a casual `delete everything under this old prefix` rule. Pick a provider only after pricing this exact mix of stored bytes, reads, writes, and outbound delivery. “Cheapest” without that workload is just a label.

## How should a startup app choose image storage for avatar thumbnails?

Start with the data flow, not a provider matrix. The application accepts an upload into a temporary key, validates and processes it, writes immutable destination keys, then commits an application record that names those keys. For an avatar, that record points to the active original and thumbnail revision. For a marketplace receipt, it points to the preserved original, any extracted result, and a `deleteAfter` value derived from the applicable retention policy. A worker may later delete a superseded avatar revision. It may delete a receipt only when the database says the retention window has ended and no hold blocks deletion.

That split is the practical answer to “simple.” S3 compatibility can reduce adapter work, but it doesn't decide ownership, audit retention, or which generated files are safe to recreate. Those are application decisions. Don't encode them only in object names or bucket lifecycle configuration; neither can express every database state that matters to the marketplace.

Policy wins.

## Put the retention decision in code

The small TypeScript example below makes deletion an explicit domain decision before any storage call occurs. Its object store is deliberately generic. A production adapter can implement the same narrow contract for an object service, while tests can use the included memory implementation.

```ts
type ObjectKind = "avatar-source" | "avatar-thumbnail" | "receipt-original";

type StoredObject = {
  key: string;
  kind: ObjectKind;
  ownerId: string;
  deleteAfter: Date | null;
  legalHold: boolean;
};

interface ObjectStore {
  put(key: string, bytes: Uint8Array): Promise<void>;
  delete(key: string): Promise<void>;
}

class MemoryObjectStore implements ObjectStore {
  private readonly objects = new Map<string, Uint8Array>();

  async put(key: string, bytes: Uint8Array): Promise<void> {
    this.objects.set(key, bytes);
  }

  async delete(key: string): Promise<void> {
    this.objects.delete(key);
  }
}

function canDelete(object: StoredObject, now: Date): boolean {
  if (object.legalHold || object.deleteAfter === null) return false;
  return object.deleteAfter.getTime() <= now.getTime();
}

async function deleteEligible(
  store: ObjectStore,
  records: StoredObject[],
  now: Date,
): Promise<string[]> {
  const deleted: string[] = [];

  for (const record of records) {
    if (!canDelete(record, now)) continue;
    await store.delete(record.key);
    deleted.push(record.key);
  }

  return deleted;
}

const store = new MemoryObjectStore();
const receiptBytes = new TextEncoder().encode("sample receipt");
await store.put("receipts/order-42/original", receiptBytes);

const records: StoredObject[] = [
  {
    key: "receipts/order-42/original",
    kind: "receipt-original",
    ownerId: "merchant-7",
    deleteAfter: null,
    legalHold: false,
  },
];

console.log(await deleteEligible(store, records, new Date()));
```

The example returns an empty deletion list because no approved deletion date exists. That is a useful default for receipt originals: absence of policy is not permission to erase evidence. Avatar thumbnails can follow a different rule because the active database revision identifies what must remain, while old derivatives can receive a deletion date after the replacement commits.

There is an awkward failure window worth designing for. Suppose the worker writes a receipt original, processing succeeds, and the database commit does not happen. The object exists but no application row owns it. If lifecycle cleanup targets every unreferenced-looking receipt, an audit original can disappear. If cleanup never runs, abandoned upload cost grows forever. The safer pattern is to upload under a staging prefix, create the authoritative row in the same application workflow, promote or copy to the durable key as appropriate for the adapter, and reconcile staging objects against durable records after a deliberate grace period. That reconciliation should compare exact keys with authoritative records, quarantine ambiguous objects, and emit an outcome that an operator can inspect; inferring ownership from age alone turns a transient database failure into a destructive storage decision. The exact grace period is a product decision; I'm not sure there is a defensible universal value without the upload retry window and audit policy.

## Model cost from object behavior

The cheapest option is the one with the lowest total cost for the measured workload, not necessarily the lowest storage-rate headline. Build a small monthly model with separate rows for avatar originals, avatar thumbnails, receipt originals, and temporary uploads. For each class, record average bytes, new objects, retained objects, reads, writes, deletes, and outbound bytes. Then apply each candidate's current pricing and rounding rules from its own documentation.

This matters because the classes behave differently. Avatars are read often and replaced occasionally. Multiple thumbnail sizes multiply objects and writes but may reduce delivery bytes. Receipt originals may be read rarely yet remain stored much longer. Temporary uploads should be short-lived, though cleanup timing affects their steady-state footprint. A solo founder doesn't need a forecasting system here — a checked-in worksheet with named assumptions is enough — but the assumptions must be visible so a traffic change can overturn the decision without an argument about old marketing pages.

Do a proof-of-fit before committing. Verify private reads, upload limits, listing and deletion behavior, lifecycle granularity, credential scope, and the exact S3 operations the application uses. S3-compatible does not mean every surrounding operational feature is identical. Your mileage may vary once image delivery, regions, and outbound traffic enter the model, so run the same adapter tests against every serious candidate.

The catch is that private object storage alone is not suitable when the product needs an integrated image transformation network, permanent public URLs, or retention controls that satisfy a specific regulatory regime. Use a dedicated image service when transformations and global delivery dominate the work. Use storage with the required retention controls when counsel or an auditor requires them. Stick with a direct cloud service already approved by the team when adding another account, credential, and billing surface would cost more operational time than the storage model saves.

## Separate lifecycle cleanup from authoritative deletion

Lifecycle rules fit expendable data: abandoned multipart work, rejected uploads, processing scratch files, and thumbnails that the application can regenerate. Authoritative deletion belongs in an application job that reads the database policy, checks holds, deletes the named object, and records the outcome. This is intentionally more explicit than applying one age-based bucket rule to everything.

Retries need idempotent keys and idempotent deletion. Use a new revision key for each accepted avatar rather than overwriting a shared `current` key; publish the revision only after its required objects exist. A repeated delete should be treated as the same completed intent, while a failed authorization or network request should remain observable and retryable. Avoid placing sensitive receipt details in object keys because keys commonly travel through logs and metrics.

For deployment, test the adapter contract against a disposable bucket and test policy logic without a network. Exercise simultaneous avatar replacements, an upload with no database row, a receipt with no deletion date, an expired receipt under hold, and a thumbnail whose source is still active. Observe counts and bytes by object class, the age of staging objects, deletion attempts, and policy-blocked deletions. Alert on trends rather than a single noisy retry.

The final operational review should read like a short argument. Confirm that receipt originals get durable keys and explicit retention state; confirm that avatar revisions name every derivative; confirm that staging has bounded cleanup; confirm that deletion checks dates and holds before touching storage; and rerun the cost worksheet with production traffic. Also rehearse credential rotation and restore the application database from backup, because an object bucket full of opaque keys is not a usable audit trail by itself. That's enough ceremony to expose the expensive mistakes without turning a small upload feature into its own platform.

## Further reading

- https://developers.cloudflare.com/r2/
- https://www.backblaze.com/cloud-storage/pricing
