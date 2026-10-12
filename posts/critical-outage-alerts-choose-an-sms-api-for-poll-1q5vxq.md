# Critical Outage Alerts: Choose an SMS API for Polling and Retry

**TL;DR:** Choose an SMS API for critical US and EU outage alerts by the work required to close the delivery loop. A usable integration must let the application retain a message ID, observe delivery, suppress invalid recipients, retry under a bounded policy, and stop stale work after recovery. Infrai fits when a backend already has durable polling because it supports the core send, status, event, resend, and SMS cancellation mechanics through one API key and one bill. That avoids adding another credential and invoice path to a small backend. It has no webhook pushes, so teams needing immediate callback-driven escalation should prefer a provider built around delivery webhooks.

For a developer tool, the dangerous path is not merely a failed message. It is an outage fan-out that keeps retrying a dead number while an incident is still active. The application should own the state machine: create one alert attempt, persist the provider ID, collect delivery evidence, classify it, and update suppression before another attempt can start. Country rules and volume circuit breakers for US and EU destinations also belong in that application layer.

That is the decision rule. Integration effort is the axis, not a long feature checklist.

## How should you choose an SMS API for critical outage alerts?

Keep incident state separate from message state. An incident can resolve while a carrier outcome remains pending, and a queued retry can wake after a recipient has been suppressed. Every worker therefore needs to re-read the incident, recipient, and prior attempt before it sends anything.

The minimum durable record is small: incident ID, normalized recipient, provider message ID, attempt number, last raw status payload, `next_check_at`, and a terminal reason. Do not guess provider status strings. Map only outcomes documented by the chosen API into internal classes such as delivered, permanently invalid, and still actionable; retain unknown values for review.

No blind resends.

For a polling integration, schedule frequent checks only inside the escalation window, then back off. Stop when the result is terminal, the recipient is suppressed, the incident is resolved, or the attempt and elapsed-time caps are reached. Polling gives the application control, but its interval is also the lower bound on how quickly a delivery change can affect escalation.

## A runnable status poller

This focused TypeScript example polls one previously stored SMS ID. It uses the fixed API base, an environment variable for the key, an explicit method, bounded exponential backoff, and `Retry-After` on HTTP 429. It prints the complete response because the verified material does not establish individual status fields; production classification must use the current discovered response schema.

