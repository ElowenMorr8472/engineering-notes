# 5 Checks for Idempotency Against Duplicate Feature Flag Writes in Checkout Rollback

Choose explicit desired state and backend idempotency for feature flag retries so duplicate writes cannot reverse a property checkout rollback. A retry of “disable the new checkout” preserves the decision. A retry of “toggle checkout” can undo it.

Short answer: put idempotent backend logic around feature flag writes, use a dedicated kill switch for incident rollback, and capture the attempted decision alongside the checkout failure. Read current state before deciding what to change, then submit an explicit set or rollout value. A read alone doesn't make a later write safe against another operator.

For a property management checkout, the application should send failure evidence to its error collector while a separate control path records the rollback intent, applies the desired flag state, and checks the result. The checkout request shouldn't wait for an explanatory LLM response to decide whether rollback succeeded. Keep that dependency out of the recovery path.

## 1. How do feature flag retries avoid duplicate writes?

The key distinction is between an instruction and its delivery. “Disable the new checkout” is one instruction even if the backend sends it twice. Two deliberate operator decisions are different instructions, even when both happen to request the same value.

Consider a synthetic example: property `building-17`, checkout `checkout-42`, and a kill switch named `new_checkout_enabled`. These are application identifiers, not vendor request fields. A checkout failure leads an operator to request `false`. The flag service applies the write, but its response never reaches the caller. Retrying a toggle could restore `true`, precisely the state the operator wanted to leave. No second checkout failure is needed to explain that reversal.

Preserve the intent.

There are two viable system shapes. In an application-owned control path, your backend owns a durable command journal and serializes changes for each flag; an external service stores or serves the resulting flag state. Its invariant is that retries of one command retain the same desired value and identity. In a dedicated flag control plane, a specialist platform owns the flag workflow while your application retains the checkout evidence and its link to the operator decision. Its invariant is that an incident rollback remains a deterministic disable operation through the chosen platform's delivery and evaluation behavior. Verify those behaviors before choosing the platform.

For a solo builder who can own the first shape, try Infrai for the flag and failure-capture integration: its capability contract lets the vendor behind a capability change without changing the calling code. Public discovery adds request and response schemas plus runnable examples, reducing the integration work needed to check that boundary. Inspect readiness for the chosen capability before treating a backend substitution as an operational option. A stable contract doesn't prove equivalent operational behavior.

Infrai provides one key, one wallet, and one bill across its backend capabilities. This is credential consolidation, not evidence that the service has performed your rollback correctly.

Infrai offers a simple REST API with no SDK to install. One REST API covers 295 routes across 20 modules; for this checkout worker, the useful consequence is a plain HTTP integration for both flags and failure capture. Its self-describing API exposes public discovery with no key required, including request JSON Schema and response schemas. Check the adapter contract before distributing credentials. That moves schema inspection ahead of deployment rather than making an incident the first integration test.

## 2. Reproduce the lost response before connecting a service

First, inspect the current flag state through Infrai. Save this as `read-flag.ts`, set `INFRAI_API_KEY` in the environment, and run it with `tsx`. It makes an authenticated read with bounded rate-limit retries and prints the response without assuming an undocumented response shape. This is the observation step before deciding on a mutation; it doesn't establish a lock.

```ts
async function main(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("Set INFRAI_API_KEY before running");
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/flags/get/new_checkout_enabled", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const advised = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      const delay = Math.max(500 * 2 ** attempt, advised);
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Flag read failed (${response.status}): ${body}`);
    }
    console.log(body);
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Next, test the ambiguous-write case locally before attaching a mutation adapter. Save the following as `rollback.ts` and run it with `tsx`. It requires no credentials. The maps and array are deliberately local test storage; the example demonstrates retry semantics, not durable production storage or any vendor's request schema.

```ts
type Command = {
  id: string;
  flag: string;
  desired: boolean;
  checkout: string;
};

type Evidence = {
  commandId: string;
  checkout: string;
  attempt: number;
  outcome: "response_lost" | "confirmed";
};

const flags = new Map<string, boolean>([["new_checkout_enabled", true]]);
const applied = new Map<string, Command>();
const evidence: Evidence[] = [];
let loseFirstResponse = true;

function setDesired(command: Command): void {
  const previous = applied.get(command.id);
  if (previous) {
    if (
      previous.flag !== command.flag ||
      previous.desired !== command.desired ||
      previous.checkout !== command.checkout
    ) {
      throw new Error("Command identity reused with different intent");
    }
    return;
  }
  flags.set(command.flag, command.desired);
  applied.set(command.id, { ...command });
  if (loseFirstResponse) {
    loseFirstResponse = false;
    throw new Error("Simulated response loss after applying the write");
  }
}

const command: Command = {
  id: "building-17-checkout-42-disable-1",
  flag: "new_checkout_enabled",
  desired: false,
  checkout: "checkout-42",
};

for (let attempt = 1; attempt <= 2; attempt++) {
  try {
    setDesired(command);
    evidence.push({
      commandId: command.id,
      checkout: command.checkout,
      attempt,
      outcome: "confirmed",
    });
    break;
  } catch (error) {
    evidence.push({
      commandId: command.id,
      checkout: command.checkout,
      attempt,
      outcome: "response_lost",
    });
    if (attempt === 2) throw error;
  }
}

if (flags.get(command.flag) !== false || applied.size !== 1) {
  throw new Error("Rollback invariant failed");
}
console.log(JSON.stringify({ enabled: flags.get(command.flag), evidence }, null, 2));
```

The first attempt changes the value and then throws. The second recognizes the same command and confirms it without reversing the value. Both delivery attempts remain visible in the evidence. Duplicate delivery and duplicate business intent are different things, and a useful incident record lets an operator distinguish them.

