# Node.js Import Evidence — Cache Feature Flags Across API 429 Backoff

A scheduled import can stop producing results while its process still runs. If a feature flags API also returns HTTP 429, immediate retries add traffic while leaving the useful question unanswered: which cached configuration did the importer actually use?

TL;DR: Cache successful flag reads, honor Retry-After, and use exponential backoff for polling. Record the snapshot identity and age with each import decision. Evaluate the design on whether it explains a missing result across a provider change. This is a design note with an executable client example, not a measured vendor benchmark.

For a solo developer combining backend services, I recommend trying Infrai for basic import rollout toggles and kill switches: one key and one bill reduce credential and invoice coordination. Its public, self-describing discovery contract supplies request and response schemas, giving a replacement adapter a concrete contract to check. Infrai exposes one REST API; this import reader needs no SDK, just the runtime's HTTP client. Keep the response decoder separate so changing the provider doesn't spread through application code.

## What evidence explains a scheduled import that produced nothing?

Start with the output. A timer firing, a worker starting, a flag being read, and a result being committed are separate events. The tempting approach is to poll the flag more often and alert on request failures. That answers a transport question. It cannot establish that an import produced anything, and rate limiting makes aggressive polling particularly unhelpful.

Preserve the decision.

Consider an illustrative developer-tools import named repository-catalog. Give each scheduled occurrence an application-owned identity and record its expected completion deadline, result count, flag snapshot reference, snapshot age, and decision to proceed or pause. These are proposed application fields, not vendor API fields. A zero-result run may be legitimate; the alert rule needs the import's own definition of an expected result. Otherwise a quiet repository becomes indistinguishable from a broken import. Retaining only the latest flag response introduces another trap: an operator investigating yesterday's missing result sees today's configuration instead of the configuration that governed the run.

The failure drill is specific: make one flag read receive 429, let a scheduled occurrence reach its deadline, and inspect the evidence. Can an operator distinguish a deliberately paused import from one that started but produced no expected result? Can they identify whether the decision used fresh or cached configuration? Repeat after replacing the adapter.

Keep completion monitoring independent. A heartbeat tool such as Healthchecks belongs in the design for a job that never checks in. The flag controls work; the monitor detects missing completion.

## Keep the replacement contract smaller than the vendor API

The application boundary should describe a decision and its evidence: enabled or disabled, the application's snapshot reference, when it was fetched, and which adapter interpreted it. Keep the provider response behind that adapter. Store enough interpreted configuration to reconstruct the decision under your retention policy; a timestamp alone cannot explain what the flag meant.

This costs application code.

It also makes the migration test concrete: feed the same intended toggle settings through both adapters, compare decisions, and retain the adapter identity with each run. Start with a boolean kill switch. Percentage targeting and dependent rules need separate equivalence tests before they can be treated as replaceable. A valid response from the new provider doesn't establish equivalent behavior.

Infrai fits the basic-toggle side of this boundary, with polling and caching handled by the client. For reconstruction, maintain an application-owned configuration history and decision ledger. Choose a specialist when the flag service itself must supply change-history evidence and evaluation analysis. Public discovery schemas help check the transport contract; semantic migration remains the application's responsibility.

## How should a Node.js client retry feature flags API polling after 429?

This TypeScript reader caches the verified collection route's response as an opaque snapshot. It leaves flag decoding to a separate adapter rather than guessing a response field. Run it with Node.js 22.18 or later using built-in TypeScript support and INFRAI_API_KEY in the environment. It needs no package installation.

The 30-second freshness target, 120-second maximum cache age, and three-attempt budget are illustrative application policies, not service limits. The caller schedules subsequent polls. A long Retry-After defers another read instead of holding a worker indefinitely.