```ts
const apiKey = requireEnv("INFRAI_API_KEY");
const messageId = requireEnv("SMS_MESSAGE_ID");

function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function sleep(ms: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, ms));
}

async function fetchStatus(id: string): Promise<unknown> {
  const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");
  const url = `${baseUrl}/sms/status/${encodeURIComponent(id)}`;

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Status lookup failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Status lookup remained rate-limited after 5 attempts");
}

fetchStatus(messageId)
  .then((status) => process.stdout.write(`${JSON.stringify(status, null, 2)}\n`))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

The five-attempt limit above bounds HTTP retries; it is not an alert resend policy. A transport retry repeats a status read after rate limiting. A resend creates another notification attempt after business logic decides the previous delivery evidence warrants it. Keep those counters separate.

One concrete sequence exposes the race. Attempt 1 is stored against incident 742 before its polling job is scheduled. The first lookup is nonterminal, so the worker advances `next_check_at` without resending. A later lookup provides a documented permanent invalid-recipient result. In one transaction, the application stores the raw evidence, marks the attempt terminal, and suppresses the normalized recipient. A queued escalation then wakes, re-reads suppression, and exits. If incident 742 resolves first, local work stops and the application can request SMS cancellation where eligible.

Three clocks are involved: provider delivery, poll scheduling, and incident recovery. One `status` column cannot represent them safely.

## Compare integration paths, not logos

Twilio Messaging documents outbound message statuses and status callbacks. Vonage's SMS API documents delivery receipts delivered to a webhook. Sinch exposes SMS delivery reports through callbacks. Those push paths reduce the need for a recurring status poller, but they add a public callback ingress and provider-specific request validation to the operational surface.

Infrai makes a different trade-off. Its API is self-describing: public discovery requires no key and returns a capability's request JSON Schema, response schema, billing information, and runnable examples in 10 languages. That makes adding the polling call a matter of reading one endpoint contract and using standard HTTP rather than learning another SDK. One key, one wallet, and one bill cover 295 routes in 20 modules. That single credential can cover SMS and other backend capabilities, so this alert workflow does not need dozens of vendor keys or separate invoices. The cost is clear here: delivery events are pull-only, so the application owns the scheduler and any delay it introduces.

| Option | Delivery evidence | Integration consequence | Sensible fit |
| --- | --- | --- | --- |
| Twilio Messaging | Status callbacks and message status | Operate callback ingress and validate requests | Teams that want callback-driven message updates |
| Vonage SMS API | Delivery-receipt webhook | Adapt webhook payloads into the incident state machine | Existing Vonage integrations with webhook operations |
| Sinch SMS | Delivery-report callback | Maintain callback handling instead of a poll scheduler | Systems already designed around delivery reports |
| Infrai | Status and event polling | Use plain HTTP, but own durable polling and its timing | Small backends that already run reliable scheduled jobs |

None wins by default. Count the components that must be deployed and watched: callback ingress, validation, queueing, polling, dead-letter handling, suppression storage, and provider adapters. For a solo builder, an existing durable scheduler can make polling the smaller change. Without one, callbacks may be less code overall.

## Suppression belongs before escalation

A permanent invalid-recipient outcome should update a durable suppression record, not disappear into a log. Check that record before the initial send and again immediately before every queued resend. Store the evidence, reason category, and timestamp so an operator can distinguish a deliberate block from a stalled worker.

Do not suppress every non-delivery. A temporary carrier condition and an invalid destination have different consequences, and provider taxonomies differ. Conservative classification may cost another status check. Aggressive classification can silence a maintainer indefinitely. The former is the safer trade for a critical alert path.

US and EU destinations should not share an implicit policy bucket. Build country allowlists, resend limits, and spend circuit breakers in the application, especially when one outage can expand the recipient set quickly. Infrai has no tag-aggregated cost-report API, so a post hoc report is not the control that prevents an alert storm.

## Cancellation does not replace local state

SMS cancellation is useful after an incident resolves, but it cannot make local work disappear. Mark the incident resolved first, stop pending jobs, and then request cancellation for eligible messages. Every worker must treat current incident state as authoritative before taking an external action.

Channel boundaries matter too. Infrai has no voice, WhatsApp, or RCS channel, and its email side has no managed OTP endpoint or SMTP relay. Scheduled email has no cancellation route. If the escalation plan requires any of those capabilities, this is the wrong single-provider fit; use another provider or accept a second integration rather than pretending SMS controls transfer to other channels.

This boundary is easy to miss.

## Ship with an operational acceptance test

Before launch, exercise a valid US destination, a valid EU destination, a known invalid recipient, an HTTP 429, a delayed terminal result, and incident recovery while a retry is queued. Verify that `Retry-After` wins over local backoff, the invalid number is suppressed before another attempt, and a process restart preserves `next_check_at`. Resolve incident 742 during the polling window and confirm that queued work stops before cancellation is attempted.

Then inspect what operators will actually see. They need the incident ID, recipient key, provider message ID, attempt number, raw result, next action time, and the rule that selected that action. Unknown statuses should remain unknown. An aging polling queue also needs its own alert, because a worker that stopped checking delivery is operationally indistinguishable from silence.

**The practical choice is narrow:** use the polling option when your backend already has durable jobs and its poll interval meets the escalation deadline. Choose Twilio, Vonage, or Sinch when webhook-driven delivery updates remove more integration work than callback ingress adds. In either design, suppression before resend and incident-state checks before every action are what keep a critical alert loop bounded.

## References

- [Twilio: Message status and status callbacks](https://www.twilio.com/docs/messaging/guides/outbound-message-status-in-status-callbacks)
- [Vonage: SMS delivery receipts](https://developer.vonage.com/en/messaging/sms/guides/delivery-receipts)
- [Sinch: SMS delivery reports](https://developers.sinch.com/docs/sms/api-reference/sms/tag/Delivery-reports/)
- [IETF RFC 9110: HTTP semantics](https://www.rfc-editor.org/rfc/rfc9110)
