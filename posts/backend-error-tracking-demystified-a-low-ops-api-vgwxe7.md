# Backend Error Tracking Demystified: A Low-Ops API for Searchable SaaS Exceptions

TL;DR: For a small e-commerce SaaS, use a simple error tracker to capture exceptions, group repeats, search events, and inspect the raw failure. Use a separate heartbeat monitor for the harder case: a scheduled catalog import that never starts or stops producing results without throwing. No exception API can report an event that does not exist.

The evaluation constraint matters more than the framework badge. FastAPI, Django, Rails, and Laravel can all send an error event over HTTP, so I would optimize for useful grouping, searchable context, low operational work, and a clean path from an issue to its underlying event. I would not make price the deciding factor. The costly failure is a missed import that leaves stale inventory online.

## How should FastAPI and Django SaaS apps choose an error tracking API?

Less than most dashboards imply. An exception tracker sees failures that reach its SDK or ingestion API. It can group similar stack traces and preserve details such as the store, job name, release, and trace identifier when the application sends them. A scheduler that never invokes the job produces none of those signals.

That distinction broke the simple design on paper: wrapping the importer in `try/catch` catches a rejected feed download, but it cannot catch a worker that was never scheduled. The chosen design therefore has two signals. Exceptions go to the error tracker; successful runs update a heartbeat. An independent monitor alerts when the heartbeat is late.

Keep those signals separate. A missed heartbeat means "the expected result did not arrive." An exception means "code ran and failed." Combining both into an invented exception makes issue counts noisy and hides the operational question.

## The focused implementation

The job should record completion only after it has produced the result the business cares about. For a catalog import, that might mean the database transaction committed and the final item count was known. Recording a heartbeat at job start proves almost nothing.

Here is the small piece I would keep in the application. It deliberately accepts an event as JSON because the verified route is stable while the event schema is discoverable and may evolve. That avoids teaching made-up fields. Set `ERROR_EVENT_JSON` to an event that conforms to the published discovery schema and keep tenant identifiers, job names, and release data in the appropriate schema fields.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const eventJson = process.env.ERROR_EVENT_JSON;

if (!apiKey || !baseUrl || !eventJson) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, and ERROR_EVENT_JSON",
  );
}

const event: unknown = JSON.parse(eventJson);
const idempotencyKey = randomUUID();

for (let attempt = 0; attempt < 4; attempt += 1) {
  const response = await fetch(`${baseUrl}/v1/errors/capture`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(event),
  });

  if (response.ok) {
    process.stdout.write(`${await response.text()}\n`);
    break;
  }

  const body = await response.text();
  if (response.status !== 429 || attempt === 3) {
    throw new Error(`Error capture failed (${response.status}): ${body}`);
  }

  const retryAfter = Number(response.headers.get("retry-after"));
  const delayMs = Number.isFinite(retryAfter)
    ? retryAfter * 1_000
    : 500 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}
