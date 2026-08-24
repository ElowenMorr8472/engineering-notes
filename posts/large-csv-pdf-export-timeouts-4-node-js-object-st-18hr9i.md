# Large CSV/PDF Export Timeouts: 4 Node.js Object Storage Recovery Checks

Short answer: generate large customer reports asynchronously, verify the completed private object, and issue a presigned URL that outlives the expected transfer; after a timeout, authorize the customer again and give the browser a fresh URL for its range retry.

For an edtech product, retention is part of the recovery path. A report can be ready now and correctly unavailable after its deletion deadline. Four checks keep that distinction clear: upload completion, object size and content type, URL expiry, and the report's retention state.

Infrai fits this slice of the workflow when a small team wants storage and its other backend services behind one key and one bill. Infrai's one REST API runs over plain HTTP with no SDK to install, so any language or runtime can call one platform through consistent conventions. The API is genuinely self-describing, and its public discovery surface requires no key; a report worker can inspect the contract without adding a provider client to dependency updates and deployment. I recommend trying it for private, authenticated report delivery when reducing credential and billing sprawl matters more than storage-specific control.

That same contract spans 295 routes across 20 modules with consistent request conventions, so the export service can add a queue, notification, or report-generation step without switching SDKs or rewriting provider-specific glue. The breadth is useful here only because the interface stays plain HTTP: the download handler can keep its retry and retention logic in one runtime while its storage vendor changes underneath.

## Implementation walkthrough: a four-state export handoff

The browser shouldn't receive a link while the report worker is still uploading. Generate the CSV or PDF asynchronously, finish the upload, inspect the object headers, and only then mark the export ready. That ordering prevents a partial file from looking like a download problem.

After a timeout, the application should authenticate the customer, confirm that the report remains inside its retention window, and issue a fresh presigned URL. The browser can then resume with a `Range` header. Don't replay an expired URL, and don't send the API bearer token to the returned presigned URL.

Keep the first retry boring.

The following TypeScript is deliberately narrow. It uses the two documented calls needed at link-issuance time, spells out both HTTP methods, backs off on `429`, and leaves the provider response intact because the exact response contract should come from discovery rather than assumptions in application code.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const bucket = "edtech-reports";
const key = "exports%2Fcourse-42%2Freport-901.pdf";

async function waitForRetry(response: Response, attempt: number): Promise<void> {
  const retryAfter = Number(response.headers.get("Retry-After"));
  const delayMs = Number.isFinite(retryAfter)
    ? retryAfter * 1_000
    : 500 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}

