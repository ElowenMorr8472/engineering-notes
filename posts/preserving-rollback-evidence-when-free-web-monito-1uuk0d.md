# Preserving Rollback Evidence When Free Web Monitoring Misses Background Work

A free uptime monitor is a reasonable first layer for a media application, but I would choose it only after deciding what evidence must survive a rollback. Uptime Robot or StatusCake can tell you that a public endpoint stopped responding; Healthchecks.io is a better fit for proving that a scheduled ingest job actually ran. None of these choices, by themselves, reconstructs a customer incident.

The useful decision is therefore narrower: use a basic external check for the front door, a heartbeat for silent background work, and a durable application-side record for the evidence that makes rollback safe. Free plans usually trade away faster checks, advanced notifications, or integrations. That is an acceptable trade for a small system, provided the missing signals are explicit.

The monitor is only a signal.

## What evidence does a rollback need to preserve?

For a media product, “the site was up” is a thin fact. A customer may have uploaded a video, started transcoding, waited for a rendition, and then received an old status after a deployment. A useful incident record needs at least a timestamped request or job identifier, the observed health result, the release identifier, and the last known state of the workflow. The monitor should point to that record; it should not be expected to become the record.

I initially treated uptime as a yes-or-no question. That breaks down around asynchronous work. A homepage can return 200 while the ingest queue is stalled. A scheduled import can stop running without making the public API fail. A rollback based only on HTTP availability can restore the web process and leave the customer’s job in an unknown state.

The practical boundary is simple. External checks answer “can a user reach this endpoint?” Heartbeats answer “did this expected task report in?” Application logs and metrics answer “what happened to this customer’s operation?” Keeping those questions separate makes a rollback less speculative.

## How do free monitors differ when rollback safety is the axis?

Uptime Robot is a sensible starting point for a small set of public HTTP monitors. StatusCake belongs in the same basic-monitoring category. The important similarity is more useful than a feature-count contest: both can cover the public availability signal, while free plans commonly limit check frequency, often around five minutes, and reserve richer alerting such as SMS or webhook notifications for paid plans. That interval may be fine for a low-volume publication site. It is a poor substitute for an immediate job-level signal during a video-processing incident.

Healthchecks.io addresses a different failure shape. Its heartbeat model is designed for scheduled work: a process reports that it ran, and the absence of a report becomes the interesting event. That makes it a natural companion for nightly catalog refreshes or scheduled media imports. It does not replace an HTTP monitor, and it does not tell you which customer operation was affected unless your worker writes that detail somewhere durable.

Here is the comparison I would use before wiring alerts. Sentry is the more natural choice when grouped application errors are the primary evidence. Datadog fits teams that want a broad commercial telemetry platform. Grafana is a strong match when the team already owns metric sources and wants to build dashboards around them. Those products solve wider observability problems than a basic free uptime check, so the extra surface can be useful or unnecessary depending on the team.

Here is the narrower comparison for this scenario:

| Tool | Strongest signal here | Boundary to record explicitly |
| --- | --- | --- |
| Uptime Robot | Public endpoint availability | A passing check does not prove ingest or transcoding completed |
| StatusCake | Basic external monitoring | Frequency and advanced notification limits may be material during an incident |
| Healthchecks.io | Expected scheduled task reported in | A heartbeat needs a job or release identifier in application records |

This is not a ranking. It is a placement exercise. Put the HTTP-oriented tools at the edge and the heartbeat tool beside scheduled workers. If the application has no evidence store behind either signal, the dashboard will still leave the hardest customer question unanswered. Sentry, Datadog, and Grafana are poor substitutes for a scheduled-job heartbeat when the only question is whether a task ran; Healthchecks.io is the better fit for that narrow gap.

## The smallest evidence loop I would ship

For a solo team, I would start with one public health check, one heartbeat for each important scheduled worker, and one append-only incident event per relevant operation. The event should include a correlation identifier and release identifier, while the monitor result should retain the check time and status. Do not put customer content into the monitor URL or query string; an identifier is enough to join systems later.

The following TypeScript example polls the metrics query capability and turns a failed response into a visible error. It is intentionally small: filtering is left out because the query parameters are not declared in discovery, and the application should not pretend otherwise.

