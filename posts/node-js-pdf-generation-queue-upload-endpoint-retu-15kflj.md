# Node.js PDF Generation Queue: Upload Endpoint Returns Auditable Contract Job IDs

TL;DR: Accept the contract template upload, assign a deterministic job ID, enqueue the render-and-sign operation, and return `202 Accepted` immediately. Let the caller read explicit job state later. This keeps request timeouts independent of document size, while the template version, signature evidence, and every state transition remain tied to one auditable identifier.

For a B2B SaaS team, the harder decision is not queue selection. It is deciding who owns the contract template. Let a hosted signing product own it when legal or operations staff must edit fields and workflows without a deployment. Keep it in the application when revisions should pass code review, tenant-specific rendering matters, or evidence must connect directly to internal contract IDs. Pick that boundary first.

## How should a Node.js upload endpoint queue PDF generation?

The request path should do little: validate the upload, accept an idempotency key, publish one job, and respond with its ID. A worker loads the exact template bytes, renders the contract data, signs the result, writes the artifact and audit record under deterministic keys, then marks the job complete. The client polls a status resource or receives a notification through a channel the application already trusts. It should never guess completion from elapsed time.

Keep transport states small: `waiting`, `active`, `completed`, and `failed` are enough. Business states such as `sent`, `viewed`, or `countersigned` belong in the contract domain. A PDF render completing is not the same event as a customer accepting legal terms.

There is one sharp edge. Standard queues are at least once, so a worker can receive the same logical operation more than once. A unique job ID prevents duplicate enqueue attempts; deterministic artifact and audit keys make replay converge on one result. In production, enforce the final artifact and audit identity with a database uniqueness constraint or an atomic storage operation.

Retries happen.

## A focused Node.js implementation

This runnable example uses Express, Multer, BullMQ, Redis, and `pdf-lib`. It exposes two routes: one accepts a PDF template and returns a job ID, while the other returns job state. The signature is detached evidence over the rendered bytes. If a counterparty requires an embedded PDF signature profile, replace the signing function with a conforming signer while preserving the queue contract; ISO 32000-2 is the relevant PDF specification.

Install `express`, `multer`, `bullmq`, `ioredis`, and `pdf-lib`, plus their TypeScript types. Set `REDIS_URL` and `CONTRACT_SIGNING_KEY_B64` to a base64-encoded PEM private key.