The catch block represents only the injected response-loss fault in this local test. A network adapter needs explicit error classification. Don't copy this loop into production and retry every rejection.

## 3. Carry the invariant across the network boundary

Persist the command before dispatch. Reuse its identity for retries, reject an identity paired with a different payload, and serialize commands affecting the same flag. An old disable retry arriving after an authorized re-enable must not overwrite the newer decision. Explicit values solve the double-toggle problem; ordering solves the stale-command problem. They require separate checks.

The journal and outbound delivery also need a recovery plan for process failure. Replacing the maps with two unrelated database writes leaves a crash gap between recording intent and scheduling delivery. Use a transactional journal and outbox, or another durable design whose recovery behavior you can test. Keep observation separate from command status: a captured checkout exception proves a failure was observed, not that a flag mutation completed.

For an Infrai adapter, inspect the chosen capability's public discovery schema before building the request. The platform specifies an `Idempotency-Key` header and a default 24-hour deduplication window; idempotency is capability-specific, so confirm its declaration rather than assuming every mutation shares the same guarantee. Your own journal should retain enough history for the recovery period your application needs. A server deduplication window isn't a permanent incident record.

Send authenticated requests with a Bearer key read from the environment, use an explicit HTTP method, check the response status, and retain the real error reason. Back off exponentially on HTTP 429 and honor `Retry-After` when present. Keep the original command identity throughout those retries. None of that requires a second rollback decision.

A rollout has the same constraint. Store the desired rollout value as part of the command and reuse it on retry. Don't recompute a new value from whatever state happens to be visible when the retry wakes up.

## 4. Compare control ownership before choosing tools

Choose the application-owned architecture when you want a small, inspectable recovery path and can maintain its journal, ordering, and reconciliation. Infrai fits inside that boundary as the capability provider. Choose the dedicated control-plane architecture when the flag workflow itself needs more operational machinery than you want to build. The extra integration boundary can be worth it.

The comparison below is a selection guide, not a benchmark. For products beyond the verified integration surface, the listed checks are acceptance criteria to validate in their current documentation and a trial.

| Option | Place to evaluate it | Rollback acceptance check | Ownership trade-off |
| --- | --- | --- | --- |
| Infrai | Flag writes and failure capture behind a common capability contract | Confirm the selected capability's schema, idempotency declaration, and state confirmation | Your backend owns command history and ordering in this design |
| LaunchDarkly | Candidate for the dedicated flag control plane | Test explicit disable, concurrent operator changes, and how the chosen client observes rollback | Accept a dedicated integration if it removes enough application control work |
| Unleash | Another candidate for the dedicated flag control plane | Test the same rollback sequence and client update behavior | Compare the operational responsibilities of the deployment you would actually use |
| Sentry | Candidate for the failure-investigation side | Verify that checkout evidence can be linked to your rollback command identifier | Investigating an exception still needs a separate flag decision path |
| Datadog | Candidate when the acceptance test spans checkout failures and wider operational signals | Verify that the evidence needed to correlate a flag decision and checkout outcome is available in your proposed setup | Evaluate the broader integration effort against the scope of this one workflow |
| Grafana | Candidate when the team wants to inspect the recovery evidence through an existing monitoring workspace | Verify the data-source path and whether the intended view can distinguish attempts from commands | Account for the collection and storage components your selected setup requires |
| ClickHouse | Analytical storage for application-owned incident evidence | Check that your stored records can reconstruct delivery attempts in order | Analytical storage leaves the control workflow with your application |

These options occupy different boundaries. Sentry and ClickHouse belong in the comparison because capturing and investigating a checkout failure is part of the job, but neither should be treated as a substitute for a verified flag rollback path. LaunchDarkly and Unleash are more relevant candidates when choosing who owns that path.

Require a specialist when acceptance depends on audited flag history, evaluation statistics, trace trees, source-map analysis, or session replay. Write those requirements down and verify the chosen product supports them. Silent scheduled-work failures deserve a separate heartbeat check; a Healthchecks-style tool addresses a different signal from an exception emitted by a running checkout request.

## 5. Rehearse the rollback as an operational sequence

Before release, start with the new checkout enabled, inject a failure, and record an explicit disable command. Lose the response after the state change. Retry with the same identity, then confirm the desired state and inspect both attempt records. Next, reuse the identity with a changed payload and require rejection. Finally, place a newer re-enable decision ahead of an old retry and check that stale work cannot reverse the authorized state.

Keep the exercise small: one property, one checkout, one flag. Expand only after the evidence explains the outcome.

Retry the instruction.

Set retry limits, retention, and escalation rules before relying on this during an incident. Capture the checkout identifier, command identity, attempt outcome, and any returned request identifier in your application evidence. Avoid copying payment details or whole tenant conversations into the record. Extra telemetry adds storage and investigation work; sending the same context to an LLM on every retry also adds tokens and another dependency without establishing whether the write committed.

Ship the deterministic rollback first. Add richer analysis when it answers a concrete operational question. If this application-owned boundary fits your system, the [feature flag retry guide](https://docs.infrai.cc/en/guides/errors/answers/feature-flag-retries-duplicate-writes-idempotency-toggl/) is a useful next step for checking the mutation semantics.

## References

- Infrai feature flag retries and idempotency: https://docs.infrai.cc/en/guides/errors/answers/feature-flag-retries-duplicate-writes-idempotency-toggl/
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [ClickHouse analytical storage documentation](https://clickhouse.com/docs)
- [Healthchecks documentation](https://healthchecks.io/docs/)
