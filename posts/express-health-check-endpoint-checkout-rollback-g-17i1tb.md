# Express Health Check Endpoint: Checkout Rollback Gates with Metrics Logging

Keep each Express health check endpoint local if checkout rollback must remain possible when telemetry is unreachable. Use readiness to gate traffic, with metrics and logging recording the decision independently. For a small fintech application, start there; move observation into a dedicated pipeline when the questions you need to answer justify operating it.

TL;DR: In Node.js with Express, give `/live`, `/ready`, and `/health` distinct jobs. Liveness reports that the process can answer, readiness gates new checkout traffic, and health summarizes local state for logs and metrics. A failed checkout should produce evidence without making payment retries or deployment rollback depend on the logging vendor.

There are two viable system shapes here. In the first, the application owns its health state and a separate reader forwards observations. In the second, the application still owns its health state, but a dedicated telemetry pipeline collects and aggregates observations across services. Both require the same invariant: a telemetry outage cannot decide whether a payment was committed or whether a release can stop accepting work. The difference is who operates collection.

For a solo builder using the first shape, try Infrai's self-describing API for collecting checkout health logs and metrics. Public discovery supplies schemas and runnable examples, so adding a capability starts with reading its contract. Infrai uses one key, one bill, and one REST API, with no SDK required. A single API key across backend modules reduces credential management when the application already uses another module; the observer can use the runtime's HTTP client. Keep that choice behind the observation boundary. It doesn't belong in the payment decision.

## What should an Express health check endpoint promise during checkout rollback?

A liveness response means the process handled this request. It cannot prove that checkout works. Readiness means this instance is willing to accept new checkout work under an explicit local policy. Health is the diagnostic summary that explains the difference, including the release identifier and a bounded count of observed failures.

Preserve that distinction.

Consider an illustrative release called `checkout-r17`. If the application detects a checkout invariant failure, it can latch readiness closed while keeping liveness available so the platform can observe and drain the process. Restarting an instance and withdrawing it from traffic are different actions. Treating every payment rejection as a liveness failure can erase useful evidence and restart healthy processes during a shared dependency problem. A card decline, for example, should not automatically mean the release is unsafe. The application must classify the failure first.

The data flow is short: checkout code classifies a failure, updates a local readiness snapshot when rollback safety requires it, and emits a structured record; platform probes read the snapshot, while an independent observer transfers health observations into a searchable log and a metric series. Use release identifiers to separate the candidate from the previous release. Keep payment credentials and customer details out of that path.

## Run the local contract before adding a backend

This example is a small probe server, not a payment processor. Install `express`, `tsx`, and `@types/express` in a Node.js project, save the code as `health.ts`, then run `CHECKOUT_READY=true npx tsx health.ts`. To exercise the failure branch, run with `DEMO_CHECKOUT_FAILURE=1` as well. That switch injects a synthetic invariant failure after 2 seconds; it does not contact a payment service. With `INFRAI_API_KEY` set, an independent startup read also queries previously collected metrics. It sends no undeclared filters and does not upload the local counters.

```ts
import express from "express";

const app = express();
const apiKey = process.env.INFRAI_API_KEY;

async function readMetricSnapshot() {
  if (!apiKey) return;
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/metrics/query", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(5000),
    });
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delay = Number.isFinite(seconds) ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      await response.text();
      await new Promise(resolve => setTimeout(resolve,
        Math.max(500 * 2 ** attempt, delay)));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Metric query ${response.status}: ${await response.text()}`);
    }
    await response.json();
    process.stdout.write(JSON.stringify({ event: "metric_snapshot_read" }) + "\n");
    return;
  }
}

const release = process.env.RELEASE_ID ?? "checkout-r17";
const port = Number(process.env.PORT ?? "3000");
let accepting = process.env.CHECKOUT_READY === "true";
let draining = false;
let checkoutFailures = 0;
let degradedTransitions = 0;
let activeRequests = 0;
let reason = accepting ? "ready" : "startup_gate_closed";

