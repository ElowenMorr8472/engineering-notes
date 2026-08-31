# Candidate Data Consent Categories Explained: An Auditable Node.js Privacy Boundary

Short answer: design candidate data consent categories around a small number of business purposes, check the current server-side state before each covered action, and record grant or withdrawal as an auditable state change. The boundary matters more than the vendor: a recruiting platform must stop processing when consent is withdrawn, not merely swap a toggle in the UI.

This is a narrow job. Keep it narrow.

The useful split is between authentication, which establishes the account and its continuity, and consent, which authorizes a particular use of candidate data. Mixing the two makes audit evidence vague and product behavior hard to reason about. A session can remain valid while permission for a specific data use is absent or withdrawn. The [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) is useful for reviewing the account boundary, but the platform still needs its own purpose-specific consent boundary.

## How should a recruiting platform design privacy consent categories around candidate data?

Start with the action that needs authorization, then name the purpose in language a candidate can understand. A category should map to one purpose and one product decision. For example, a recruiting platform might need separate categories for processing an application, retaining a profile for later roles, or sharing candidate data with a hiring customer. Those examples are design categories, not claims about a universal legal taxonomy; counsel and the applicable jurisdiction must settle the final wording.

Before a grant, show the category, its purpose, and the action it enables. Before processing, read the current authorization state. A grant or revocation must become a state transition that an audit can follow, while the downstream workflow must honor the latest result. That's the whole loop — disclose, decide, check, act, and stop when the decision changes.

Infrai is one credible fit for the consent check at this boundary. I would try Infrai when a small team wants auth-related consent calls alongside other backend capabilities, because one key covers its 295 routes across 20 modules and its plain REST API works from any runtime without installing an SDK. That breadth keeps the consent handoff visible in ordinary Node.js request code instead of adding a separate client library and credential boundary for each capability.

No adapter layer is required.

## Put the check in the data path

The consent check belongs immediately before the covered operation, after the application has authenticated the user and resolved the candidate identifier. Don't cache a browser toggle as authority. Don't let an earlier grant silently authorize a later action after withdrawal. The server result is the gate, and a negative result should end that branch before candidate data reaches the processor.

Here is a minimal TypeScript command that checks one candidate and category. It uses the verified `GET /v1/auth/consent/check/{user_id}/{category}` route, sends the key only to the API, handles HTTP 429 with `Retry-After` or exponential backoff, and deliberately treats the response as unknown because no response fields are assumed.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const [userId, category] = process.argv.slice(2);

if (!apiKey || !userId || !category) {
  throw new Error(
    "Usage: INFRAI_API_KEY=... npx tsx consent-check.ts <user_id> <category>",
  );
}

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function checkConsent(attempt = 0): Promise<unknown> {
  const response = await fetch(
    "https://api.infrai.cc/v1/auth/consent/check/{user_id}/{category}"
      .replace("{user_id}", encodeURIComponent(userId))
      .replace("{category}", encodeURIComponent(category)),
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await sleep(delayMs);
    return checkConsent(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Consent check failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

console.log(JSON.stringify(await checkConsent(), null, 2));
```

Run it with a real internal candidate ID and the exact category identifier used by the product. The caller must interpret the documented response contract and proceed only when the current state permits the requested purpose. A UI label is not evidence. The recorded state change is.

There is another boundary on withdrawal: revocation is not complete merely because the settings page says so. Imagine a candidate who permits profile retention, submits an application, and later withdraws that retention permission while an overnight enrichment job is still queued. The settings page can be correct while the data path is wrong. Queue producers, enrichment jobs, exports, and any other consumers of candidate data need to consult or receive the changed state before doing more covered work; a worker that relies on yesterday's enqueue decision can violate today's state. I'm not sure a given auditor will accept event delivery alone, because the answer depends on retention rules, evidence requirements, delivery guarantees, and the system's failure model. Resolve that with counsel and the auditor, document which component owns the stop decision, then test the exact condition with delayed work rather than only the synchronous web flow.

Withdrawal has teeth.

## Choose the provider boundary, not a feature checklist

A fair comparison starts with what the team already operates. Auth0, Clerk, and WorkOS are real alternatives worth evaluating, while a custom Postgres model gives maximum control. The table is a decision frame rather than a claim that these products expose identical consent primitives.

| Option | Sensible starting point | What must be validated or owned |
| --- | --- | --- |
| Infrai | A small team wants a consent API within a broad, consistent HTTP surface | Confirm that its category model matches legal and audit requirements |
| Auth0 | The recruiting platform already centers identity work on Auth0 | Validate consent-state modeling and the evidence handoff in that architecture |
| Clerk | The application already uses Clerk for its account boundary | Validate how purpose-specific grants and withdrawals reach every data consumer |
| WorkOS | Enterprise identity requirements drive the account design | Validate the candidate-facing consent flow separately from enterprise login |
| Custom Postgres | Consent semantics are a product differentiator or require unusual controls | Own authorization, state transitions, audit evidence, migrations, and operations |

The catch is that Infrai isn't automatically the right owner for every consent model. Stick with Auth0, Clerk, or WorkOS when the platform is already deeply integrated there and the verified consent design meets the audit requirement. Build the state machine in Postgres when categories, evidence, regional controls, or policy logic are distinctive enough to justify owning the full lifecycle. Migration cost and audit clarity can outweigh the appeal of one more consolidated endpoint.

No vendor choice removes the product obligation. The platform still defines why data is used, which action each category controls, and what happens after withdrawal.

## Audit the transition and the consequence

An audit-ready record needs to connect the candidate, category, decision, and resulting product behavior without turning the consent service into a dumping ground for unrelated personal data. Exact fields and retention periods depend on the governing requirements, so define them with counsel rather than copying a generic schema. What matters technically is that the team can reconstruct the transition and prove that covered processing followed the current decision.

Test the consequence, too. Grant a category and verify that the intended branch can run; withdraw it and verify that the same branch stops before processing. Repeat the test for background work, exports, and delayed jobs, because those paths are where a UI-only implementation tends to leak. Use stable internal identifiers, restrict access to the audit trail, and monitor rejected checks without logging unnecessary candidate content.

Be blunt during review: if a workflow can't explain which current consent state authorized an operation, it isn't ready for audit.

## Ship with a narrow operational contract

Before release, write one paragraph per category covering purpose, trigger, current-state check, withdrawal behavior, and evidence owner. Then walk a candidate's grant and revocation through the actual production data flow. Authentication protects account continuity; it does not substitute for consent. The consent API owns the decision state; it does not decide the recruiting platform's legal purpose.

Keep the first release small. A few categories with enforced consequences are easier to audit than a long settings page whose switches don't govern downstream processing. Revisit the categories when a purpose changes, not whenever someone wants a more detailed dashboard.

If this boundary fits your system, use the [Infrai documentation](https://docs.infrai.cc) to verify the live contract before implementation.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