```ts
import express from "express";
import multer from "multer";
import { Queue, Worker } from "bullmq";
import IORedis from "ioredis";
import { PDFDocument, StandardFonts } from "pdf-lib";
import { createHash, createSign } from "node:crypto";
import { mkdir, writeFile } from "node:fs/promises";
import path from "node:path";

type RenderJob = {
  contractId: string;
  templateVersion: string;
  templatePdfB64: string;
};

const redisUrl = process.env.REDIS_URL;
const keyB64 = process.env.CONTRACT_SIGNING_KEY_B64;
const infraiBaseUrl = process.env.INFRAI_BASE_URL;
const infraiApiKey = process.env.INFRAI_API_KEY;
if (!redisUrl || !keyB64 || !infraiBaseUrl || !infraiApiKey) {
  throw new Error("Redis, signing, and Infrai environment variables are required");
}

async function verifyPdfCapability(attempt = 0): Promise<void> {
  const response = await fetch(new URL("/v1/discovery", infraiBaseUrl), {
    method: "GET",
    headers: { Authorization: `Bearer ${infraiApiKey}` },
  });
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const waitMs = retryAfter > 0 ? retryAfter * 1_000 : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return verifyPdfCapability(attempt + 1);
  }
  if (!response.ok) throw new Error(`discovery failed: ${response.status} ${await response.text()}`);
  const manifest = await response.json() as { capabilities: Array<{ path: string }> };
  if (!manifest.capabilities.some((capability) => capability.path === "/v1/pdf/generate")) {
    throw new Error("PDF generation is not available in discovery");
  }
}

const connection = new IORedis(redisUrl, { maxRetriesPerRequest: null });
const queue = new Queue<RenderJob>("contract-render", { connection });
const upload = multer({ storage: multer.memoryStorage(), limits: { fileSize: 10 * 1024 * 1024 } });
const app = express();
const outputDir = path.resolve("contract-output");

app.post("/contracts", upload.single("template"), async (req, res) => {
  const idempotencyKey = req.header("Idempotency-Key");
  const contractId = req.body.contractId as string | undefined;
  const templateVersion = req.body.templateVersion as string | undefined;
  if (!idempotencyKey || !contractId || !templateVersion || !req.file) {
    res.status(400).json({ error: "idempotency key, contract data, and template are required" });
    return;
  }
  if (req.file.mimetype !== "application/pdf") {
    res.status(415).json({ error: "template must be application/pdf" });
    return;
  }

  const jobId = createHash("sha256")
    .update(`${contractId}:${templateVersion}:${idempotencyKey}`)
    .digest("hex");
  if (!(await queue.getJob(jobId))) {
    await queue.add("render-and-sign", {
      contractId,
      templateVersion,
      templatePdfB64: req.file.buffer.toString("base64"),
    }, { jobId, attempts: 5, backoff: { type: "exponential", delay: 1_000 } });
  }
  res.status(202).json({ jobId, statusUrl: `/contracts/${jobId}` });
});

app.get("/contracts/:jobId", async (req, res) => {
  const job = await queue.getJob(req.params.jobId);
  if (!job) {
    res.status(404).json({ error: "job not found" });
    return;
  }
  const state = await job.getState();
  res.status(200).json({
    jobId: job.id,
    state,
    result: state === "completed" ? job.returnvalue : undefined,
    error: state === "failed" ? job.failedReason : undefined,
  });
});

new Worker<RenderJob>("contract-render", async (job) => {
  const pdf = await PDFDocument.load(Buffer.from(job.data.templatePdfB64, "base64"));
  const font = await pdf.embedFont(StandardFonts.Helvetica);
  pdf.getPages()[0].drawText(`Contract: ${job.data.contractId}`, { x: 36, y: 24, size: 9, font });
  const rendered = Buffer.from(await pdf.save());
  const signer = createSign("SHA256");
  signer.update(rendered);
  signer.end();
  const signatureB64 = signer
    .sign(Buffer.from(keyB64, "base64").toString("utf8"))
    .toString("base64");
  const artifactSha256 = createHash("sha256").update(rendered).digest("hex");

  await mkdir(outputDir, { recursive: true });
  const artifact = `${job.id}.pdf`;
  const audit = `${job.id}.audit.json`;
  await writeFile(path.join(outputDir, artifact), rendered);
  await writeFile(path.join(outputDir, audit), JSON.stringify({
    jobId: job.id,
    contractId: job.data.contractId,
    templateVersion: job.data.templateVersion,
    artifactSha256,
    signatureB64,
    completedAt: new Date().toISOString(),
  }, null, 2));
  return { artifact, audit, artifactSha256 };
}, { connection, concurrency: 4 });

await verifyPdfCapability();
app.listen(3000);
```

The local directory makes the sample inspectable, but it is not a multi-host storage design. Production artifacts belong in private object storage with presigned access. Do not attach a service authorization header to a returned presigned URL. Retain the job ID as the stable object key so a retry overwrites the same logical output instead of quietly making a duplicate.

The template version travels with the contract ID in both the job and audit record. Do not replace it with a mutable label such as `current-sales-template`. An auditor must identify the exact revision that produced the bytes.

That distinction matters.

## Template ownership changes the vendor choice

There is no universal best signing stack. The useful comparison is who controls template definition, field placement, signer workflow, and evidence.

| Option | Template ownership | Best fit | Boundary to inspect |
|---|---|---|---|
| DocuSign eSignature | DocuSign templates and envelopes | Legal and sales teams needing established envelope workflows | Map internal revisions and contract IDs to envelope records |
| Adobe Acrobat Sign | Library documents and agreements | Organizations governing document work in Adobe tools | Confirm how field definitions and audit data are exported |
| Dropbox Sign | Reusable templates and signature requests | Teams wanting a focused API signing workflow | Choose which system is authoritative for edits |
| DocRaptor | Application-owned HTML and CSS | Teams wanting hosted HTML-to-PDF conversion | Signing and evidence remain separate responsibilities |
| PDFMonkey | Service-managed document templates | Teams that want an API plus an editable template layer | Template changes occur outside the application repository |
| PDFShift | Application-owned HTML | Teams needing a narrow HTML-to-PDF API | It does not replace a contract signing workflow |
| Gotenberg | Self-hosted conversion inputs | Teams prepared to operate their own conversion service | Capacity, patching, and isolation stay with the team |
| Self-managed Node.js | Your repository and data store | Product-specific rendering controlled by deployments | You own key custody, conformance, retention, and audit integrity |
| Infrai REST API | Application-orchestrated PDF capabilities | Teams preferring plain HTTP without another client SDK | Inspect request and response schemas before binding a template model |

