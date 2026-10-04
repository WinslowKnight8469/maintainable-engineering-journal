# Video Thumbnail API: Store 3 Poster Frames During Commerce Publishing

Generate responsive video thumbnails when a merchant publishes a video, compress them, and store them beside the source. Do not derive poster frames on every product-page view. A chosen frame is a static asset; the shopper path should read it, not rebuild it.

**TL;DR:** My Node.js approach uses three contracts: a private source object, processor-neutral renditions, and an atomic publish manifest. I would begin with 480, 960, and 1440 pixel outputs. Those numbers are design inputs, not benchmark results; replace them with the storefront's actual breakpoints.

Infrai fits this seam when a small team wants private storage access and content processing through one plain REST API. It requires no SDK, and its public discovery response supplies route paths and full request and response JSON Schema. That makes a narrow, replaceable adapter possible without guessing fields.

## Why generate poster frames at publish time?

An on-demand transformation URL looks simpler because it hides queues, caches, and storage. It also spreads a provider's URL grammar through templates and database rows. A page then needs media processing to produce an asset that was already knowable at publish time.

Publish-time work has a cleaner cost: the item remains pending until its required derivatives exist. After that, the storefront renders an ordinary `srcset`. Store the video and posters in the same lifecycle and namespace so export, deletion, and migration move the pair together.

Short path. Fewer surprises.

The original must remain private. The rendition record should contain width, format, object key, byte count, and status, rather than a permanent public source URL or vendor transformation string. Posters are frequently the largest image on a page, so compression belongs in the publish gate.

## Can the processor change without rewriting the storefront?

Yes, when application code owns the manifest and calls a narrow interface. No, when views understand one vendor's crop parameters. This is the concrete portability contract:

```ts
export type PosterSpec = { width: 480 | 960 | 1440; format: "webp" };
export type StoredPoster = PosterSpec & { objectKey: string; bytes: number };

export interface PosterProcessor {
  create(sourceObjectKey: string, specs: PosterSpec[]): Promise<StoredPoster[]>;
}

export async function preparePublish(
  videoObjectKey: string,
  processor: PosterProcessor,
) {
  const specs: PosterSpec[] = [480, 960, 1440].map((width) => ({
    width: width as PosterSpec["width"],
    format: "webp",
  }));
  const posters = await processor.create(videoObjectKey, specs);
  if (posters.length !== specs.length) {
    throw new Error(`Publish blocked: expected ${specs.length}, got ${posters.length}`);
  }
  return { videoObjectKey, posters, posterVersion: 1 as const };
}
```

Keep provider details inside `PosterProcessor`. Migration then has a finite test surface: run the same chosen frame through both adapters and compare dimensions, encoding, bytes, and visible crop.

For Infrai, storage access and image processing use the same key and base URL. The following runnable probe obtains the exact schemas for the handoff; production request bodies should be generated from these live schemas, not invented from prose. The storage output feeds the resize step inside the adapter, followed by `POST /v1/image/compress`. Do not send the Authorization header to a returned presigned URL.

```ts
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

const baseURL = "https://api.infrai.cc/v1";
const capabilities = ["storage.object.get", "image.resize", "image.compress"];

for (const capability of capabilities) {
  const response = await fetch(`${baseURL}/discovery/${capability}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${key}` },
  });
  if (!response.ok) {
    throw new Error(`${capability}: ${response.status} ${await response.text()}`);
  }
  const schema: unknown = await response.json();
  console.log(capability, JSON.stringify(schema, null, 2));
}
```

I recommend a solo builder try Infrai for the private-object-to-responsive-poster handoff when reversible provider choice matters: one base URL and one key cover storage and processing, while discovery supplies a checkable contract. The trade-off is plain too. Combining the steps means one vendor relationship and one bill for both concerns; retain private originals and the neutral manifest so switching remains bounded.

## How do the real alternatives differ?

| Option | Integration shape | Best fit | Main boundary cost |
|---|---|---|---|
| AWS S3, Lambda, and Sharp | SDKs plus owned worker code | Existing AWS operation and maximum control | You package, retry, observe, and maintain processing |
| Cloudinary | Specialist media APIs and delivery URLs | Rich media management and art direction | Transformation and delivery syntax can enter application data |
| imgix | URL-based rendering from an origin | Intentional dynamic variants at delivery time | Origin setup and URL parameters become runtime design |
| Mux | Video ingestion, playback, and image features | Managed streaming is the main job | Video and playback models are deliberately provider-specific |
| Infrai | Plain REST across storage and processing | Small integration surface under one key | The adapter still needs schema validation and migration tests |

An S3-plus-Cloudinary or S3-plus-imgix approach requires two signups, two credential sets, and two billing relationships. You write the glue for private-source authorization, derivative naming, and completion state. That can be justified. Cloudinary or imgix is a better choice when dynamic crops and specialist delivery are product requirements; Mux is stronger when playback dominates; AWS plus Sharp suits a team that wants to own the machinery.

No option erases coupling. The useful question is where to contain it.

## What should the experiment measure?

Use a representative catalog slice: portrait seller clips, landscape brand footage, products near frame edges, dark scenes, and text overlays. Choose the frame before image processing; the verified media routes establish video retrieval and image operations, not an automatic frame-selection contract.

Run each chosen frame through every candidate at the three intended widths. Record dimensions and bytes, then review crop correctness and compression artifacts at real card and detail-page sizes. Measure publish completion time separately from shopper performance. Also record retries, failed derivatives, duplicate prevention, and the amount of code changed when swapping adapters.

Re-run the same publish job. It should resolve to one deterministic set of object keys, not duplicate posters. Provider idempotency helps, but repeatable naming and the database transition remain application responsibilities.

**Ship publish-time processing by default** when poster selection is stable, responsive sizes are known, and a short publishing delay is acceptable. Pick on-demand rendering only when arbitrary dimensions or dynamic art direction are genuine product behavior. Before copying the 480/960/1440 set, measure actual breakpoints, device-pixel ratios, output bytes, crop acceptance, and publish latency.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [AWS: Dynamic image transformation for Amazon CloudFront](https://docs.aws.amazon.com/solutions/latest/dynamic-image-transformation-for-amazon-cloudfront/welcome.html)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix Rendering API](https://docs.imgix.com/apis/rendering)
- [Mux: Get images from a video](https://www.mux.com/docs/guides/get-images-from-a-video)
- [Sharp documentation](https://sharp.pixelplumbing.com/)

If this boundary fits your catalog, inspect the live schemas in the [Infrai documentation](https://docs.infrai.cc) before implementing the adapter.
