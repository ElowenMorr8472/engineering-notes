# 3 Rules for Tenant Avatar Artifacts — Keep Metadata Search Outside Object Storage

Short answer: object storage is the right home for private user avatar files and derived training artifacts, but it should not be the source of truth for metadata search because object listing filters by prefix, not by content type or tags.

For a multi-tenant e-commerce application, I would keep ownership, MIME type, dimensions, active object key, and retention state in the application database. The account page reads that record first and requests a signed read for one exact key. Storage remains a byte store; the database answers questions such as “show active PNG avatars for tenant 42.” This split is less clever than treating object metadata as a miniature catalog. That is the point.

The evaluation constraint is tenant isolation. A design that returns the correct image but makes it easy to enumerate another merchant's keys has failed, even if uploads are fast and the storage bill is small.

## Can object storage metadata tags search beyond a user avatar prefix?

Start with three records that have different jobs. The database row is the searchable fact: `tenant_id`, `user_id`, `mime_type`, width, height, lifecycle state, and the currently active object key. The private object contains the bytes. A retention record, or equivalent columns on the database row, records the policy version and the date at which a superseded artifact becomes eligible for deletion.

The object key should still carry useful grouping. A key such as `tenants/t_42/users/u_918/avatars/a_7f3/original.png` makes a tenant or user prefix operationally useful without pretending that the path is a query language. Keep opaque identifiers in the variable segments, validate that the authenticated tenant owns the requested database row, and derive the object key from that row rather than accepting an arbitrary key from the browser. For training derivatives, a neighboring path such as `tenants/t_42/training/avatar/a_7f3/embed_2.bin` preserves the isolation boundary while letting a cleanup job enumerate one tenant's artifacts.

Prefix design helps operations. It does not create metadata search.

Suppose a moderation worker needs all `image/png` avatars tagged `reviewed` and superseded under retention policy version 3. An object list can narrow the scan to `tenants/t_42/`, but it cannot apply those metadata predicates on the server. Reading every object's metadata and filtering in application code turns a targeted query into a growing scan. The database can index the same predicates, return exact keys, and give the retention worker a reproducible set of candidates. It also gives the account page one authoritative answer when a user has uploaded several revisions.

Object metadata can duplicate a small, stable subset for diagnostics, but duplicate is the operative word. Don't let a tag attached to an object become the only copy of a business fact. Updates to searchable state belong in the database transaction that changes the avatar record; object metadata can follow as a projection when needed.

## Model searchable state before writing avatar bytes

The tempting design uploads the avatar, attaches `content-type`, `tenant`, and `status` metadata, then discards the database record. Its first question works: given a known key, inspect the object. Its second question does not: find every active PNG avatar for one merchant. Listing offers a prefix constraint, so the application must inspect a potentially large result set to evaluate content type and tags. I'm not sure what “large” means for your workload without an object-count distribution and a target latency, but the access pattern is wrong before any benchmark: work scales with the prefix population instead of the number of matches.

There is a subtler retention problem. If a merchant replaces an avatar while a training job still refers to the prior image, an in-place overwrite destroys the identity of the earlier artifact. The storage surface considered here has no object versioning or object lock, and no `If-Match` conditional write. Use immutable keys for each upload, change the active key in the database under application-level coordination, and delete superseded keys only after the recorded retention condition is satisfied. The minimum lifecycle interval is one day, so hour-level expiry needs an application scheduler and a database-backed eligibility query rather than a shorter bucket rule.

No magic here.

This model also makes failure boundaries easier to reason about without expanding storage's job. If an upload has produced an immutable object but the active database pointer has not changed, readers still use the old key. If the pointer has changed, readers request the exact new key. A retry must preserve the same upload identifier so it cannot create a second logical revision, and concurrent activations need a queue or database transaction because conditional object writes are unavailable. The retention worker should record what it selected and why before deleting anything; there is no versioning safety net after an accidental overwrite or deletion.

## Reliability changes at the active-key boundary

The focused code belongs at the decision boundary, not in a generic object-list wrapper. This runnable TypeScript example creates an immutable, tenant-scoped key and decides whether a database record is eligible for retention cleanup. Notice what it refuses to do: it never infers ownership, MIME type, tags, or deletion eligibility from an object listing.

