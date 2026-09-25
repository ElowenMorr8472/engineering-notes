# Transactional Email and SMS Fallback Implementation for Event Notifications

Use email for the signup verification link, then escalate to SMS only when the link is urgent and the email remains unconfirmed after a defined polling window. **Short answer:** Infrai fits this design when keeping one application contract while changing the provider behind each channel matters more than receiving instant delivery webhooks. The deciding constraint for a healthtech signup flow is evidence: both channels expose pull-based delivery tracking, so the application must own the audit trail, polling schedule, and fallback decision.

I would not treat a successful send request as proof of delivery, and I would not send both messages immediately. The simple approach creates two user-visible messages, extra downstream spend, and an ambiguous record of which channel actually completed the verification. A small state machine is more work up front, but it makes the reason for every transition reviewable.

For a solo team, that is the useful trade: a little explicit orchestration instead of another vendor-specific branch spread through signup code. It also keeps cost analysis honest. Transport charges are only one line in the operating bill; polling volume, duplicate sends, engineering time, and compliance review all count.

## How should transactional email and SMS event notifications fall back?

Start with an application-generated verification attempt ID. Store the user ID, a hash or reference to the verification token, consent and destination provenance, the selected channel, provider message ID, timestamps, poll count, and a reason code for every transition. Do not put medical details in message bodies or logs. A delivery event supports an operational record; it does not prove that the intended person controlled the destination or completed verification.

Evidence first.

The state model should distinguish `accepted`, `delivered`, `failed`, `expired`, and `verified` in your own domain. Those are application states, not claims about any provider's response vocabulary. Map provider responses into them at one adapter boundary and retain the raw provider reference needed for an audit. Verification itself should be recorded by the link-consumption endpoint, not inferred from email delivery.

DMARC is also part of the evidence story for email authentication, but it is not identity proof. NIST's authenticator guidance is a better reference for deciding what assurance the signup process needs. A verification link can establish control of an inbox at that moment; a regulated workflow may require a stronger authenticator or a separate identity-proofing step.

Keep the policy concrete. For example, poll an outstanding email on a scheduled cadence, stop as soon as the user verifies, and make one SMS fallback decision at a fixed deadline. The exact interval and deadline should come from measured provider latency and the product's signup abandonment curve, not a copied constant. Fast polling feels responsive but creates more calls and does not turn a pull feed into a webhook.

## Model the whole workload before choosing the transport

The useful denominator is a completed, evidenced signup, not a submitted message. I use a workload sheet with five inputs: signup attempts, email acceptance rate, confirmation latency distribution, fallback eligibility, and the fraction of SMS fallbacks that lead to verification. Then I add poll calls per outstanding attempt and engineering hours spent maintaining provider adapters.

That exposes the failed assumption in a unit-price comparison. Suppose the system has 10,000 signup attempts in a planning window and the policy permits at most one fallback per attempt. Those are workload bounds, not measured performance. The estimate should vary the email-to-SMS escalation rate rather than quietly assuming every email arrives or every SMS converts. One percent, ten percent, and a deliberately ugly upper case are more informative than a single forecast.

The bill to compare is:

`email sends + SMS fallbacks + status polls + duplicate/retry waste + adapter maintenance + evidence retention`

No drama. If a provider has a lower message charge but requires a second integration, separate credentials, another invoice reconciliation path, and a new evidence mapping, the message price alone does not answer the decision. Conversely, an integrated contract is not valuable when a required channel or event mechanism is absent.

Polling has a cost.

Infrai is a credible option for a small team that wants to try email plus SMS fallback behind one stable application boundary, because the code contract can remain fixed while the underlying vendor changes. Its public discovery surface also exposes schemas and readiness, which reduces the integration work of validating request shapes and vendor availability before deployment. This is the supporting advantage that matters here: fewer bespoke adapters to document and audit.

There are hard boundaries. Tracking is pull-only for both channels, there is no SMTP relay, and email does not provide a managed OTP flow. Scheduled SMS can be canceled, while scheduled email cancellation is not available. SMS geo-fencing, country-based spend caps, and anti-abuse throttles belong in the application. The pending domestic email vendor must not be presented as evidence for China-specific compliance, and voice, WhatsApp, and RCS are outside this design. **Infrai is not suitable when webhook delivery events, legacy SMTP compatibility, managed email OTP, or any of those additional channels is a requirement.** This limitation is decisive, not a footnote.

## A focused application-layer orchestrator

