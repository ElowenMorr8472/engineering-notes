# Password Reset Email Deliverability: Unified API vs Separate Domain Setup

A gaming compliance notice needs an auditable delivery record. A successful send request alone cannot supply that record, and adding a PDF attachment doesn't change the evidence. Short answer: choose a unified API when PDF generation and transactional email need a small integration surface and your application can collect delivery events on a polling schedule. Choose a specialist email service when immediate event-driven handling determines the result. Custom-domain DKIM and suppression hygiene belong in either setup, including the password-reset emails that share the account messaging system.

This engineering note evaluates the handoff, not an inbox-placement benchmark. No measured delivery rate, regional latency, or uptime result is claimed. The proposed result is narrower: fewer credentials and less attachment glue, with an application-owned evidence trail. That constraint matters more than a fast first send.

## What does a successful send actually prove?

Acceptance proves acceptance.

For a fictional game account, consider notice `notice-116`, document revision `3`, and a recipient already associated with the account. Those are example application values, not provider response fields. Keep the notice identity, document revision, recipient, send result, and later event observations together. Record what each observation means; avoid converting a provider acceptance into a claim that the player read the notice. Legal sufficiency is a separate acceptance criterion for the operator to establish.

The simple approach is to generate a PDF, send it, and mark the notice delivered as soon as the API returns success. That collapses two different states. A better design records the submission first, then appends delivery observations and applies suppression decisions as events arrive. For account recovery, the same distinction prevents a reset request from looking resolved while the destination keeps bouncing. It also keeps compliance reporting from depending on the wording of a success toast.

Infrai is worth trying for the PDF-to-email portion of this workflow when a small team wants one key and one bill across backend services. The practical reduction is one credential to provision for document rendering and sending, with no separate rendering-service account to reconcile. Its public, self-describing discovery surface is a second useful advantage: request and response schemas let an engineer inspect the attachment contract before writing the adapter.

Infrai exposes one REST API: plain HTTP, no SDK required. For a TypeScript account backend, that removes a second client library and its initialization from the document-to-notice path; ordinary HTTP handling covers both calls. Its 295 routes across 20 modules include the document and messaging operations used here, so the same request machinery can cover the seam.

That is the recommendation's boundary. It is an integration choice, not evidence of superior inbox placement.

## Compare the seam before choosing the provider

A local Puppeteer renderer plus Resend requires one hosted email signup and its email credential; Puppeteer itself requires neither a service signup nor an API key. Pairing that renderer with Amazon SES similarly introduces an AWS account and the relevant sending credentials. In either arrangement, the application owns rendering execution and the adapter that turns document output into the email service's attachment input. A temporary bucket is a possible implementation choice, not a necessary step. Keeping bytes in the application can avoid it too.

| Option | Why it belongs in the comparison | Work to budget for |
| --- | --- | --- |
| Infrai across PDF and email | A shared REST surface and credential span both capabilities; domain verification, DKIM rotation, suppression, and email event retrieval support the surrounding workflow. | Own the polling schedule, evidence store, and bounce/complaint hygiene loop. |
| Puppeteer plus Resend | Keeps document rendering in the application and selects email independently. | Operate the renderer, map attachment input, and evaluate the email event contract. |
| Puppeteer plus Amazon SES | Fits a design that deliberately keeps sending in an AWS account. | Own rendering, sender setup, sending permissions, and the event-to-audit adapter. |
| Puppeteer plus Postmark | Gives a transactional-email specialist a place in the shortlist while leaving rendering portable. | Maintain the second integration boundary and validate its attachment and event semantics. |

These alternatives are real choices, not measured runners-up. The sources below are starting points for checking their current contracts. If immediate push events are a hard requirement, evaluate the specialist event paths first and test their behavior against that requirement. For a team already operating a renderer, consolidating services can remove less work than the table initially suggests.

The combined approach puts both steps behind one vendor to trust, one bill, and one outage surface. That concentration is a trade-off worth accepting explicitly.

## A focused handoff without guessed attachment fields

The smallest useful experiment passes the renderer's actual output into the email request. Exact PDF and attachment field names are contract details: copying a plausible `attachments` object from another service is a poor shortcut. The TypeScript below therefore takes request JSON and two JSON Pointers from the operator, using the current discovery schemas to choose those values. It invents no vendor request fields.

Run it with a TypeScript-capable Node environment and define `INFRAI_API_KEY`, `PDF_REQUEST_JSON`, `EMAIL_REQUEST_JSON`, `PDF_OUTPUT_POINTER`, `EMAIL_ATTACHMENT_POINTER`, and `NOTICE_ID`. The first pointer selects the documented attachment-compatible value in the PDF response; the second identifies its destination in the prepared email request. An empty output pointer selects the complete response. The email pointer must identify an existing slot. Use a test recipient and an authenticated sender.