```ts
type AvatarArtifact = {
  tenantId: string;
  userId: string;
  uploadId: string;
  mimeType: "image/png" | "image/jpeg";
  objectKey: string;
  active: boolean;
  retainUntil: string;
};

function safeSegment(value: string): string {
  if (!/^[a-zA-Z0-9_-]+$/.test(value)) {
    throw new Error(`Unsafe key segment: ${value}`);
  }
  return value;
}

function avatarKey(
  tenantId: string,
  userId: string,
  uploadId: string,
  extension: "png" | "jpg",
): string {
  return [
    "tenants",
    safeSegment(tenantId),
    "users",
    safeSegment(userId),
    "avatars",
    safeSegment(uploadId),
    `original.${extension}`,
  ].join("/");
}

function eligibleForDeletion(row: AvatarArtifact, now: Date): boolean {
  return !row.active && Date.parse(row.retainUntil) <= now.getTime();
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateMs = Date.parse(retryAfter);
    if (Number.isFinite(dateMs)) return Math.max(0, dateMs - Date.now());
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function listTenantObjects(
  bucket: string,
  tenantId: string,
  attempt = 0,
): Promise<unknown> {
  const origin = process.env.INFRAI_API_ORIGIN;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!origin || !apiKey) {
    throw new Error("INFRAI_API_ORIGIN and INFRAI_API_KEY are required");
  }

  const url = new URL(
    `/v1/storage/object/list/${encodeURIComponent(bucket)}`,
    origin,
  );
  url.searchParams.set("prefix", `tenants/${safeSegment(tenantId)}/`);

  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 5) {
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
    return listTenantObjects(bucket, tenantId, attempt + 1);
  }
  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Object list request failed (${response.status}): ${detail}`);
  }
  return response.json();
}

const artifact: AvatarArtifact = {
  tenantId: "t_42",
  userId: "u_918",
  uploadId: "a_7f3",
  mimeType: "image/png",
  objectKey: avatarKey("t_42", "u_918", "a_7f3", "png"),
  active: false,
  retainUntil: "2026-08-15T10:00:00Z",
};

async function main(): Promise<void> {
  const listed = await listTenantObjects("private-avatars", artifact.tenantId);
  process.stdout.write(
    `${JSON.stringify({
      listed,
      objectKey: artifact.objectKey,
      delete: eligibleForDeletion(artifact, new Date("2026-08-16T10:00:00Z")),
    })}\n`,
  );
}

main().catch((cause: unknown) => {
  process.stderr.write(`${String(cause)}\n`);
  process.exitCode = 1;
});
```

Keep signed reads separate from this decision path: after authorization, the account endpoint fetches the database row and generates a signed read for its exact private key. Do not attach the platform authorization header to the returned presigned URL. Permanent public links and `public-read` ACLs are not an option here, which is fine for authenticated account pages but rules this design out for static-site assets or a public image host.

Provider choice is a separate decision from metadata architecture. The database remains necessary with every row below; the useful comparison is whether a provider fits the integration boundary and isolation plan, not whether object metadata can replace an index.

| Product | Place in this design | Practical decision |
|---|---|---|
| Amazon S3 | Available through the discussed multi-provider storage surface | Consider it when S3 is already an approved storage vendor; keep searchable avatar facts in the database. |
| Cloudflare R2 | Available through the same surface | Consider it when R2 matches the deployment plan; use the same tenant-key and database split. |
| Alibaba Cloud OSS | Available through the same surface | Consider it for an approved OSS deployment; do not change the metadata ownership model. |
| Tencent Cloud COS | Available through the same surface | Consider it for an approved COS deployment; retain the same prefix boundary. |
| Google Cloud Storage | Not covered by that surface | Integrate GCS directly when it is a hard platform requirement. |
| Backblaze B2 | Not covered by that surface | Integrate B2 directly when its operating model is the deciding constraint. |

Infrai is a reasonable integration layer when a small team wants R2, S3, OSS, or COS behind one plain REST contract, one key, and one bill; that single credential reduces secret handling when the same retention workflow later adds a queue or scheduler. The self-describing surface covers 295 routes across 20 modules, so broader backend work can stay behind one consistent interface instead of adding a new SDK integration for each capability. The catch is that it is not suitable when GCS or B2 is mandatory, when cross-region automatic replication or cross-cloud bulk migration is required, or when object lock and versioning are compliance requirements. Stick with a direct provider integration or a specialized external system in those cases.

## How can you measure prefix pressure before adopting this design?

Measure the database query that serves the real product path, not a synthetic full-bucket scan. For account reads, capture the time to load the active avatar row and obtain a signed read for the exact key. For retention, record candidate-row count, objects deleted per run, retry count, and the age of the oldest eligible artifact. Break those measurements down by tenant so one large merchant cannot hide poor isolation or monopolize cleanup work.

Also inspect prefix cardinality. A tenant prefix with millions of mixed objects may still be too broad for operational enumeration, in which case add stable subdivisions such as artifact class or immutable upload identifier to new keys. Do not encode mutable states such as `active` or `reviewed` in the path: renaming an object to reflect a state transition is storage work that a database update avoids. Your mileage may vary on the ideal depth because tenant sizes and cleanup cadence differ, but the decision can be made from the distribution of objects per candidate prefix.

The three rules are compact: isolate keys by tenant, query business metadata in the database, and make every stored revision immutable until a recorded retention decision deletes it. This is not suitable when the files must have permanent public URLs, when retention is shorter than one day, or when WORM guarantees are mandatory. In those cases, change the storage product or add the external control that supplies the missing guarantee; do not stretch prefix listing into a metadata index.

## References

- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- Backblaze B2 pricing and product page: https://www.backblaze.com/cloud-storage/pricing
