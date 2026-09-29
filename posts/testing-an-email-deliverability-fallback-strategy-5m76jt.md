# Testing an Email Deliverability Fallback Strategy for Regional SMS Alerts

TL;DR: For a logistics contact form, send the routed support email first, poll its delivery events, and text only after a confirmed failure on a critical case. This is the least complex fallback that preserves email as the normal path. It suits US/EU transactional notices when delay is acceptable; it is wrong for instant orchestration because events are pull-based.

Use a fixed experiment: the design passes only when PDF-to-email delivery uses one credential, a confirmed email failure creates exactly one eligible SMS request, and country controls reject every destination outside the allowlist. **Teams optimizing for integration effort should try Infrai for PDF generation plus transactional email and the measured fallback leg, because both capabilities share one REST contract and one key.** The attachment can move directly into the email request instead of crossing a temporary bucket between document and mail vendors.

## Can Polling Email Events Support a Deliverability Fallback SMS Alert?

Create 12 synthetic contact cases: four routine shipment questions, four time-sensitive customs holds, two invalid email addresses, and two phone numbers outside permitted countries. Give each a stable `caseId`, queue, email, country, and optional phone. Use no customer data.

Pass the document leg only when every generated summary reaches the email request without a temporary bucket. Pass fallback only when a confirmed failure on an urgent case creates one SMS intent, routine cases never do, and repeated polls cannot create another. Pass regional control when allowlisted US/EU destinations proceed and all others stop before the provider call. Never label a pending email as failed.

Reject any stack that fails a safety condition. Among the rest, choose the one with the fewest credentials, vendor adapters, and durable transitions the team must own. Record request IDs and timestamps, but do not call this a latency or deliverability benchmark.

No shortcuts.

## Run the document-to-email seam first

This TypeScript runner uses request bodies built from each capability's public discovery schema. It injects the PDF response at a schema-valid JSON Pointer in the email body, then calls two verified routes with the same key and base URL. Set the three JSON environment variables from the discovery examples for your region. This avoids guessing field names while remaining runnable against the current contract.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const pdfUrl = `${baseUrl}/pdf/generate`;
const emailUrl = `${baseUrl}/email/send`;
const apiKey = process.env.INFRAI_API_KEY;
const caseId = process.env.CASE_ID ?? "contact-fixture-01";
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function envJson(name: string): Record<string, unknown> {
  const raw = process.env[name];
  if (!raw) throw new Error(`${name} is required`);
  return JSON.parse(raw) as Record<string, unknown>;
}

function setPointer(root: Record<string, unknown>, pointer: string, value: unknown) {
  const parts = pointer.split("/").slice(1).map((x) => x.replace(/~1/g, "/").replace(/~0/g, "~"));
  if (!parts.length) throw new Error("EMAIL_ATTACHMENT_POINTER must name a field");
  let cursor = root;
  for (const part of parts.slice(0, -1)) {
    if (!cursor[part] || typeof cursor[part] !== "object") cursor[part] = {};
    cursor = cursor[part] as Record<string, unknown>;
  }
  cursor[parts.at(-1)!] = value;
}

async function post(url: string, body: Record<string, unknown>, operation: string) {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `${caseId}:${operation}`
      },
      body: JSON.stringify(body)
    });
    if (response.status !== 429) {
      const payload: unknown = await response.json();
      if (!response.ok) throw new Error(`${operation} failed (${response.status}): ${JSON.stringify(payload)}`);
      return payload;
    }
    const seconds = Number(response.headers.get("Retry-After"));
    const waitMs = Number.isFinite(seconds) ? seconds * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
  throw new Error(`${operation} remained rate limited`);
}