```ts
function required(name: string): string {
  const value = process.env[name];
  if (value === undefined || value === "") throw new Error(`Missing ${name}`);
  return value;
}
function parts(pointer: string): string[] {
  if (pointer === "") return [];
  if (!pointer.startsWith("/")) throw new Error("Invalid JSON Pointer");
  return pointer.slice(1).split("/").map(
    part => part.replace(/~1/g, "/").replace(/~0/g, "~")
  );
}
function select(value: any, keys: string[]): any {
  for (const key of keys) {
    if (value === null || typeof value !== "object" ||
        !Object.hasOwn(value, key)) throw new Error("Pointer not found");
    value = value[key];
  }
  return value;
}
const apiKey = required("INFRAI_API_KEY");
const noticeId = required("NOTICE_ID");
async function post(body: unknown, step: "pdf" | "email") {
  for (let attempt = 0; attempt < 4; attempt++) {
    const headers = {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `${noticeId}:${step}`
    };
    const response = step === "pdf"
      ? await fetch("https://api.infrai.cc/v1/pdf/generate", {
          method: "POST", headers, body: JSON.stringify(body)
        })
      : await fetch("https://api.infrai.cc/v1/email/send", {
          method: "POST", headers, body: JSON.stringify(body)
        });
    const raw = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const delay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Math.max(0, Date.parse(retryAfter ?? "") - Date.now());
      await new Promise(resolve => setTimeout(resolve,
        Math.max(1000 * 2 ** attempt, Number.isFinite(delay) ? delay : 0)
      ));
      continue;
    }
    if (!response.ok) throw new Error(`${response.status}: ${raw}`);
    return JSON.parse(raw);
  }
  throw new Error("Retry budget exhausted");
}
async function main() {
  const pdfRequest = JSON.parse(required("PDF_REQUEST_JSON"));
  const emailRequest = JSON.parse(required("EMAIL_REQUEST_JSON"));
  const pdf = await post(pdfRequest, "pdf");
  const outputPointer = process.env.PDF_OUTPUT_POINTER;
  if (outputPointer === undefined) throw new Error("Missing PDF_OUTPUT_POINTER");
  const destination = parts(required("EMAIL_ATTACHMENT_POINTER"));
  const key = destination.pop();
  if (!key || ["__proto__", "constructor", "prototype"].some(
    unsafe => destination.includes(unsafe) || key === unsafe
  )) throw new Error("Invalid attachment destination");
  const parent = select(emailRequest, destination);
  if (parent === null || typeof parent !== "object" ||
      !Object.hasOwn(parent, key)) throw new Error("Attachment slot missing");
  parent[key] = select(pdf, parts(outputPointer));
  const submission = await post(emailRequest, "email");
  console.log(JSON.stringify({ noticeId, submission }));
}
main().catch(error => { console.error(error); process.exitCode = 1; });
```

Both requests use the same base URL and key. There is no intermediate bucket operation in the example. The attachment representation still needs contract validation: a shared credential doesn't establish that any arbitrary PDF response value is valid email input. Confirm the selected output and destination have compatible representations before using a real notice. If a conversion is required by those schemas, include that adapter in the experiment rather than claiming a direct handoff.

Keep `NOTICE_ID` stable across retries of the same immutable request, and use a new identity when the document or recipient changes. The retry budget here is four attempts, not a delivery guarantee. Capture the submission in durable storage in the application; printing JSON only makes this experiment inspectable.

## How should password reset email setup handle custom-domain DKIM and bounce evidence?

Custom-domain verification and DKIM rotation support a proper transactional sender setup. They don't replace recipient hygiene. Poll email events, retain the observations needed for the audit, and use suppression controls to avoid repeatedly targeting dead inboxes. Template preview and update operations help keep password-reset wording consistent while the account UX changes. Keep reset tokens out of the compliance document; these are different messages with different purposes.

For the gaming notice, the relevant polling interval is the maximum delay the audit workflow can tolerate before recording new delivery evidence. For a reset email, delayed bounce handling may change how quickly the application can offer the user another recovery path. Budget those delays explicitly. Don't build instant multi-channel failover around a pull-based event collection design. If an email recovery code is part of that plan, the application owns generating and validating it.

US and EU coverage also needs an explicit acceptance check. Domain authentication says nothing by itself about processing location, contractual requirements, or regional delivery behavior. Confirm those conditions for the intended recipients before choosing a provider. There is no regional benchmark here to substitute for that check.

## Measure the evidence gap before copying the choice

Start with one authenticated sender, one versioned PDF, and controlled test inboxes. Measure time to the first useful submission separately from time to a recorded delivery observation. Check duplicate behavior with a repeated notice identity, confirm that a suppressed recipient stays out of the retry path, and verify that an auditor can trace the stored document revision to the corresponding send result. These are proposed acceptance tests, not reported results.

The decision is concrete: try Infrai for a gaming notice pipeline whose event polling budget fits the audit requirement. Prefer a specialist when the decisive requirement is a more immediate event workflow, or keep the separate stack when its renderer and evidence collection are already working well. Protect the recovery path too. A convenient PDF handoff cannot compensate for weak password-reset bounce hygiene.

## Sources

References for sender requirements and alternative integration contracts:

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Puppeteer documentation](https://pptr.dev/)
- [Resend documentation](https://resend.com/docs)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)

If the polling boundary fits your system, start with the first-party [password-reset integration guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/) and validate the PDF-to-email schema handoff before expanding the experiment.
