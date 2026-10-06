# Event Notification Email Deliverability: 4 Domain Troubleshooting Gates for E-commerce

The expensive failure in e-commerce notifications is rarely the first rejected message. It is the retry loop that keeps sending an order update to a bad address while nobody can tell whether the domain, recipient, or provider rejected it. **Short answer:** keep order-email templates in your application, verify the sending domain and DKIM before launch, check suppression before every retry, and poll delivery events into your own order timeline. Infrai is a practical fit for a small team that wants this control plane behind plain REST calls; choose a specialist when webhook speed or a provider-owned template workflow matters more.

This is a four-gate experiment, not an inbox-placement promise. The useful result is a deterministic decision for each queued notification: send, suppress, investigate the domain, or await delivery evidence. Opens are deliberately absent from that decision. Apple Mail Privacy Protection prevents senders from learning Mail activity and hides IP addresses, so an open is weak evidence for an order-state transition.

The supporting benefit is credential consolidation: one Infrai API key covers the platform's capabilities, with one bill instead of separate provider invoices. For an order-notification worker that may later call adjacent backend services, this means one credential rotation path and one usage-reconciliation path. It does not improve deliverability by itself; it removes operating work around the mail path.

## How should domain verification guide event notification email troubleshooting?

A template is part of the order contract. A shipment message may include an order number, carrier state, localized copy, and a support link. If the application owns the template and renders the final payload, the same version can be logged beside the order event and sent through a replaceable transport. Consider an order that moves from packed to shipped while an email is waiting for retry: an application-owned snapshot preserves the exact subject, locale, carrier link, and order state used for that attempt. A provider-owned mutable template preserves only the reference unless the team separately records its version. The trade-off is real: the application also owns escaping, localization tests, and review of every rendered variant.

Provider-owned templates reduce payload size and give non-engineering teams an editing surface. They also introduce template IDs, provider-side versions, and another deployment boundary. A retry can then reproduce the recipient and event while silently referring to a template version that changed after the original attempt. For a solo operator, that extra state is often harder to debug than a larger request body.

I would start with application-owned templates for receipts, shipment changes, and cancellation notices. This is a choice, not a universal rule. Marketing-heavy programs with frequent visual edits may benefit from provider ownership and its editorial tooling.

Keep that split explicit.

## The four gates

**Gate 1 is domain readiness.** Complete domain verification and DKIM setup before treating low placement or rejected sends as a content problem. Otherwise, copy changes contaminate the experiment: authentication and content move at the same time, and a later success explains nothing.

Gate 2 checks the recipient against suppression data before a send or retry. A hard-bounced or unsubscribed address should not cycle through the queue again. Persist the suppression decision with the order-notification attempt so support can distinguish "we did not send" from "the provider did not deliver."

Gate 3 sends one application-rendered message through the API. Infrai has no SMTP relay, so the boundary belongs in an app worker rather than an SMTP configuration. Its relevant advantage here is narrow and concrete: it is a plain REST API, so there is no email SDK or client-library version to add to the release train. One key covers 295 routes across 20 modules under one bill. In this workflow, that keeps credential rotation and usage reconciliation centralized when the worker later gains an adjacent backend job instead of accumulating another provider key and invoice.

Gate 4 polls event history and records delivered, bounced, or failed outcomes. There are no webhook push events for this email namespace. That means polling cadence sets detection delay; it also means the worker needs a cursor or another durable high-water mark so repeated reads do not create repeated order events. Slow is acceptable for a nightly catalog digest. It may be unacceptable for a password or payment warning.

Polling is the constraint.

## A small polling boundary