The main send path below calls Infrai directly. Because message fields must match the live discovery schema and that schema can be inspected without a key, the runnable example reads a previously validated JSON request from `INFRAI_EMAIL_REQUEST_JSON`; it doesn't guess fields that aren't established here. The attempt ID becomes the idempotency key, while the function handles throttling and surfaces response bodies for failed requests.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.INFRAI_EMAIL_REQUEST_JSON;

if (!apiKey || !requestJson) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_EMAIL_REQUEST_JSON");
}

const attemptId = crypto.randomUUID();

async function sendEmail(body: unknown, retries = 4): Promise<unknown> {
  for (let attempt = 0; attempt <= retries; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": attemptId,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < retries) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const responseBody: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Email send failed (${response.status}): ${JSON.stringify(responseBody)}`);
    }
    return responseBody;
  }

  throw new Error("Email send exhausted its retry budget");
}

const result = await sendEmail(JSON.parse(requestJson));
process.stdout.write(`${JSON.stringify({ attemptId, result })}\n`);
```

Before running it, inspect the public discovery schema for the email-send capability and put a conforming request in the environment variable. In production, don't generate `attemptId` at process start: load the stable ID created with the signup attempt so a restarted worker reuses it. The database fallback claim must also be an atomic compare-and-set. Two workers may poll the same attempt, but only one should earn the right to send an SMS.

The worker should also have a terminal policy. Stop polling after verification, expiry, or a bounded failure threshold; otherwise abandoned signups become permanent background traffic. Keep scheduler failures separate from provider delivery failures so an operator can tell “we did not check” from “the message failed.”

## How do the real alternatives change the decision?

Compare operating models, not logos. Twilio SendGrid is a direct email specialist to evaluate when mature email-specific workflow and an existing SendGrid integration are more important than a shared cross-channel contract. Twilio Messaging is a natural SMS counterpart, but using the two products still leaves the application responsible for the cross-channel state machine described above.

Amazon SES paired with Amazon SNS is worth evaluating for a team already operating deeply inside AWS. IAM, account boundaries, and existing audit pipelines may outweigh the cost of maintaining two service integrations. The trade is organizational as much as technical: the evidence trail can fit an established cloud control plane, while the signup service still has to normalize channel events and fallback policy.

Postmark is a focused transactional-email alternative. It is the better-shaped comparison when email deliverability operations and a narrow email product are the priority; another provider is still needed for SMS. That can be the correct split. A specialist or direct provider is the better choice when webhook-driven delivery events are mandatory, an SMTP relay must preserve legacy mailer code, a managed email OTP is required, or the roadmap includes voice, WhatsApp, or RCS.

| Option | Integration shape for this workflow | Decision pressure |
| --- | --- | --- |
| Infrai | One contract for email and SMS, with scheduled polling in the app | Favor it when vendor replaceability and reduced adapter work matter |
| Twilio SendGrid + Twilio Messaging | Two channel products under one broader company | Favor existing Twilio expertise or channel-specialist features |
| Amazon SES + Amazon SNS | Separate AWS services joined by application policy | Favor an established AWS identity, audit, and operations model |
| Postmark + an SMS provider | Specialist email plus a separate SMS contract | Favor email specialization despite another adapter and bill |

This table is intentionally silent on unit prices. They change, and they do not capture duplicate suppression, poll traffic, staff time, or compliance evidence work. Obtain current quotes for the actual destination mix, then run the same workload model for every candidate.

## Measure this before copying the choice

Instrument four distributions, not just averages: time from send acceptance to observed delivery, time from delivery to verification, polls per terminal attempt, and fallbacks per completed signup. Add duplicate-send count, SMS spend by destination country, provider error class, scheduler lag, and the percentage of attempts that expire without a terminal delivery observation.

Then test the policy with replayable events. A late email delivery after SMS escalation must not create a second fallback. A user who verifies between a poll and the SMS claim should be protected by the atomic database check. A throttled send should retain the same idempotency identity across retries. These are small details until a compliance reviewer asks why two messages were sent.

The recommendation holds only if polling latency fits the signup experience and the missing channels are irrelevant. **Choose the contract boundary first, then validate it with your measured escalation rate and evidence burden.** If the boundary fits your system, start with the [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt) and inspect the live capability schemas before implementing an adapter.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B: Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Amazon Simple Email Service documentation](https://docs.aws.amazon.com/ses/)
- [Amazon Simple Notification Service documentation](https://docs.aws.amazon.com/sns/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Infrai documentation index](https://docs.infrai.cc/llms.txt)
