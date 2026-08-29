# GDPR App Files: Private Object Storage Architecture for Signed Upload and Download

**Short answer:** Put each tenant's backup objects in private object storage, but keep authorization, snapshot state, and restore decisions in the application database; use short-lived signed upload and download grants only for the browser transfer.

For a learning app serving users in the US and EU, this is the useful split between access control and delivery simplicity. The browser can move a large export without making the application server proxy every byte. The application still decides which tenant may create a snapshot, which snapshot may be restored, and who may download it.

Bytes move directly. Policy does not.

## Should a private file upload architecture use signed download grants?

An authenticated user asks the application to create a backup for one tenant. The application checks membership and role, creates a `pending` snapshot row, chooses an opaque object key, and requests a signed upload grant. The browser uploads directly to private storage. A worker verifies the object and changes the row to `ready`. For download or restore, the application checks the current authorization against that snapshot and issues a grant for the narrow operation.

Start with states.

| State or event | Application invariant | Next action |
|---|---|---|
| `pending` | No user can restore or download the object as a finished backup | Await upload and verification, then mark `ready` |
| `ready` | The object has a catalog record and passed the product's checks | Permit an authorized download or restore job |
| `restore_requested` | One authorized request is recorded for one snapshot | Queue an idempotent worker job |
| `deleted` | The catalog no longer grants access | Delete the object and retain only the audit data required by policy |

This state model exposes the real failure boundary. Suppose a browser retries after losing its response: the upload can arrive twice, but the snapshot creation request must remain idempotent. Suppose a worker verifies a 2 GB export and the process stops before the database update: the object is still not restorable. Suppose an administrator loses tenant membership while a download link is in circulation: the next grant must fail, and the old bearer link must have a short enough lifetime for that residual exposure to be acceptable. Those are product invariants, not storage naming tricks, so I keep them in application code and test them there. The exact recovery window depends on the data classification and threat model; your mileage may vary.

The storage key is not ownership. A key such as `tenant_42/2026-08-10.zip` leaks structure and invites guessing; a random identifier under a controlled prefix is easier to rotate and safer to log. The database record should carry the tenant ID, snapshot ID, object key, creator, content length, declared and observed media type, checksum where available, creation time, and state. It should also record a deletion or retention decision rather than relying on a storage listing to answer product questions.

This is where the access-control versus delivery-simplicity trade-off lands. A direct browser transfer is simple for the data path, but it adds a signing endpoint, CORS configuration, expiry handling, and a completion check. A proxy upload is easier to reason about at first, but your application process becomes the data pipe and needs its own size, timeout, and back-pressure controls. I choose direct transfer for large tenant exports when the control plane is narrow and observable.

## Implementation details for browser grants

The endpoint below is provider-neutral. `storage.sign` represents the adapter owned by the application; it is intentionally not a storage SDK call in the request handler. The browser receives a temporary capability, while the secret stays server-side. A real handler would obtain `user` from the session middleware and persist the row in the same authorization flow.

```ts
type User = { id: string; tenantId: string; role: "admin" | "member" };
type Snapshot = {
  id: string;
  tenantId: string;
  objectKey: string;
  state: "pending" | "ready";
};

type StorageAdapter = {
  sign(input: {
    operation: "upload" | "download";
    objectKey: string;
    contentType?: string;
    expiresInSeconds: number;
  }): Promise<{ url: string; expiresAt: string }>;
};

declare const storage: StorageAdapter;
declare const insertSnapshot: (snapshot: Snapshot) => Promise<void>;
declare const findSnapshot: (id: string) => Promise<Snapshot | null>;

function canManageBackup(user: User, tenantId: string): boolean {
  return user.tenantId === tenantId && user.role === "admin";
}

export async function createUploadGrant(
  user: User,
  tenantId: string,
  contentType: string,
): Promise<{ snapshotId: string; uploadUrl: string; expiresAt: string }> {
  if (!canManageBackup(user, tenantId)) {
    throw new Error("Forbidden");
  }

  const snapshotId = crypto.randomUUID();
  const objectKey = `private/${tenantId}/${crypto.randomUUID()}.backup`;
  const grant = await storage.sign({
    operation: "upload",
    objectKey,
    contentType,
    expiresInSeconds: 300,
  });

  await insertSnapshot({
    id: snapshotId,
    tenantId,
    objectKey,
    state: "pending",
  });

  return { snapshotId, uploadUrl: grant.url, expiresAt: grant.expiresAt };
}

export async function createDownloadGrant(
  user: User,
  snapshotId: string,
): Promise<{ downloadUrl: string; expiresAt: string }> {
  const snapshot = await findSnapshot(snapshotId);
  if (!snapshot || snapshot.state !== "ready" || snapshot.tenantId !== user.tenantId) {
    throw new Error("Not found");
  }

  const grant = await storage.sign({
    operation: "download",
    objectKey: snapshot.objectKey,
    expiresInSeconds: 120,
  });
  return { downloadUrl: grant.url, expiresAt: grant.expiresAt };
}
```