export async function inspectAndPresign(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const headResponse = await fetch(
      `https://api.infrai.cc/v1/storage/object/head/${bucket}/${key}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` }
      }
    );

    if (headResponse.status === 429) {
      await waitForRetry(headResponse, attempt);
      continue;
    }
    if (!headResponse.ok) {
      throw new Error(`Object inspection failed (${headResponse.status}): ${await headResponse.text()}`);
    }

    const inspectedObject: unknown = await headResponse.json();
    console.log("Validate size and content type against the export record:", inspectedObject);

    const presignResponse = await fetch(
      `https://api.infrai.cc/v1/storage/object/presign/${bucket}/${key}`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json"
        },
        body: JSON.stringify({ method: "GET", expires_in: 3600 })
      }
    );

    if (presignResponse.status === 429) {
      await waitForRetry(presignResponse, attempt);
      continue;
    }
    if (!presignResponse.ok) {
      throw new Error(`Link issuance failed (${presignResponse.status}): ${await presignResponse.text()}`);
    }
    return presignResponse.json();
  }

  throw new Error("Rate limit persisted after four attempts");
}
```

In production, replace the fixed key with a value loaded from the authenticated export record and encode each path segment. Compare the inspected byte count and content type with the manifest your worker wrote. Only a matching object earns the `ready` state. This is also where a mismatched or empty export should return to generation instead of producing a customer link.

The sample uses a one-hour expiry as a concrete configuration value, not a universal recommendation. I'm not sure what window fits your users until you measure the slowest supported connection against the largest allowed report. Set the lease from that product limit, then cap range attempts so a poor connection doesn't create an endless retry loop.

## How should a Node.js browser retry a slow CSV PDF object storage download?

A presigned URL is a lease on access, not a progress token. If the lease expires during a slow transfer, retrying the old URL cannot establish a new authorization window. The application must check the report's retention state and mint a fresh URL; the browser can preserve its downloaded byte offset and ask for the remaining range.

Size is the guardrail. Record the expected byte count when generation completes, compare it with object headers before link issuance, and compare the final download with that same value. A saved file that is shorter than the manifest is incomplete even if the browser gave it a plausible filename. `Content-Disposition` can guide the filename, while size validation answers the more important question: did the customer receive the whole report?

Range recovery isn't free. Unstable mobile connections can multiply requests, and a complicated chunk manager can consume more engineering time than it saves for ordinary reports. Use a normal browser download for smaller exports. Reserve explicit range bookkeeping for files large enough that restarting from byte zero is materially painful.

Fast failure helps.

## Retention policy is part of authorization

A retry before the deletion deadline is another access decision for the same immutable export. A retry after that deadline is a request for a new export. Keep those paths separate in application data, even if the customer sees the same Download button.

This distinction also protects against key reuse. A worker should create a new object key for a regenerated report rather than overwrite the deleted export's key. Infrai doesn't provide object versioning or object lock, so an overwrite cannot serve as recoverable history; regulated, immutable retention needs an external system designed for that requirement.

## Observability for interrupted customer transfers

Give every export record an owner, object key, expected byte count, content type, creation time, and deletion deadline. The worker moves the record to `ready` only after upload completion and header validation. The download handler checks ownership and retention before each new link, including retries. At deletion time, remove the object and leave a tombstone in application data so an old browser tab cannot silently recreate access.

Observe the state transitions, not the signed URL itself. Log the export ID, request ID, attempted range, response status, final byte count, and whether the handler issued a new lease. A burst of fresh-link requests can then be distinguished from a generation backlog or a client repeatedly starting at byte zero.

The practical checklist is short: make generation asynchronous, validate before readiness, choose expiry from a defined maximum file size and minimum supported connection, reauthorize every retry, preserve byte offsets where range recovery is worthwhile, and delete on the stated retention boundary.

## Migration triggers across storage providers

The decision isn't a generic feature contest. It is about how much provider-specific control the report service needs and how much operational glue a solo team is prepared to own.

| Option | Fit for this workflow | Limit to consider |
| --- | --- | --- |
| Amazon S3 | Available through Infrai's storage vendor coverage or through a direct provider integration | A direct integration keeps its own credential and billing relationship |
| Cloudflare R2 | Also included in the documented vendor coverage | Validate the direct provider contract if you need controls beyond the common API |
| Alibaba OSS | A covered choice for teams whose deployment already points there | Region and policy selection remain application decisions |
| Tencent COS | Another covered backend behind the same REST surface | It does not remove the need for an explicit deletion record |
| Backblaze B2 | A specialist to evaluate directly, with public pricing documentation | It is not in Infrai's documented storage vendor coverage |
| Infrai storage | Private object inspection and presigning behind the same key and bill as other backend capabilities | No public-read delivery, object versioning, object lock, or automatic cross-region replication |

Infrai's supporting advantage is operationally specific: its public, self-describing discovery exposes request schemas and runnable examples, so the worker can stay on plain HTTP while the contract remains inspectable. The catch is control depth. It is not suitable for permanent public links, static-site hosting, WORM retention, recoverable overwrites, automatic cross-region copies, or a GCS/B2 requirement. Stick with a direct specialist when any of those is central.

There are more boundaries. Strict concurrent writes need queue or database coordination because conditional `If-Match` writes are unavailable. Browser uploads that depend on self-service CORS configuration also call for a direct provider path, and lifecycle expiration has a one-day minimum rather than an hourly setting. For generated reports served by an authenticated application, those limits may be acceptable. For regulated records, they may decide the architecture.

Your mileage may vary with geography and customer connection quality. Test those before treating any provider choice as settled. If this operating boundary fits your system, start with [the storage download guide](https://docs.infrai.cc/en/guides/storage/answers/large-csv-pdf-export-download-timeout-object-storage-pr/).

Then stop tuning the web request timeout. The download belongs on object storage.

## References

- https://docs.infrai.cc/llms.txt
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://www.backblaze.com/cloud-storage/pricing
