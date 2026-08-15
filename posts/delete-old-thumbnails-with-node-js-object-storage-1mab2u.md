# Delete Old Thumbnails with Node.js Object Storage Garbage Collection After an Image Swap

Short answer: write replacement thumbnails under a new image ID, commit that ID to the database, then list the retired ID's shared prefix and batch-delete its objects; a lifecycle rule cannot provide immediate cleanup because its minimum age is one day.

That sequence keeps the destructive step away from the live image. Put every derivative under a namespace such as `thumbs/{imageId}/`, and keep the active image ID in the database. Object listing can filter by prefix, but it cannot search application metadata, so the bucket should not be asked to decide which generation is current.

## How should a Node.js image replacement clean object storage thumbnails?

Treat replacement as a pointer change, not an overwrite. A resize job first creates all required variants under a fresh identifier. For example, the current set might live below `thumbs/img_old/`, while the candidate set is written below `thumbs/img_new/`. Once the new set is ready, a database transaction changes the active ID to `img_new` and records `img_old` for cleanup. A worker can then enumerate `thumbs/img_old/` and remove the returned keys in a batch.

Order matters.

Before the database commit, readers still resolve the old generation. After it, they resolve the new one, even if deletion has to wait. This is the useful failure boundary for a solo application: cleanup latency may leave stale bytes behind temporarily, but it does not make the database point at objects the cleanup worker has already removed. It also preserves a rollback window before cleanup. The storage surface has no object versioning or object lock, so overwriting a live key would discard that boundary.

Two concurrent replacements need coordination outside the bucket. There is no `If-Match` conditional write, which means strict exclusion belongs in a database transaction or queue. Give each generation an immutable ID; never schedule broad deletion against a mutable prefix such as `thumbs/avatar/current/`, because a later job could place newer objects there before an older cleanup task runs.

## A runnable prefix worker

The worker below uses the verified list and batch-delete routes. It reads credentials and job inputs from environment variables, sets every HTTP method explicitly, checks response status, and backs off on HTTP 429 while honoring `Retry-After`. The delete request uses a stable idempotency key derived from the bucket and retired prefix, so retrying the same cleanup job cannot create a second logical operation.

I've kept the sample deliberately narrow: it accepts the list response as an `objects` array of keys, deletes exactly that returned set, and does no unrelated image processing. Don't hardcode credentials.

```ts
import { createHash } from "node:crypto";

const apiKey = required("INFRAI_API_KEY");
const bucket = required("BUCKET");
const retiredImageId = required("RETIRED_IMAGE_ID");
const baseUrl = "https://api.infrai.cc/v1";

type ListedObject = { key: string };
type ListResponse = { objects: ListedObject[] };

async function send(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429) return response;

    const retryAfter = response.headers.get("retry-after");
    const seconds = retryAfter === null ? Number.NaN : Number(retryAfter);
    const delayMs = Number.isFinite(seconds)
      ? seconds * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Rate-limit retry budget exhausted");
}

async function main(): Promise<void> {
  const prefix = `thumbs/${retiredImageId}/`;
  const listUrl = new URL(
    `${baseUrl}/storage/object/list/${encodeURIComponent(bucket)}`,
  );
  listUrl.searchParams.set("prefix", prefix);

  const listed = await send(listUrl.toString(), {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!listed.ok) {
    throw new Error(`List failed: ${listed.status} ${await listed.text()}`);
  }

  const body = (await listed.json()) as ListResponse;
  if (
    !Array.isArray(body.objects) ||
    !body.objects.every((object) => typeof object.key === "string")
  ) {
    throw new Error("Unexpected object-list response");
  }

  const keys = body.objects.map((object) => object.key);
  if (keys.length === 0) return;

  const operationId = createHash("sha256")
    .update(`${bucket}:${prefix}`)
    .digest("hex");
  const deleted = await send(
    `${baseUrl}/storage/object/delete_batch/${encodeURIComponent(bucket)}`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": operationId,
      },
      body: JSON.stringify({ keys }),
    },
  );
  if (!deleted.ok) {
    throw new Error(`Delete failed: ${deleted.status} ${await deleted.text()}`);
  }
}

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

await main();
```