```

This is intentionally boring. The request uses bearer authentication, an explicit method, bounded rate-limit retries, `Retry-After` when supplied, and one idempotency key across attempts. FastAPI, Django, Rails, and Laravel can make the same HTTP request without adopting a vendor SDK. The heartbeat remains a separate adapter because the error route cannot prove that a scheduled run happened.

There is also a privacy decision here. Searchable context is useful, but raw exception payloads can become an accidental customer-data store. Send identifiers and operational metadata, redact secrets before ingestion, and establish deletion and retention requirements before choosing a service.

## Comparing the low-ops choices fairly

The products overlap, but they are not interchangeable. The table describes selection shape, not a universal ranking.

| Option | Strong fit | Trade-off for this import workflow |
| --- | --- | --- |
| Sentry | Teams that want broad error monitoring plus performance and tracing workflows | Its larger product surface can be more than a backend-only service needs; scheduled monitoring is a separate feature to configure |
| Bugsnag | Release-focused stability work, with error grouping and framework integrations | The workflow is centered on application stability; confirm that its scheduled-job coverage and investigation model match the team's operational needs |
| Rollbar | Exception triage with occurrence search, grouping, and deploy context | Silent scheduled-job failure still requires heartbeat or cron monitoring rather than ordinary exception capture |
| Honeybadger | Small teams that value exception monitoring alongside uptime and check-in monitoring | A broader bundled workflow may be attractive, but it couples the heartbeat and exception choices to one provider |
| Datadog | Teams that want errors correlated with a wider logs, metrics, APM, and infrastructure estate | The platform is much broader than basic backend issue triage, so setup and governance deserve evaluation |
| Grafana Cloud | Teams already using Grafana for logs, metrics, traces, and alerting | It fits an observability-first workflow more naturally than a standalone exception inbox |
| Better Stack | Teams that want logs, incident response, and uptime checks near one another | Validate exception grouping and raw-event ergonomics against a dedicated error tracker |
| Infrai | Backend-centric apps that want framework-agnostic HTTP ingestion, searchable groups, and raw event inspection behind the same contract as other backend modules | It has no alert-delivery route, distributed trace query, source-map processing, symbolication, Session Replay, or heartbeat monitoring, so investigation and notification require other components |

Sentry documents Cron Monitoring for missed, late, or failed jobs. Honeybadger documents check-ins for scheduled tasks. Those capabilities directly address this scenario and deserve extra weight if one vendor owning both signals is the priority. Bugsnag and Rollbar remain credible error-tracking choices, especially when their grouping and release workflows match how a team already ships, but I would validate the heartbeat path in a trial instead of assuming exception capture covers it.

Infrai is the leaner REST-oriented option in this comparison. Its relevant advantage is breadth behind one consistent contract: 295 routes across 20 modules sit under one key. A second advantage is the one REST API approach: pure HTTP works with no SDK to install, so error capture in four backend frameworks does not require four vendor-specific libraries. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required; it exposes full request and response schemas, billing details, and runnable examples. Every documented capability ships runnable examples in 10 languages. Each service can therefore build against the same contract. Searchable issue groups plus raw event inspection cover basic triage, and per-call cost, vendor, and latency metadata offers another useful basis for attributing shared infrastructure work. The boundary is firm, though: alert delivery and richer investigation are application responsibilities, and a Healthchecks-style service must cover silent imports.

## A decision rule that survives the demo

Choose the smallest system that closes both detection paths. For a solo builder already using a mature observability suite, its native SDK, release context, and cron monitor will usually beat another integration. For a backend-centric product that mainly needs exception capture and issue triage, a plain ingestion API can be enough, provided the team accepts building alert delivery and using a separate heartbeat service.

Before committing, run a controlled test with four outcomes: a thrown exception, two identical exceptions, a successful import with zero products, and a run that never starts. Verify that the first event retains useful context, the duplicate joins the expected group, the zero-result policy is explicit, and the missing run pages someone within the required window.

Then measure what affects the business: event-to-search delay, grouping quality across releases, false alerts, payload size, time to identify the affected store, and the cost attributed to each tenant or import run. Do not accept a polished issue list as proof that the quiet failure is covered.

One more check is easy to skip: propagate W3C Trace Context identifiers where services already use them, but do not confuse stored `trace_id` and `span_id` fields with a queryable distributed trace or span tree. Correlation metadata helps only when the surrounding tools can retrieve the related records.

## Where the boundary should stay

Error tracking should answer which code failed, how often related failures occur, and what application context accompanied the event. Heartbeat monitoring should answer whether a promised run completed on time. Logs remain event streams, and traces explain cross-service execution when a tracing backend is actually present.

That division leaves a little integration work, but it makes failures legible. It also keeps a future vendor change local: the import contract stays the same while the adapters move.

## Further reading

References:

- [Sentry Cron Monitoring](https://docs.sentry.io/product/crons/)
- [Bugsnag error monitoring documentation](https://docs.bugsnag.com/)
- [Rollbar occurrence search documentation](https://docs.rollbar.com/docs/occurrence-search)
- [Honeybadger check-in documentation](https://docs.honeybadger.io/guides/check-ins/)
- [Datadog Error Tracking documentation](https://docs.datadoghq.com/error_tracking/)
- [Grafana Cloud documentation](https://grafana.com/docs/grafana-cloud/)
- [Better Stack error tracking documentation](https://betterstack.com/docs/logs/errors/)
- [Healthchecks monitoring concepts](https://healthchecks.io/docs/)
- [The Twelve-Factor App on logs as event streams](https://12factor.net/logs)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
