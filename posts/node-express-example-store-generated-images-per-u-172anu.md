# Node Express Example: Store Generated Images per User Folder with Object-Storage Expiry

Short answer: store each generated image as a private immutable object, retain its owner and object key in application metadata, delete by that recorded key when product state changes, and use an age-based lifecycle policy only as a backstop.

For an AI feature, the difficult part is rarely writing bytes. It is deciding which system can answer ownership, when a removal is actually complete, and what happens when the database and object store observe the same request at different times. A per-user "folder" helps operators navigate a bucket, but it does not authorize a read. The application must do that.

The data flow is short: an authenticated Express request or generation worker creates an image ID, the server constructs a key, the storage adapter receives the bytes, and a metadata row records the owner, key, media type, and state. On removal, look up the row by both image ID and authenticated user, then delete the recorded key and settle the metadata through a retryable workflow. Keep the bytes private by default.

## What should a Node Express AI image storage and lifecycle design protect?

Treat object keys as addresses, not permissions. A key such as `tenants/{tenantId}/renders/{imageId}.webp` makes bulk review and prefix-scoped retention understandable, but a client must never be allowed to choose the whole key. After authentication, the service generates the image ID and derives the key from the server-side identity. That stops a guessed key from becoming a cross-tenant write target.

The database, or another indexed metadata service, is the authority for ownership and product state. It should retain the object key rather than recreate it during cleanup. Naming conventions change; stored metadata is the historical record. A row can also hold the creation time, content type, byte size, generator request ID, and a state such as `active` or `pending_delete`. Those fields make account erasure, support investigation, and retention queries concrete without scanning an entire bucket.

Do not use public object URLs as the default delivery mechanism for user output. Authorization needs to happen before the application streams an object or creates a time-limited access mechanism. Credentials belong in a secret-management system or protected runtime configuration, with rotation and least-privilege access rules. OWASP's guidance is useful here because storage credentials can otherwise end up in source control, logs, or a browser bundle.

There is one further boundary: image validation. Generated output still needs an allowlisted media type and a size limit before it reaches storage. The exact limit depends on the model output and delivery path, so it should be a product setting with metrics rather than a magic constant copied between services. Small detail, large consequence.

Ship it in small slices.

## A focused TypeScript implementation

The adapter below is deliberately narrow. It can be backed by an object-storage protocol, a self-hosted implementation, or a test double. Express owns authentication, key construction, and the ordering of metadata changes; the adapter only stores and removes bytes.

```ts
import express, { Request, Response } from "express";
import { randomUUID } from "node:crypto";

type ImageRecord = {
  id: string;
  tenantId: string;
  objectKey: string;
  contentType: "image/png" | "image/jpeg" | "image/webp";
  state: "active" | "pending_delete";
  createdAt: Date;
};

interface ObjectStore {
  put(input: { key: string; body: Buffer; contentType: string }): Promise<void>;
  delete(key: string): Promise<void>;
}

interface ImageRepository {
  insert(record: ImageRecord): Promise<void>;
  findOwned(imageId: string, tenantId: string): Promise<ImageRecord | null>;
  markPendingDelete(imageId: string, tenantId: string): Promise<void>;
  remove(imageId: string, tenantId: string): Promise<void>;
}

declare const store: ObjectStore;
declare const images: ImageRepository;

type AuthedRequest = Request & { user: { tenantId: string } };
const app = express();
const allowedTypes = ["image/png", "image/jpeg", "image/webp"] as const;

app.use(express.raw({ type: allowedTypes, limit: "12mb" }));

app.post("/images", async (request: Request, response: Response) => {
  const req = request as AuthedRequest;
  const contentType = req.headers["content-type"];
  if (!allowedTypes.includes(contentType as (typeof allowedTypes)[number])) {
    response.status(415).json({ error: "unsupported_image_type" });
    return;
  }

  const id = randomUUID();
  const extension = contentType === "image/png" ? "png" : contentType === "image/jpeg" ? "jpg" : "webp";
  const objectKey = `tenants/${req.user.tenantId}/renders/${id}.${extension}`;
  const record: ImageRecord = {
    id,
    tenantId: req.user.tenantId,
    objectKey,
    contentType: contentType as ImageRecord["contentType"],
    state: "active",
    createdAt: new Date(),
  };

  await store.put({ key: objectKey, body: req.body as Buffer, contentType });
  await images.insert(record);
  response.status(201).json({ id });
});

app.delete("/images/:imageId", async (request: Request, response: Response) => {
  const req = request as AuthedRequest;
  const image = await images.findOwned(req.params.imageId, req.user.tenantId);
  if (!image) {
    response.sendStatus(404);
    return;
  }

  await images.markPendingDelete(image.id, image.tenantId);
  await store.delete(image.objectKey);
  await images.remove(image.id, image.tenantId);
  response.sendStatus(204);
});
```

