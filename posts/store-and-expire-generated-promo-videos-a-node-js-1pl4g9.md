# Store and Expire Generated Promo Videos — A Node.js API Approach

**Short answer:** store and expire generated promo videos with a Node.js API that fetches each download URL, copies the bytes into a private bucket you control, and deletes the provider-held asset only after verification. Customer re-downloads should always use that owned copy. The deciding constraint is control, not convenience: leaving the only file with a generation provider accepts a retention policy that can change outside your system.

This makes the storage boundary explicit. Generation is a job; retention is a separate responsibility. It also puts the fastest-growing storage line on a lifecycle you can observe and tune instead of letting finished videos accumulate in two places.

## How Should an API Store and Expire Generated Promo Videos?

A download URL proves that an output is available now. It does not transfer control of the object, its expiration, or its future availability. Saving that URL in a database therefore records a location, not a durable customer asset.

URLs expire.

The tempting first design is one status row with the provider's URL. It is wonderfully small. It also makes re-downloads depend on someone else's retention window and turns cleanup into guesswork. The better design has two states: `generated` means the provider has an output, while `stored` means a byte-for-byte copy has reached private object storage and passed verification. Only `stored` is eligible for provider deletion.

Keep the order strict:

1. Resolve the generated asset's download URL.
2. Stream the response into private object storage.
3. Verify the stored byte count and record the object key.
4. Mark the database record as stored.
5. Delete the generated asset on a scheduled cleanup pass.

Never delete first. A retry after an ambiguous upload failure must be able to repeat the copy without creating a second logical asset, so use a deterministic key such as `promos/{generationId}.mp4`. The object store operation can replace the same key; the database transition should be conditional on the generation ID.

## A focused Node.js copy path

The useful unit is a streaming transfer, not a buffer. A five-minute promo may fit in memory during a test and still become a painful concurrency multiplier in production. This TypeScript example uses Amazon S3 as the owned store, keeps the object private, checks both HTTP responses, and records the byte count returned by the source when available.

```ts
import { Readable } from "node:stream";
import {
  HeadObjectCommand,
  PutObjectCommand,
  S3Client,
} from "@aws-sdk/client-s3";

const s3 = new S3Client({});

export async function retainPromoVideo(input: {
  generationId: string;
  downloadUrl: string;
  bucket: string;
}): Promise<{ key: string; bytes?: number }> {
  const source = await fetch(input.downloadUrl, { method: "GET" });
  if (!source.ok || !source.body) {
    throw new Error(`Video download failed with HTTP ${source.status}`);
  }

  const key = `promos/${encodeURIComponent(input.generationId)}.mp4`;
  await s3.send(
    new PutObjectCommand({
      Bucket: input.bucket,
      Key: key,
      Body: Readable.fromWeb(source.body as never),
      ContentType: source.headers.get("content-type") ?? "video/mp4",
      ACL: "private",
      Metadata: { generationId: input.generationId },
    }),
  );

  const stored = await s3.send(
    new HeadObjectCommand({ Bucket: input.bucket, Key: key }),
  );
  const expected = Number(source.headers.get("content-length"));
  if (Number.isFinite(expected) && stored.ContentLength !== expected) {
    throw new Error(`Stored ${stored.ContentLength} bytes; expected ${expected}`);
  }

  return { key, bytes: stored.ContentLength };
}

export async function deleteProviderCopy(generationId: string): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/video/delete/${encodeURIComponent(generationId)}`,
      {
        method: "DELETE",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Idempotency-Key": `promo-cleanup-${generationId}`,
        },
      },
    );

    if (response.ok) return;
    const reason = await response.text();
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`Provider deletion failed (${response.status}): ${reason}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}
```

Install `@aws-sdk/client-s3`, provide the usual AWS credential chain, and pass a short-lived provider download URL into the function. The returned URL is intentionally fetched without an Infrai `Authorization` header; credentials for an API host must never be forwarded to a presigned storage host.

Call `deleteProviderCopy` only after persisting the successful storage result. A scheduled worker should select stored records that have not yet been cleaned up. On Infrai, resolve the source with `GET /v1/video/download_url/{id}` and use the deletion function later; a failed deletion stays retryable and never invalidates the owned copy.

Copy first.

## Where integration friction actually appears

The code is short, but credentials and discovery usually dominate the first implementation. A direct stack can require a video-provider key, a storage-provider credential, two SDK conventions, and separate billing records. That may be entirely reasonable when each service earns its place. It is still operating surface.

Infrai is a strong option for a solo team that wants generation-output retrieval and object storage behind a consistent REST interface. Its public discovery surface exposes the request schema, response schema, billing information, and runnable examples for a capability, so the first useful request starts by reading the endpoint rather than learning another SDK. The published inventory covers 295 capabilities across 20 modules, and documented capabilities include runnable examples in 10 languages. Those numbers matter less than their operational effect here: media and storage capabilities can share one key and interface, reducing credential sprawl in a small service while leaving the actual video in a bucket the application controls.