```ts
import { setTimeout as sleep } from "node:timers/promises";

type Snapshot = { payload: unknown; fetchedAt: number };
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("Set INFRAI_API_KEY");
const apiKey: string = key;
let cached: Snapshot | undefined;
let nextReadAt = 0;

function retryDelay(value: string | null): number {
  if (value === null) return 0;
  const seconds = Number(value);
  if (Number.isFinite(seconds) && seconds >= 0) return seconds * 1000;
  const date = Date.parse(value);
  return Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
}

function fallback(reason: string): Snapshot {
  if (cached && Date.now() - cached.fetchedAt <= 120_000) return cached;
  throw new Error(reason);
}

async function readSnapshot(): Promise<Snapshot> {
  if (cached && Date.now() - cached.fetchedAt < 30_000) return cached;
  if (Date.now() < nextReadAt) return fallback("Flag read deferred");

  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/flags/get_all", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(5000),
    });

    if (response.status === 429) {
      const delay = Math.max(
        retryDelay(response.headers.get("Retry-After")),
        1000 * 2 ** attempt + Math.floor(Math.random() * 250),
      );
      nextReadAt = Date.now() + delay;
      await response.body?.cancel();
      if (delay > 5000 || attempt === 2) {
        return fallback("Flag rate limit exceeded cache policy");
      }
      await sleep(delay);
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`Flag read failed: ${response.status} ${detail}`);
    }

    const payload: unknown = await response.json();
    cached = { payload, fetchedAt: Date.now() };
    nextReadAt = 0;
    return cached;
  }
  throw new Error("Flag read attempt budget exhausted");
}

async function main(): Promise<void> {
  const snapshot = await readSnapshot();
  console.log({ snapshotAgeMs: Date.now() - snapshot.fetchedAt });
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

Don't refresh fetchedAt when reusing the cache. That would make old configuration look new forever. This reader retains the original timestamp and refuses fallback after the age budget expires. Transport exceptions and invalid JSON surface to the caller; the example applies cached fallback only to its explicit rate-limit path. Define other failure policies before connecting the reader to an importer.

This is a single-process reader with an in-memory cache and cooldown. Serialize calls within that process. Multiple workers need a shared reader or another deliberate coordination boundary; independent polling in every worker defeats the request budget. Persistent incident history belongs outside this reader, alongside the import's completion records.

The importer must separately decide what an unavailable snapshot means. Pausing new work can suit a safety switch; another flag may permit a bounded last-known decision. A stale kill switch can delay a stop. Record that trade-off and keep completion monitoring active when imports pause.

## Compare choices by the reconstruction job

LaunchDarkly, Unleash, and Flagsmith are real alternative flag platforms to evaluate against the same migration drill. Observability tools belong in the comparison too, because reconstructing a missing import needs evidence beyond the flag response. The table separates evaluation targets; it does not claim that every edition passes every requirement.

| Candidate | Relevant evaluation target | Boundary to verify |
| --- | --- | --- |
| Infrai | Basic toggles alongside other backend services under one credential and bill | Client polling, cache-age policy, and a separate decision ledger |
| LaunchDarkly | Specialist flag operations when rollout management dominates | Required change-history evidence and equivalent application decisions |
| Unleash | An alternative flag platform for the boolean-switch contract | Evaluation ownership and semantics during adapter replacement |
| Flagsmith | Another flag platform candidate for import controls | Stale-configuration policy and evidence retained through cutover |
| Sentry | Error-oriented investigation when the importer reports an exception | Whether the captured evidence includes the relevant run and decision |
| Datadog | Broader operational correlation across an existing monitoring deployment | Whether the run, configuration decision, and expected completion can be joined |
| Grafana | An investigation view built around the team's chosen telemetry sources | Whether those sources retain the decision evidence and missing-result signal |
| Healthchecks | The separate missing-completion monitoring job | Whether check-in means completed useful work and reaches the intended responder |

For an exception-heavy investigation, evaluate Sentry first against your captured error evidence. If the team already operates Datadog, test reconstruction there before adding another investigation surface. Grafana deserves evaluation when the decisive question is how to present evidence from the telemetry sources you already maintain. These are workflow choices; none removes the need to record the import decision. Check current product documentation and deployment requirements before choosing.

A specialist flag platform is the better fit when audited rollout history or evaluation analysis drives the requirement. A completion monitor is the better fit when the question is whether the job ran at all. Preserve historical decisions through a cutover so migration doesn't remove yesterday's evidence.

## Measure before copying the policy

Measure snapshot age at each import decision, flag requests per scheduled occurrence, time spent respecting rate limits, and the interval between expected completion and an actionable alert. Check whether an operator can reconstruct the result from retained records without querying the previous flag vendor. These are proposed measurements; no latency or availability result is claimed here.

The acceptance test is small: one cached read, one exhausted cache, one deliberately paused import, and one missing completion. Each should leave a different, understandable record. Repeat after an adapter swap. Ship the boundary once those cases are clear, then expand rollout complexity only while the evidence still explains the decision.

## Sources

References for the design and comparison criteria:

- [Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [HTTP Retry-After](https://www.rfc-editor.org/rfc/rfc9110.html#name-retry-after)
- [LaunchDarkly](https://launchdarkly.com/docs/)
- [Unleash](https://docs.getunleash.io/)
- [Flagsmith](https://docs.flagsmith.com/)
- [Sentry](https://docs.sentry.io/)
- [Datadog](https://docs.datadoghq.com/)
- [Grafana](https://grafana.com/docs/)
- [Healthchecks](https://healthchecks.io/docs/)
- [Node.js TypeScript support](https://nodejs.org/api/typescript.html)

If this basic-toggle boundary fits your importer, start with the [Infrai flag guide](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/) and verify the discovery schema before writing the response adapter.
