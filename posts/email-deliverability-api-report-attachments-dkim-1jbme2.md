# Email Deliverability API Report Attachments, DKIM Rotation, and EU US SaaS Compliance

Short answer: for a media SaaS that emails generated reports, choose the design that keeps templates in your repository, treats domain verification and DKIM rotation as state machines, and records suppression and delivery events durably. The API matters, but ownership of the message template is the decision that determines how painful every later migration will be.

A design review of a report-mail workflow often starts with a harmless sequence: render a Mustache template, attach a PDF, call an email endpoint, and poll for status. Putting the template in the delivery provider makes the demo fast and makes every copy change a support ticket. Keeping the template, attachment manifest, and compliance tests in the application leaves the delivery API with a rendered payload. Small distinction. Large operational effect.

## What makes report email different from ordinary transactional mail?

A generated report has two lifecycles. The document may be regenerated when a data query finishes, while the email may be retried after a timeout. If those lifecycles share one opaque request, a retry can send two reports or attach yesterday's file. Give the report a stable job ID, a content hash, and an idempotency key. Store the rendered output before sending, and retain the exact template version used.

Template ownership is a portability control. Mustache's deliberately small syntax makes a renderer easy to run in a worker, a test process, or a migration script. It also means you must define escaping, missing-variable behavior, and attachment naming yourself. A template that only exists in a vendor dashboard is hard to review, diff, or reproduce during an incident.

Keep the artifact, too. In one realistic failure path, the report worker finished at 09:00, wrote a PDF to a shared temporary filename, and queued an email for a retryable network error. A second run refreshed that filename before the first upload completed. The message was accepted, but its attachment belonged to the newer report. The fix is not a clever retry loop: write immutable bytes to content-addressed storage, persist the hash beside the job record, and have the sender load that exact object for every attempt. Log the template version, renderer version, recipient policy result, and attachment hash together. When support asks which report was sent, those four fields should answer without reconstructing state from provider logs. This is also why a dashboard editor is a poor source of truth for regulated mail: you cannot review a diff for a one-word change, reproduce an old render after a migration, or prove which approval preceded a send.

Ship it.

Here is the shape I use at the application boundary. It is intentionally boring.

```ts
type ReportMessage = {
  jobId: string;
  templateVersion: string;
  to: string;
  subject: string;
  html: string;
  attachmentSha256: string;
  attachmentName: string;
};

async function deliverReport(message: ReportMessage): Promise<void> {
  const response = await fetch(process.env.MAIL_API_URL!, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "idempotency-key": message.jobId,
    },
    body: JSON.stringify(message),
  });

  if (!response.ok) {
    throw new Error(`delivery submission failed: ${response.status}`);
  }
}
```

The endpoint is an adapter, not business logic. That separation lets a team compare SendGrid, Postmark, Mailgun, Amazon SES, or a self-hosted SMTP relay against the same contract without rewriting report generation. Their retention, template editors, event models, and regional controls differ, so the adapter should expose those differences instead of hiding them behind a false universal abstraction.

## How should an email deliverability API handle verification, DKIM rotation, suppression, and polling?

Treat domain verification as a record with a desired state and an observed state. DNS checks are asynchronous, and a green result today does not remove the need to monitor tomorrow. For DKIM rotation, publish the new selector before switching signing traffic, wait for resolver propagation, then retire the old selector after a measured overlap window. Keep both selectors in configuration during the overlap; deleting the old one first turns a routine rotation into a deliverability incident.

Suppression is a policy decision, not merely a provider feature. A hard bounce, a complaint, and an unsubscribed recipient should have distinct reasons, retention rules, and audit entries. Your send path should check the local suppression store before submission, then reconcile provider events back into it. Do not infer consent from a successful API response.

Polling can be the safer integration for a small team when webhooks would require a public endpoint, signature verification, replay handling, and a second monitoring path. Poll with a cursor, persist the last cursor only after processing a page, and make event application idempotent. Webhooks reduce detection latency, but they do not remove the need for reconciliation. I am not sure which model wins for your traffic until you measure event lag, duplicate rate, and operational pages over a week.

For EU and US SaaS, compliance is a data-flow property. Record where addresses, report content, event logs, and suppression reasons reside; document the lawful basis and retention period for each. Regional routing can help, but a US-hosted analytics sink can still pull personal data out of the region. Ask each platform for subprocessor lists, deletion semantics, export behavior, and log retention instead of treating a compliance badge as an architecture.

## A comparison method that survives vendor changes

Run the same fixture through every candidate: one verified domain, one rotating selector, one valid recipient, one hard bounce, one complaint, and one delayed event. Score observable behavior rather than dashboard polish. Can you export events? Can you delete a recipient's data? Can you replay a missed page? Can you attach a 12 MB report without changing the template? Can an operator see why a message was suppressed?

The useful comparison table is about boundaries:

| Concern | Provider-hosted template | Repository-owned template |
| --- | --- | --- |
| Copy review | Dashboard permissions and screenshots | Pull request and tests |
| Migration | Rebuild templates in a new syntax | Keep renderer, replace adapter |
| Compliance evidence | Export depends on provider tooling | Versioned source and render logs |
| Failure recovery | Provider may own the last render | Stored artifact can be retried |

The catch is that repository ownership costs engineering time. Someone must maintain rendering tests, sanitize variables, and manage attachment storage. If a nontechnical team edits dozens of campaigns daily, a mature visual editor may be the better fit; stick with a provider-managed template system when its audit trail and approval workflow are more important than portability.

## The failure modes worth testing before launch

Test the awkward paths, not just a 200 response. Submit the same idempotency key twice. Rotate DKIM while a resolver still serves the old selector. Remove a recipient between suppression lookup and send. Let a poller crash after applying half a page. Re-run it. A useful event ledger has a unique provider-event ID, received time, processed time, and reason for every state transition.

One incident pattern deserves a blunt warning: an attachment generated from a mutable temporary path can be replaced between hashing and upload. Hash the final bytes, upload those bytes, and compare the provider's accepted size or checksum when available. A report that is deliverable but wrong is still a production failure.

Measure before copying this design: time from submission to inbox, bounce and complaint rates by domain, event lag, duplicate sends, template-render failures, and the percentage of events reconciled by polling. Those numbers tell you whether a webhook, a regional endpoint, or a different provider is actually solving your problem.

## References

- https://mustache.github.io/mustache.5.html
- https://senders.yahooinc.com/best-practices/
- https://www.rfc-editor.org/rfc/rfc6376
- https://www.rfc-editor.org/rfc/rfc7489
- https://www.rfc-editor.org/rfc/rfc8058