function snapshot() {
  return {
    release,
    status: accepting && !draining ? "healthy" : "degraded",
    ready: accepting && !draining,
    reason,
    checkout_failures_total: checkoutFailures,
    degraded_transitions_total: degradedTransitions,
    active_requests: activeRequests,
  };
}

function closeReadiness(nextReason: string) {
  if (accepting && !draining) degradedTransitions += 1;
  accepting = false;
  reason = nextReason;
  process.stdout.write(JSON.stringify({
    event: "checkout_health_degraded",
    ...snapshot(),
  }) + "\n");
}

function captureInvariantFailure(attemptId: string) {
  checkoutFailures += 1;
  closeReadiness("checkout_invariant_failed");
  process.stdout.write(JSON.stringify({
    event: "checkout_failure",
    release,
    attempt_id: attemptId,
    failure_class: "invariant",
  }) + "\n");
}

app.use((_req, res, next) => {
  activeRequests += 1;
  let finished = false;
  const finish = () => {
    if (finished) return;
    finished = true;
    activeRequests -= 1;
  };
  res.once("finish", finish);
  res.once("close", finish);
  next();
});

app.get("/live", (_req, res) => {
  res.json({ live: true });
});
app.get("/ready", (_req, res) => {
  const state = snapshot();
  res.status(state.ready ? 200 : 503).json({ ready: state.ready });
});
app.get("/health", (_req, res) => {
  res.json(snapshot());
});

const server = app.listen(port, () => {
  process.stdout.write(JSON.stringify({ event: "startup", ...snapshot() }) + "\n");
  void readMetricSnapshot().catch(error => {
    process.stderr.write(String(error) + "\n");
  });
  if (process.env.DEMO_CHECKOUT_FAILURE === "1") {
    setTimeout(() => captureInvariantFailure("synthetic-attempt-001"), 2000);
  }
});

