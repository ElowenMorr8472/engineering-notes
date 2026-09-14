# Node.js Campaign Asset Retention: Explicit Disposal for Images and Generated Videos

Short answer: treat every uploaded image and generated video as a temporary campaign asset with an explicit, auditable deletion state; do not let a storage timeout become your retention policy. Keep quality decisions separate from bandwidth decisions, and require a recorded delete result before the campaign can be closed.

In a fintech media library, an asset might be a product screenshot, a compliance-approved banner, or a short generated explainer. Search quality matters because a missed tag hides the right evidence. Bandwidth matters because campaign teams upload large originals and generate several video variants. The useful design is a small lifecycle: ingest, inspect, tag, publish a derivative, expire, then delete every copy you own.

## How should campaign assets move from upload to explicit deletion?

Give each asset an immutable ID and a retention deadline at ingest. Store metadata beside the object, not only in a filename: campaign ID, media type, checksum, creation time, retention class, and deletion state. A worker can then process an image or generated video without guessing which campaign it belongs to.

The state machine below is intentionally boring. Boring is good when an auditor asks what happened to a file.

```ts
type MediaKind = "image" | "generated-video";
type AssetState = "received" | "tagged" | "published" | "expired" | "deleted";

interface CampaignAsset {
  id: string;
  campaignId: string;
  kind: MediaKind;
  objectKey: string;
  checksum: string;
  deleteAfter: string;
  state: AssetState;
}

interface AssetStore {
  get(id: string): Promise<CampaignAsset | undefined>;
  markExpired(id: string, at: string): Promise<void>;
  deleteObject(key: string): Promise<void>;
  markDeleted(id: string, at: string): Promise<void>;
}

export async function expireAsset(
  store: AssetStore,
  id: string,
  now = new Date(),
): Promise<"deleted" | "not-due" | "missing"> {
  const asset = await store.get(id);
  if (!asset) return "missing";
  if (new Date(asset.deleteAfter) > now) return "not-due";
  if (asset.state === "deleted") return "deleted";

  await store.markExpired(id, now.toISOString());
  await store.deleteObject(asset.objectKey);
  await store.markDeleted(id, now.toISOString());
  return "deleted";
}
```

The delete operation is idempotent. A retry after a worker restart should produce the same final state, while the event log keeps the first expiry time and the last successful deletion time. In practice, I also keep a tombstone containing the checksum and object key. The bytes are gone, but the record proves which bytes were removed.

Generated video needs one extra check: a visible preview, poster frame, audio track, and any transcoded rendition are separate objects. Deleting the manifest alone is not deletion. Enumerate the asset's descendants, delete each key, and only then mark the parent as deleted. For images, include thumbnails and moderation derivatives in the same inventory.

## What fails when retention is implicit?

The first failure is orphaned derivatives. A resize service writes `thumb/` and a cache writes another copy, but the campaign database knows only the original key. The second is an ambiguous clock: “seven days” measured from upload, approval, or publication produces different answers. Pick one event, store it in UTC, and test the boundary at one second before and after expiry.

The third failure is a false success. Object deletion can be acknowledged before a queue consumer records the audit event. Make the audit write part of the same workflow and expose a metric for assets stuck in `expired`. A small reconciliation job can compare metadata rows with object listings; it should report discrepancies for review instead of silently deleting more data. That comparison deserves more attention than it usually gets: an image may have a thumbnail in a different prefix, a video encoder may leave an audio-only rendition, and a CDN may still serve a cached response after the origin object is gone. I write the expected object set into the metadata row at ingest, then compare that set during deletion. When the sets differ, the worker pauses the asset and emits a review event with the missing key, rather than guessing. This adds a little bookkeeping, but it prevents a green dashboard from hiding bytes that remain reachable.

Keep it explicit.

Quality and bandwidth pull in different directions. Keeping the original image improves future tagging, yet repeatedly downloading it is expensive. I've learned to set a 2 MB preview ceiling for the tagging path and measure recall against the original before changing it; the exact ceiling is a policy choice, not a universal benchmark. Generate tags near ingest, retain only the fields needed for search, and pass a bounded preview to downstream classifiers. For generated video, sample a fixed set of frames and retain those tags after the binary expires, provided your policy allows derived metadata to outlive the source.