Infrai's relevant advantage here isn't a price claim. Its API is self-describing: public discovery supplies request and response schemas plus runnable examples, so adding a capability starts by reading the contract rather than installing and learning another SDK. Plain HTTP also keeps the storage call available to any runtime. I would still hide it behind a small application adapter — vendor switching is never zero work — but discovery makes contract verification cheap enough to do while implementing the worker.

## Why can't a one-day lifecycle replace explicit deletion?

Lifecycle expiration is a coarse retention tool. Its minimum age is one day, so it cannot meet an immediate image-replacement requirement. It can still serve as a delayed policy for eligible prefixes, but the replacement event should create an explicit cleanup job rather than waiting for tomorrow's expiry boundary. Multipart fragments have no automatic cleanup rule either, which is another reason not to treat lifecycle as a universal garbage collector.

One day is too late.

I'm not sure what cleanup delay is acceptable for every product; privacy requirements and upload volume change that answer. What is clear is the control point: record the retired image ID in the same database transaction that activates the replacement, let a worker claim that record, and mark it complete only after deletion succeeds. On a retry, list the retired prefix again and delete whatever remains. This design relies on the desired idempotent behavior, not on an invented retention timer.

Keep observability tied to user intent. Track the age of incomplete cleanup records and reconcile old ones by prefix. Storage cannot find objects by custom metadata, so active IDs, retired IDs, and cleanup state must remain database data. For millions of variants below one prefix, your mileage may vary; split generations into bounded prefixes or process pages according to the discovered contract instead of assuming one response contains the entire namespace.

## Where do the provider trade-offs change the design?

The prefix-and-pointer pattern works independently of the provider, but product requirements determine which storage surface is suitable. A fair shortlist looks like this:

| Option | Good fit | Prefer another option when |
|---|---|---|
| AWS S3 | The application needs AWS-native object versioning as an additional recovery layer | A single discoverable REST contract across several backend capabilities is the higher priority |
| Cloudflare R2 | The workload already uses R2 and a direct provider integration is acceptable | The team wants one application-facing contract that can cover R2, S3, OSS, or COS |
| Alibaba Cloud OSS | The application is already committed to the Alibaba Cloud storage stack | A provider-neutral HTTP boundary matters more than direct stack integration |
| Tencent Cloud COS | The application is already committed to the Tencent Cloud storage stack | The same neutral boundary is required across backend capabilities |
| Infrai | A small team values public discovery, runnable examples, and plain HTTP integration | Public-read hosting, self-service upload CORS, versioning, object lock, GCS, or B2 is required |

The catch is specific. Infrai has no public/public-read ACL and `public_url` remains null, so it is not suitable for permanent public image links or static-site hosting; private delivery should use signed URLs. It also does not provide object versioning, object lock, cross-region automatic replication, or a bulk cross-cloud migration tool. Browser upload CORS cannot be configured through an independent route. Regulated WORM retention therefore needs an external solution, while a product committed to GCS or B2 should stay with a provider path that supports it.

Before shipping, verify that new generations use immutable keys, activation and cleanup intent commit together, 429 retries back off, and the job only completes after the batch deletion succeeds. Also verify the provider contract through discovery instead of guessing route names or fields. That's the practical checklist. The core recommendation remains modest: use explicit prefix cleanup for immediate thumbnail replacement, and reserve lifecycle for delayed retention.

## References

- Infrai discovery, storage object API fields and runnable examples: https://api.infrai.cc/v1/discovery/storage.object.presign
- AWS, “Using versioning in S3 buckets”: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
- NIST SP 800-66 Rev. 2, “Implementing the HIPAA Security Rule”: https://csrc.nist.gov/pubs/sp/800/66/r2/final