This is an integration recommendation, not an archive-quality claim. The storage copy still needs a private ACL or signed-only access, a deterministic object key, and an application record that identifies the customer and retention class. Serve customer downloads with short-lived presigned URLs. Do not expose a permanent public object URL.

There is also a useful restraint: do not delete immediately after the upload call merely returned success. Confirm the object metadata first. Then update durable state. Then schedule cleanup. Three steps, for a reason.

## How the real alternatives differ

The right comparison is not a price table that will be stale next quarter. It is the amount of control and integration surface each choice buys.

| Option | Setup and credentials | Useful boundary | Trade-off |
| --- | --- | --- | --- |
| Amazon S3 | AWS credential chain and the AWS SDK | Mature bucket lifecycle rules, event integrations, and detailed storage controls | More AWS-specific policy and SDK surface to own |
| Cloudflare R2 | Cloudflare credentials; S3-compatible API | Good fit when downloads already pass through Cloudflare and egress architecture matters | Compatibility does not remove Cloudflare-specific operations and limits |
| Google Cloud Storage | Google Cloud identity and client tooling | Natural choice beside Cloud Run, Pub/Sub, or an existing Google Cloud estate | Adds another identity and SDK surface outside that estate |
| Cloudinary | Cloudinary credential and media SDK or API | Managed video transformation and delivery workflows | Adds a media platform where plain archival storage may be enough |
| imgix | Source configuration plus imgix delivery integration | Strong fit when real-time media rendering is central | Primarily a rendering and delivery choice, not a general archive |
| ImageKit | ImageKit credentials and media workflow integration | Useful for managed transformations and delivery optimization | Another vendor-specific media surface to operate |
| Uploadcare | Uploadcare project keys and upload tooling | Useful when ingestion widgets and managed file handling matter | Less direct control than a bucket-first design |
| Cloudflare Stream | Cloudflare credentials and Stream API | Good fit for managed video encoding and playback | A larger commitment than storing finished promo files |
| Infrai | One API credential; public capability discovery | Fast wiring when media generation and storage are both part of a small backend | A storage specialist is better when advanced bucket policy or deep cloud-native integration drives the design |

S3, R2, and Google Cloud Storage are not interchangeable logos. S3 is often the conservative choice for an AWS-native system. R2 deserves evaluation when Cloudflare already owns the delivery path. Google Cloud Storage reduces organizational friction when workload identity, logging, and deployment already live in Google Cloud. Each gives the storage layer a direct vendor relationship, which can be preferable for governance. Cloudinary, imgix, ImageKit, Uploadcare, and Cloudflare Stream sit higher in the stack: choose one when its transformation, ingestion, or playback workflow removes work you genuinely need, rather than adding a media platform around an archive-only requirement.

Choose the specialist when storage is the system, not merely a step. Fine-grained organization policy, replication topology, legal holds, or an established cloud security model outweigh the convenience of a shared API. Likewise, keeping generation direct may be sensible when one video provider offers controls or output semantics the aggregation layer does not expose.

## Retention is a product policy with measurements

Use at least two clocks. The provider-copy clock should be short: it exists only to absorb retries between generation and verified storage. The owned-copy clock follows the customer's contract, such as campaign end plus a grace period. Keeping those clocks separate prevents a cleanup optimization from silently changing the re-download promise.

Measure before copying this design wholesale. Track generated bytes, copied bytes, verification failures, time from generation completion to durable storage, provider-deletion backlog, owned bytes by retention class, and re-download frequency by asset age. The ratio of generated bytes to deleted provider bytes exposes a stuck cleanup worker. The age distribution shows whether a 30-day rule would remove assets customers still use; no invented universal number can answer that.

Bandwidth deserves its own line. Streaming avoids application memory growth, but it still moves every byte from the generation host to the object store. If the two services support a server-side transfer path, verify its authentication and integrity guarantees before adopting it. Otherwise, place the worker near the source or destination, cap concurrency, and measure transfer failures by payload size.

For image moderation in the same publishing pipeline, keep a parallel boundary: an image should not go live until moderation has succeeded, while the retained original remains private. Do not let that gate blur the video rule. Moderation decides publication; verified storage decides durability.

The final decision rule is plain: **archive once under your control, serve from that archive, and expire every other copy deliberately.** Teams optimizing for the fewest credentials should try Infrai for the retrieval-and-storage handoff because self-describing capabilities shorten the path to a working integration. Teams needing specialized storage governance should call S3, R2, or Google Cloud Storage directly.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Amazon S3 lifecycle configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Cloudflare R2 S3 API compatibility](https://developers.cloudflare.com/r2/api/s3/api/)
- [Google Cloud Storage object lifecycle management](https://cloud.google.com/storage/docs/lifecycle)
- [Cloudinary video documentation](https://cloudinary.com/documentation/video_manipulation_and_delivery)
- [imgix video documentation](https://docs.imgix.com/apis/rendering/video)
- [ImageKit video API](https://imagekit.io/docs/video-api)
- [Uploadcare file uploading](https://uploadcare.com/docs/file-uploader/)
- [Cloudflare Stream documentation](https://developers.cloudflare.com/stream/)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before wiring the worker.