One hard lesson: “temporary” is not a storage class. It is a business rule that needs an owner, a deadline, and an observable transition.

## A runnable Node.js worker for the deletion boundary

The worker should claim due rows in short batches, use a lease to prevent two workers deleting the same asset, and emit structured events. The example keeps the storage interface generic so it works with an S3-compatible service, a filesystem adapter, or a test double.

```ts
type DeleteEvent = {
  assetId: string;
  campaignId: string;
  kind: MediaKind;
  keys: string[];
  deletedAt: string;
};

async function deleteDueBatch(
  db: {
    claimDue(limit: number): Promise<CampaignAsset[]>;
    descendants(id: string): Promise<string[]>;
    appendEvent(event: DeleteEvent): Promise<void>;
  },
  store: Pick<AssetStore, "deleteObject">,
  limit = 25,
): Promise<number> {
  const due = await db.claimDue(limit);
  let completed = 0;
  for (const asset of due) {
    const keys = [asset.objectKey, ...(await db.descendants(asset.id))];
    for (const key of keys) await store.deleteObject(key);
    const deletedAt = new Date().toISOString();
    await db.appendEvent({ assetId: asset.id, campaignId: asset.campaignId, kind: asset.kind, keys, deletedAt });
    completed += 1;
  }
  return completed;
}
```

`claimDue` should move rows into an in-progress state with a lease expiry. If the process stops after deleting bytes but before the event write, the next run must safely retry both operations and deduplicate the event by `assetId` plus deletion timestamp. This is the same discipline as a payment outbox: the durable record is what lets operations explain the side effect.

Test the worker with a fake store that records keys. Include cases for an image with two thumbnails, a video with poster and renditions, an already deleted asset, and a lease that expires mid-run. Assert that no key outside the asset's descendant set is touched. I am not sure every object provider gives identical listing consistency, so the reconciliation interval should be a measured operational choice, not a hidden assumption.

## Which retention policy is suitable for a fintech campaign?

Write the policy as a table in your design review, then encode the same values in configuration:

| Asset or record | Suggested boundary | Reason to keep it | Deletion proof |
| --- | --- | --- | --- |
| Original upload | Campaign close plus a fixed review window | Re-tagging and dispute review | Original key and checksum tombstone |
| Generated video bytes | Publication or rejection plus a shorter window | Re-rendering is usually cheaper than storage | Manifest, poster, audio, and rendition events |
| Search tags and model metadata | Per legal and product policy | Search continuity without the binary | Versioned metadata record |
| Audit events | Per compliance policy | Demonstrates who deleted what and when | Append-only event identifier |

The catch is that this policy is not suitable when investigators must recover the original media after the review window. In that case, keep a controlled archive with separate access and a separately approved retention clock. Stick with a longer-lived archive when legal hold or customer evidence outweighs bandwidth savings; do not stretch a temporary bucket and call it compliant.

A vendor-neutral interface also keeps the decision reversible. A single HTTP contract can sit in front of different object stores, but the contract must expose conditional delete, metadata lookup, and a way to enumerate descendants. If an adapter cannot provide those operations, it is not suitable for explicit campaign deletion, regardless of its advertised throughput.

## Operational checklist for the last mile

Before shipping, have one person trace an asset from upload through search indexing and deletion using only its ID. Verify that generated-video renditions and image thumbnails appear in the inventory, that UTC deadlines survive a restart, and that retries do not duplicate side effects. Alert on expired rows older than the worker lease, deletion events without a matching tombstone, and objects with no metadata row.

Keep bandwidth measurements beside quality measurements. Record bytes read per tag, tagging latency, missed-search samples, and the percentage of assets whose derivatives were found. Those numbers let a solo team adjust preview size or sampling without changing the retention promise. They also make a future storage migration a controlled engineering task instead of a campaign-by-campaign scramble.

The practical decision rule is simple: retain only what improves a named workflow, attach a deadline to it, and make deletion observable. Everything else is an orphan waiting for an audit.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://www.rfc-editor.org/rfc/rfc3339
- https://www.rfc-editor.org/rfc/rfc9110