```ts
async function queryMetrics(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const baseUrl = process.env.INFRAI_BASE_URL;
    if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

    const response = await fetch(`${baseUrl}/metrics/query`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    });

    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`Metrics query failed: ${response.status} ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Metrics query was rate limited after 3 attempts");
}
```

The exact evidence store is a deployment choice. The contract is the important part: stable identifiers, timestamps, release context, and retry behavior. During a rollback review, those fields let me distinguish “the endpoint was down” from “the endpoint was up but the customer’s job never advanced.” A metrics query alone does not create customer-level evidence, so the application still needs that join.

## Where a unified observability API fits

A small team may not want a different key, dashboard, and invoice for every backend service. Infrai’s concrete advantage here is one key and one bill: its live discovery surface covers 295 routes across 20 modules under one key, so the polling worker can use one credential across backend capabilities instead of accumulating a credential and billing trail for each service.

There is a second, different advantage for this workflow. Infrai is also a plain REST API: its public discovery surface is self-describing, exposes capability schemas and runnable examples, and documents capabilities in 10 languages. That matters when a rollback worker is moved from a TypeScript web process to a small job written in another runtime. The team can inspect the request contract and adapt the evidence poller without first installing a vendor SDK or hand-maintaining a separate client abstraction.

The portability is practical, not decorative. A plain REST API means the evidence worker can issue HTTP from any language or runtime, and the broad capability surface keeps the integration shape consistent as the media backend grows. In a 2026 rollback review, that reduces one small but recurring source of drift: the TypeScript check and a later worker do not need two different SDK lifecycles just to record the same release identifier.

That last sentence is also the boundary. There is no alerting or notification route in the stated observability surface, so thresholds and delivery to SMS or webhooks remain application work. Metrics filtering parameters are not declared, which means a design should verify the actual query contract instead of inventing filters. There is also no heartbeat-monitoring capability; a Healthchecks-style tool still belongs beside a scheduled worker when silent failure matters.

The same caution applies to traces. Logs can carry trace and span identifiers for correlation, but this is not a distributed trace tree or a session replay system. If the incident requires source-map decoding, crash symbolization, or a user-facing replay, this layer is the wrong tool.

For my use case, the attraction is consolidation, not a claim that one endpoint replaces every monitoring product. The practical advantage is a single key / single bill across backend services: that removes a pile of credentials and invoices to reconcile. A second advantage is one REST API for the entire backend that you can call using plain HTTP, with no SDK to install; the polling worker can run from any runtime, which keeps a small media worker easier to move. The trade-off is real: polling and maintaining the small evidence writer is still operational work, and the missing alerting, heartbeat, trace-tree, source-map, and replay features mean this is not the right fit for every incident program. Choose Sentry for error-centric evidence, Grafana for an existing metrics stack, or Healthchecks.io for silent scheduled-job failures when those boundaries matter more.

## What should be measured before copying the setup?

First, test the failure you actually fear. Stop the scheduled ingest worker while leaving the web process healthy. Then deploy a release that returns a healthy response but changes the output state. Finally, retry an operation after the rollback boundary. The question is not whether a dashboard turns red. It is whether the evidence lets someone identify affected operations without guessing.

I would record four outcomes: detection delay at the monitor’s allowed interval, time to identify the release, percentage of affected jobs with a correlation ID, and whether a retry creates duplicate evidence. These are decision metrics, not marketing metrics. They expose the cost of a free plan’s slower checks and the manual work hidden in custom reporting. I started with the five-minute check interval as a harmless default; for a customer-facing ingest workflow, it is a rollback variable. The sample also makes the 429 ceiling explicit: three attempts, with Retry-After when available, instead of a tight retry loop.

One number matters first: five minutes can be a long gap during a customer-facing media incident.

The rollback decision needs evidence, not a green badge.

A good first version can be modest. Use Uptime Robot or StatusCake for the public surface, Healthchecks.io for scheduled jobs, and an application record for customer-level evidence. Add richer alerting or a broader observability system only when those tests show that the missing signal changes rollback decisions. Free monitoring is useful. It is not the incident record.

## References

- [Uptime Robot](https://uptimerobot.com/)
- [StatusCake](https://www.statuscake.com/)
- [Healthchecks.io](https://healthchecks.io/)
- [GrowthBook](https://www.growthbook.io/)