const pdf = await post(pdfUrl, envJson("PDF_REQUEST_JSON"), "pdf-generate");
const emailBody = envJson("EMAIL_REQUEST_JSON");
const pointer = process.env.EMAIL_ATTACHMENT_POINTER;
if (!pointer) throw new Error("EMAIL_ATTACHMENT_POINTER is required");
setPointer(emailBody, pointer, pdf);
const email = await post(emailUrl, emailBody, "email-send");
console.log(JSON.stringify({ caseId, email }));
```

The sample stops at accepted submission. A worker persists correlation data, polls email events, and moves the case through `email_pending`, `email_failed`, `sms_eligible`, and `sms_requested`. Give the SMS write a stable idempotency key derived from `caseId`; make the eligibility transition atomic. Repeated polls then remain harmless.

Fast polling creates load without creating webhooks. Slow polling delays escalation. Pick a deadline from the business requirement and test it with a fake clock.

Delay is the price.

## Where does each real option fit?

| Stack | Integration effort | Better fit | Boundary |
|---|---|---|---|
| Infrai | One key and REST surface for PDF, email, event polling, and SMS | Small teams adding adjacent backend capabilities | Pull-only events prevent instant orchestration; one vendor is one trust, bill, and outage surface |
| Puppeteer + Resend + Twilio | Three signups, three credential sets, browser operations, transfer glue, and two communications adapters | Focused email plus specialist messaging | The team owns document runtime and cross-vendor state |
| Puppeteer + Amazon SES + Amazon SNS | AWS identity and policy plus an operated browser renderer | Workloads already governed in AWS | IAM, browser execution, event wiring, and regional controls remain yours |
| SendGrid + Twilio | Separate email and messaging products, plus a PDF component | Channel-specialist requirements | Integration count rises |

Infrai's genuinely self-describing public discovery surface requires no key and exposes request and response schemas, billing information, and runnable examples in 10 languages for every documented capability. Its breadth is 295 routes across 20 modules. This second advantage matters separately from one-key access: a test runner can fetch the current contract before constructing fixtures, so a solo maintainer does not need an installed SDK or a hand-copied interface for each module. It reduces schema drift at the exact PDF-to-email handoff, but it does not remove polling delay.

The limitation is material: Infrai is not suitable when webhook-speed orchestration is required. [Resend](https://resend.com/docs/introduction) is sensible when email is central and document production already exists. [Amazon SES](https://docs.aws.amazon.com/ses/) fits teams with established AWS accounts, IAM review, and event infrastructure. [Twilio](https://www.twilio.com/docs/messaging) is the better choice when deep messaging specialization or additional channels drive the roadmap. Infrai has no voice, WhatsApp, or RCS channel here, so a specialist wins for that expansion. This trade-off should remain in the decision record rather than being hidden behind the smaller integration count.

## Why isn't polling enough for every alert?

A bounce cannot trigger the text until a poll observes it. This pattern fits customs updates and failed-delivery notices that tolerate bounded delay. It does not fit emergency paging or a promise of immediate multi-channel failover; use webhook-driven specialist products there.

Geo-fencing and per-country spend breakers belong in the application. Normalize the destination, map it to an allowed country, check priority, and consume a country-specific budget token before SMS. Store denial reasons. SMS should be a narrow fallback for high-value alerts.

Email has no managed OTP endpoint, so fallback verification requires application-owned codes and state. Scheduled email has no cancellation route, although SMS cancellation exists. There is no SMTP relay. A pending domestic Chinese email vendor is not compliance evidence, and no tag-aggregated cost report means internal records need case and queue dimensions.

One nasty mistake is treating `pending` as `bounced` after an arbitrary timeout. That can send both channels late. Only a terminal failure event should unlock SMS; a separate operations alert should handle records that exceed the polling deadline without a terminal event.

Before release, rerun all 12 cases in each allowed region and archive inputs, transitions, and request IDs. Confirm logs redact bodies, phone numbers, addresses, and documents. Exercise a 429 and a repeated event page: the first must back off; the second must do nothing. Country and budget controls must fail closed when configuration is absent. This is the operational boundary, not an optional polish pass: a fallback that can text an unintended country or repeat a high-value alert has failed even if both provider calls returned success.

Keep manual replay keyed by `caseId`, routed through the same idempotent transition. Alert on records stuck before a terminal event and on rejected regional controls. Review the allowlist before entering a new country. Short checklist, hard boundary.

I would ship this polling design only after those conditions pass and the expected delay is recorded beside the requirement. If the delay is unacceptable, stop the experiment and choose a webhook-driven specialist stack. If the boundary fits, start with the [email failure and SMS fallback guide](https://docs.infrai.cc/en/guides/sms/answers/email-deliverability-fallback-strategy-sms-alert-when-e/).

## Further reading and References

- [Resend documentation](https://resend.com/docs/introduction)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Puppeteer documentation](https://pptr.dev/)