This is intentionally not a distributed transaction. An upload and a metadata insert cross separate durability boundaries. Consider a worker that stores the bytes, receives a process termination, and never commits its row: the object exists, but no user can safely discover it. The reverse ordering has a different shape: the row says `pending_delete`, the delete request times out after the storage service has accepted it, and a retry must be harmless whether the object is already gone or still present. A production implementation should make each transition observable, write a cleanup intent or durable job when it matters, retry idempotent deletion, and reconcile objects with no matching metadata on a schedule. Keep the original key in that intent. Reconstructing it from a newer naming rule can point a later worker at the wrong tenant. The lifecycle policy then covers objects that remain after the retention delay. It is a safety net, not proof that a user-visible delete happened immediately.

## How should old AI images be deleted without trusting a user folder?

Explicit deletion expresses a product event: a person removes a render, replaces an avatar, deletes a project, or closes an account. The request must query metadata with the caller's tenant ID, use the returned key, and record enough state for a later worker to finish the operation. Returning `404` for a non-owned image is often preferable to revealing whether another tenant's ID exists.

Lifecycle expiration expresses a different rule: objects matching a known scope and age may be removed later. It is useful for abandoned uploads, temporary previews, old intermediate renders, and orphan protection. It cannot inspect database state, user entitlements, or a legal hold. The catch is that a lifecycle rule is not suitable when a user-facing control promises immediate removal, or when eligibility depends on metadata the object store cannot see. Use explicit deletion in those cases; use lifecycle expiration where a key scope and age fully describe the policy.

| Decision | Choose it for | What you give up |
| --- | --- | --- |
| Recorded-key delete | Immediate user action and account erasure | A retryable cross-system workflow |
| Prefix-and-age lifecycle | Orphans and simple retention windows | Immediate timing and metadata awareness |
| Metadata-aware sweeper | Project state, legal holds, or quotas | A query, worker state, and another alert path |

For larger files or unreliable client connections, multipart upload is a separate concern. It divides one object into independently uploaded parts and requires a final completion step or an abort; incomplete multipart uploads require their own cleanup rule. The extra state is usually unnecessary for ordinary generated images, but it can be warranted when artifact size and connection behavior justify it. The AWS documentation describes that lifecycle in detail.

## The checks that keep retention from becoming a quiet cost leak

Before release, test tenant isolation first: a valid session asking for another tenant's image must not reach the storage adapter. Then test a rejected media type, an oversized body, repeated deletes, an object written without a committed metadata row, and a deletion that is resumed from `pending_delete`. These tests target the awkward boundaries, where an otherwise clean handler tends to leave data behind.

Observability should use identifiers that are safe to log. Track operation name, image ID, request correlation ID, byte count, upload latency, delete attempts, cleanup age, and the count of unmatched objects. Do not log credentials, signed access tokens, or prompt text by default. A rising difference between stored objects and active metadata is a useful retention signal; it tells the team to inspect reconciliation before retained bytes become an unplanned bill.

Deployment order matters because policy is part of the feature. Apply and verify retention on a disposable prefix, deploy the metadata fields and cleanup worker, then enable image writes. During a key-format change, deletes must continue to use each row's recorded key, not a reconstructed prefix. The operational checklist is prose-sized: authenticate before lookup, generate keys on the server, bound inputs, keep credentials out of application logs, make cleanup retryable, and monitor the object-to-metadata gap. That is enough structure for a small team to ship without pretending storage is just a folder.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