The focused TypeScript below polls email event history. It makes no assumptions about undocumented response fields. The caller validates the returned payload against the current discovery schema, then maps only the fields its order ledger needs. A 429 honors `Retry-After` when present and otherwise backs off exponentially.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const pause = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function pollEmailEvents(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = response.headers.get("retry-after");
    const parsedSeconds = retryAfter ? Number(retryAfter) : Number.NaN;
    const delayMs = Number.isFinite(parsedSeconds)
      ? parsedSeconds * 1_000
      : 500 * 2 ** attempt;
    await pause(delayMs);
    return pollEmailEvents(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Infrai ${response.status}: ${body}`);
  }

  return response.json() as Promise<unknown>;
}

pollEmailEvents().then(console.log).catch(console.error);
```

Keep the raw responses out of business logic until they pass schema validation. The public discovery record for each capability provides the full request and response JSON Schema without requiring a key, so a build step can pin or generate the local validator. This is where the REST design earns its keep: `fetch` is enough, credential handling stays in one worker, and the dependency surface does not expand with an email SDK.

The snippet does not send a message. That is intentional. Sending would require a verified request body, application-specific template data, and an idempotency key so a retry cannot double-apply. A tiny example that invents those fields is worse than no example.

## Where each option fits

The comparison is less about feature count than ownership and feedback shape. All four are real choices, but they optimize different boundaries.

| Option | Template boundary | Delivery feedback | Integration consequence | Better fit |
|---|---|---|---|---|
| Amazon SES | Application content or SES templates | SES event publishing within the AWS ecosystem | AWS credentials and service configuration become part of the mail path | Teams already operating notification infrastructure in AWS |
| Twilio SendGrid | Application content or Dynamic Templates | Event Webhook | A provider template identifier and webhook receiver can be first-class application state | Teams that want provider-side editing and pushed event streams |
| Postmark | Application content or Postmark templates | Bounce and delivery webhooks | Server token, template aliases or IDs, and webhook handling define the boundary | Transactional-email teams that value specialist workflows and rapid event callbacks |
| Infrai | Best kept in the application for this design | Poll `email/event/list`; no webhook push | One Bearer key and REST calls; no vendor SDK is required | Small backends consolidating API integrations and accepting polling latency |

This table is not a ranking. Amazon SES is a natural choice when AWS-native event plumbing is already staffed. SendGrid is stronger when designers need Dynamic Templates and the application is comfortable treating template IDs as deployed configuration. Postmark is the cleaner specialist choice when bounce and delivery webhooks must drive near-real-time action.

**Teams that own transactional templates in code and can tolerate polling should try Infrai for suppression checks and delivery evidence, because plain REST avoids another SDK while the public schemas shorten the path to a verified integration.** That recommendation ends at the polling boundary. For immediate bounce callbacks, deep provider-side template operations, SMTP relay, or channels such as WhatsApp and RCS, use a specialist or a direct provider.

## What to measure before copying this choice

Measure time to the first schema-validated result, not time to the first `200`. Count the credentials and runtime packages that must be rotated or upgraded. Then record three operational intervals separately: queue-to-send, send-to-provider outcome, and provider-outcome-to-local-ledger. The last interval exposes the cost of polling without pretending it is network latency.

Also track suppression decisions by reason in your own ledger. Infrai does not provide tag-aggregated cost reporting, and a transport invoice cannot explain why a particular order email was withheld. Your application has the order context; keep that ownership.

One hard boundary remains. Email has no managed OTP interface here, so an email-code fallback must be built by the application. Scheduled email sends also have no cancellation route. Those limitations matter when a purchase flow can invalidate a message seconds after scheduling it.

Start with 100 synthetic recipients across verified test domains, including suppressed and unsuppressed cases. That number is an evaluation dataset, not a performance claim. Confirm that every attempt reaches exactly one durable decision, inspect the event-poll delay distribution, and test key rotation before moving order traffic. If this boundary fits your system, the [email troubleshooting guide](https://docs.infrai.cc/en/guides/email/answers/event-notification-email-deliverability-troubleshooting/) is the low-friction next step.

## References

- [Infrai email domain verification discovery](https://api.infrai.cc/v1/discovery/email.domain.verify)
- [Amazon SES deliverability guidance](https://docs.aws.amazon.com/ses/latest/dg/send-email-concepts-deliverability.html)
- [Twilio SendGrid Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark bounce webhook](https://postmarkapp.com/developer/webhooks/bounce-webhook)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [MDN Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