The order deserves attention. Creating a database row after signing creates an orphan if the process stops between those operations; creating it first requires a cleanup policy for pending rows. In production, wrap the row transition and grant request in an idempotent application operation, and give the worker a clear rule for abandoned snapshots. The exact transaction boundary depends on the adapter, so I'm not sure a single universal ordering exists; the invariant does: no download or restore may use a snapshot that the application has not marked `ready`.

I also put a 403 test in the first test pass: a member of tenant A must not obtain a grant for tenant B, even when the snapshot ID is valid. That small test catches the dangerous version of “direct to storage,” where delivery is easy and the authorization boundary quietly disappears.

A signed URL is a bearer capability. Anyone who obtains it can use it until it expires or the storage layer rejects the operation. It is not a substitute for a user session, tenant membership, malware scanning, retention policy, or a deletion workflow. Do not put signed URLs into analytics payloads, exception messages, screenshots, or support tickets. Redact query strings in request logs.

The first test is a 403.

For uploads, allow-list the file types the backup workflow actually needs, enforce a maximum size, generate names on the server, and treat the client-provided type as an input rather than proof. OWASP's file-upload guidance also calls for storage outside the webroot and a malware scan where the threat model requires it. A backup worker should verify the object and metadata before it becomes restorable. If verification cannot finish, keep the state pending or rejected; never make an unverified object a normal download.

For GDPR, the architecture gives the product a place to implement data minimization, retention, deletion, access requests, and regional processing decisions. It does not decide the lawful basis, controller or processor role, transfer mechanism, or appropriate retention period. US and EU are not one homogeneous operational policy. Record the region and policy version with the tenant, and make deletion cover both the database row and the object, including failed or abandoned uploads. I don't treat a region label as legal advice; it is an input to the review and retention process.

## Restore reliability needs its own job and audit trail

Restoring a selected snapshot needs a second authorization decision and a reviewable operation record. The user selects a snapshot ID; the application confirms that it belongs to the tenant, is `ready`, and is allowed for that environment. It then creates a restore job with an idempotency key. A worker reads the object, verifies its manifest and checksum, validates the schema version, and applies changes in a transaction or in explicitly resumable steps.

Do not let the browser turn a signed download URL into an implicit restore command. A download can be a read-only inspection; a restore changes tenant state and deserves a separate role check, confirmation, audit event, and concurrency rule. If two restores can run at once, define which one wins before shipping. My default is one active restore per tenant, with the application rejecting a second request rather than making ordering an accident.

The failure cases are concrete: an expired upload grant, a retry that creates two snapshot rows, a client that stops halfway through a transfer, a snapshot deleted while its download grant is still valid, or a restore worker that dies after applying half a manifest. Each state transition needs an owner, a retry policy, and an observable event. A periodic job can reconcile `pending` rows with object metadata, but it should not silently mark arbitrary objects ready.

The catch is operational ownership. This pattern is not suitable when the product needs permanent public URLs, browser-editable origin policies for every customer, immutable compliance archives, or native cross-region recovery that the chosen storage system already provides. In those cases, select a storage platform with the required controls, or add a dedicated archive and recovery layer; do not pretend signed links supply them.

It is also a poor fit for tiny teams that cannot monitor a worker, reconcile pending objects, test restore integrity, and answer deletion requests. A managed backup product may be the better boundary when restore correctness matters more than a custom file workflow. Conversely, if the application already has a durable job system and needs tenant-specific authorization, keeping the catalog and policy in the app usually avoids coupling business rules to bucket permissions.

The decision rule is short: use private object storage and signed browser transfers when the application can own the control plane. Keep the upload and download grants narrow, make restore a separate job, and document the cases where the storage layer must be replaced or supplemented.

## References

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