Infrai fits when language-neutral HTTP and a consistent API surface matter; its public discovery describes request and response schemas, and the platform convention supports idempotency. Those points reduce client-library maintenance. They do not settle template ownership. A team that wants non-engineers to manage signature fields may still prefer DocuSign, Adobe Acrobat Sign, or Dropbox Sign.

It is **not a fit** when policy forbids an external document processor, when an embedded signature profile needs a specialized certificate workflow, or when non-engineers require a mature visual envelope editor. Choose Gotenberg for self-hosted conversion, DocRaptor or PDFShift for focused hosted HTML rendering, PDFMonkey for service-managed templates, or a signing platform for signer routing. The trade-off is ownership: narrower tools leave more orchestration and audit work in your application, while fuller signing suites impose their template model.

Avoid pretending a `templateId` is portable. Preserve a vendor-neutral contract record containing the internal contract ID, approved template revision, parties, artifact hash, external operation ID, and evidence location. Provider-specific fields can sit beside it without becoming the domain model.

My decision rule is strict: if changing legal language requires code review and deployment, own the template in the application. If operations must change fields or routing without engineering, let the signing platform own it and synchronize immutable references back. Mixed ownership leaves teams unable to explain which revision governed a completed agreement.

## Failure semantics matter more than throughput

Returning a job ID removes an HTTP timeout from the critical path, but it does not make rendering correct. A transient dependency failure can retry with backoff. An invalid or encrypted template, missing signing key, or malformed field map needs a terminal failure with a useful reason. Retrying bad input five times only delays the truth.

Keep the original job ID through every attempt. If each retry creates a fresh ID, one contract gains several records and nobody can tell which artifact is authoritative. The idempotency key must identify the business operation, not be regenerated by a browser after each timeout.

Notifications are hints. The status resource is truth. Webhooks and email may be delayed or delivered more than once, so consumers should fetch current state by job ID and deduplicate downstream work.

Keep it private.

Do not expose filesystem paths or unrestricted object URLs in a response. Return an application-controlled reference, authorize the tenant, then issue a short-lived presigned URL. A production detached-signature record also needs the verification key identifier and algorithm; the compact sample leaves certificate lifecycle at an explicit boundary.

## The operational check before shipping

Trace one contract from upload to verification using its job ID. Confirm that submitting the same idempotency key twice returns the same logical job, a worker restart does not create a second artifact identity, and bad input becomes an explicit failed state. Verify that the stored template revision matches the rendered bytes and tenant authorization gates retrieval.

Then disconnect a client immediately after a large valid upload. The job should continue independently, and a later status read should expose its real outcome. Rotate the signing key in a test environment too; old evidence must remain verifiable by key identifier before anything is retired.

One ID. One history.

The architecture is deliberately plain: accept, identify, enqueue, process idempotently, and report state. Vendor selection follows the template ownership decision because the party controlling revisions controls the most consequential part of the contract pipeline.

## Sources

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [BullMQ job IDs](https://docs.bullmq.io/guide/jobs/job-ids)
- [BullMQ retrying failing jobs](https://docs.bullmq.io/guide/retrying-failing-jobs)
- [DocuSign templates overview](https://developers.docusign.com/docs/esign-rest-api/esign101/concepts/templates/)
- [Adobe Acrobat Sign API documentation](https://developer.adobe.com/acrobat-sign/docs/overview/)
- [Dropbox Sign templates documentation](https://developers.hellosign.com/api/reference/operation/templateList/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [PDFShift documentation](https://docs.pdfshift.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