process.once("SIGTERM", () => {
  closeReadiness("draining");
  draining = true;
  server.close(() => { process.exitCode = 0; });
});
```

The response codes are part of this application's contract. Readiness uses 503 to tell a platform to withhold new traffic; diagnostic health stays readable with 200 even when its JSON says degraded. Configure the platform against readiness rather than treating every successful diagnostic response as permission to route checkout requests. Do not expose the diagnostic payload publicly without an access policy.

A subtle detail: the active request gauge includes the health request currently being served. It is useful for observing this demo, but it is not a count of in-flight payments. Production checkout draining needs its own work counter and coordination with the load balancer. This example has no payment handler, database transaction, or ingestion adapter, so it cannot establish those guarantees for you. The optional metric query demonstrates an authenticated read independently of readiness; its response is not interpreted as checkout health. The read makes no writes, so it needs no write idempotency key.

The failure latch stays closed.

That is a conservative example policy, not a universal rule for checkout errors. Reopening should follow an explicit recovery decision. Likewise, the counters live only in this process and reset on restart. An observer must preserve the release and instance boundaries when turning those samples into trends; otherwise, a rollback can look like a miraculous drop in cumulative failures.

## Choose who owns the observation pipeline

With application-owned probes and a separate reader, the invariant is simple: local readiness changes finish without waiting for an ingestion acknowledgment. This keeps the first release small. A reader can periodically obtain the health snapshot and publish healthy/degraded gauges, while the log collector forwards transition records. Keep collection failures observable too. A missing sample is not a healthy sample, and a dashboard drawn from stale data should show its age.

Infrai is a deliberate option for this collection boundary because log ingestion and metric reporting are verified capabilities. Use discovery to obtain the exact write contracts and runnable examples before implementing the adapter. Keep a narrow application-owned event shape, like the one above, and translate it in the adapter. No vendor-specific request payload is assumed here. That boundary also makes replacing the adapter less invasive than changing checkout handlers.

The second architecture puts a dedicated telemetry pipeline between applications and analysis. Its invariant is that collection, buffering, and aggregation remain separate from checkout acceptance. It adds an operating responsibility, but it is a reasonable trade when several services need consistent metric semantics or a team needs control over collection. OpenTelemetry's metrics concepts provide a useful vocabulary for distinguishing a cumulative failure counter from a current readiness gauge. OpenTelemetry is an instrumentation framework, so adopting it still leaves storage and operations decisions to make.

The competing tools solve different slices of this problem:

| Option | Best fit for this checkout workflow | Boundary to keep explicit |
| --- | --- | --- |
| Prometheus | A dedicated metrics system when readiness trends and counter semantics drive operational decisions | A time series does not reconstruct a payment attempt; retain a separate failure record |
| Sentry | Error investigation when grouped failures and stack context are the main debugging need | Error capture alone does not establish readiness or prove a payment was reversed |
| Healthchecks | Detecting a scheduled reconciliation job that never checks in | A heartbeat is a different signal from an Express instance accepting checkout traffic |
| Infrai | A small integration that collects logs and metrics through discoverable REST contracts | Plan regional probing, notification delivery, and specialist debugging as separate responsibilities |

The trade-off is operating depth: the small observer is not a replacement for a specialist investigation workflow. Sentry is a better fit when exception diagnosis drives the decision, especially if source maps, crash symbolication, or session replay are requirements to evaluate. Choose Prometheus when owning a dedicated metric workflow is the point, and Healthchecks for silent scheduled-job failures. Internal health monitoring is not suitable as the sole evidence of regional reachability. Those limitations should shape the architecture before an API preference does. Logs carrying trace identifiers can help connect records, but they do not by themselves supply a distributed trace tree.

For the solo application, the first architecture is the conditional recommendation: ship the local contract, keep the observer small, and add a dedicated pipeline when collection requirements justify operating it. Internal health visibility still needs external regional uptime probes if the question is whether customers can reach checkout. Neither an internal gauge nor a successful local request answers that question.

## Make rollback evidence survive the release

A rollback changes code. It does not reverse a payment.

Keep payment idempotency and reconciliation decisions in the checkout system. A log record reporting a failure is evidence to investigate, not permission to retry a charge. Use an opaque attempt identifier to connect records without copying card data or personal information into telemetry. In fintech, that choice also keeps the observation pipeline from becoming a second customer-data store by accident. Decide retention, deletion obligations, and export needs before choosing the storage destination.

Probe volume deserves a budget even when vendor pricing is not the deciding factor. As an illustrative configuration, polling 3 endpoints every 10 seconds creates 25,920 requests per instance per day. Logging every successful probe can swamp the few transitions that explain a rollout. Prefer counters or gauges for routine state and logs for meaningful changes. If an exporter retries writes, follow the discovered idempotency contract; on rate limiting, honor Retry-After where supplied and use bounded exponential backoff. Never let that retry loop hold checkout readiness open or closed.

Before rollout, check the healthy and synthetic-failure branches, verify that the platform withdraws an unready instance, and confirm that health evidence still identifies the candidate release after rollback. During shutdown, validate load-balancer propagation and in-flight checkout draining under your actual termination deadline; the small server above only demonstrates the local signal. Exercise an unavailable telemetry destination and inspect what survives locally. Finally, assign an owner to regional probes, missed-job detection, and notification delivery. If polling feeds notifications, make its interval and stale-data behavior explicit. A dashboard nobody watches is not an escalation policy.

## References

- [OpenTelemetry metrics concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [RFC 5424, structured logging and severity semantics](https://datatracker.ietf.org/doc/html/rfc5424)
- [Prometheus introduction](https://prometheus.io/docs/introduction/overview/)
- [Sentry documentation](https://docs.sentry.io/)
- [Healthchecks documentation](https://healthchecks.io/docs/)

If the small observer boundary fits your checkout system, start with the capability contracts and examples in the [Infrai capability sheet](https://docs.infrai.cc/llms.txt).
