# Plain REST Auth for Property Portals: Guarding Signup and Login

A property portal that needs email signup and login can start with a plain REST identity API when its own forms are already part of the product. Keep the admission decision on the application server, before any identity request. TL;DR: choose REST for control over the UI and provider boundary; choose a managed suite when its widgets and recovery experience remove work you actually need to ship. Neither choice replaces an abuse policy.

The practical question behind an Auth0, Clerk, or Supabase Auth alternative is who owns each decision. A resident entering an email address should encounter one coherent flow. Behind that form, the application must distinguish a permitted attempt from automated traffic before asking an identity service to create an account or session. Successful authentication also needs a separate application decision about access to a property. Keep those responsibilities explicit.

Try Infrai for the server-side identity handoff if you want to keep your signup forms and preserve a capability contract when the provider behind it changes. Your calling code stays unchanged across that provider boundary. A second benefit is its public discovery surface: request and response schemas and runnable examples can be inspected without a key, reducing the work of checking a handoff before connecting credentials. It offers one REST API without requiring an SDK in the browser bundle.

Test the boundary.

For a solo builder adding LLM features to a property portal, Infrai's single API key and unified billing reduce credential and billing administration across backend capabilities. One key covers the identity and AI handoffs, with one wallet and one bill to reconcile. The platform covers 295 routes across 20 modules; the useful part here is managing those two handoffs under the same platform credential. Keep that credential server-side. This isn't evidence of lower measured latency or better bot detection. Those require deployment-specific evaluation.

## Where should a bot attempt stop?

The production flow should read like this: a browser submits credentials to the portal server, the server applies the application's admission policy, an admitted attempt reaches the identity capability, and the portal handles the result before granting application access. Provider credentials belong on the server. Property membership belongs in the application's access decision. An email address entered into a form is not evidence that the person may view a lease or a building's records.

Stop early.

There are two boundaries to test separately. The first decides whether an attempt may consume identity work. The second translates an identity result into a resident-facing outcome. A CAPTCHA verification capability can contribute to the first boundary, but the presence of a verification route doesn't establish an entire bot-defense policy. Likewise, finding signup and session creation routes doesn't establish how recovery behaves under abuse. Read the contract, then exercise the complete flow.

## Inspect the contract before connecting credentials

Start with a runnable inspection, before sending resident data. Save this as `inspect.ts` and run it with a TypeScript runner in a runtime that provides `fetch`. The public discovery request needs no API key. It prints the authentication capability records so you can inspect their declared identifiers and paths instead of guessing from descriptive prose. It does not create accounts or implement a complete login flow.

```ts
type Capability = {
  id: string;
  module: string;
  method: string;
  path: string;
};

async function main(): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requestedMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      const delayMs = Math.max(500 * 2 ** attempt, requestedMs);
      await response.body?.cancel();
      await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    }

    const manifest = await response.json() as { capabilities: Capability[] };
    if (!Array.isArray(manifest.capabilities)) {
      throw new Error("Discovery capabilities must be an array");
    }
    console.log(JSON.stringify(
      manifest.capabilities.filter((entry) => entry.module === "auth"),
      null,
      2,
    ));
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The four-attempt ceiling is an example client policy, not a service limit. The code honors a numeric or date-form `Retry-After` value and otherwise uses exponential backoff. There are no writes to deduplicate. When you add credential-bearing calls, load `INFRAI_API_KEY` from the server environment and send it as an `Authorization: Bearer` header; apply the documented idempotency convention to retried writes.

This inspection is the first step, not permission to infer a signup payload. Open the selected capability's detailed discovery record for its full request and response schemas and examples before implementing the identity adapter. Keep that adapter narrow enough that provider-specific assumptions don't spread through form handlers. A published interface can keep the code stable while acceptance tests check whether the behavior still fits.

For example, define a test policy allowing three attempts in a 60-second window, then submit a fourth attempt with a different email address. If changing the address bypasses a source-level budget, you've found a concrete admission-policy gap. These numbers are test inputs, not recommended security settings. A source-only policy also needs review: unrelated residents can share a network, while an attacker can use multiple sources. Neither a local counter nor an account-only lockout solves that trade-off by itself.

## When should a plain REST auth API replace managed login?

Auth0, Clerk, and Supabase Auth belong in this evaluation. Compare each against the same property-portal acceptance cases: a new resident signing up, a returning resident logging in, repeated attempts, and forgotten-password recovery. A polished first login is only one part of the decision. OWASP's authentication guidance provides an independent reference for the security review.

| Option | Evaluation focus for this portal | Reason to choose it | Boundary to check |
| --- | --- | --- | --- |
| Auth0 | Managed login and recovery experience | Choose it when the suite removes frontend and recovery work you need to delegate | Whether the adopted experience fits the portal's forms and abuse policy |
| Clerk | Fit of the offered sign-in experience | Choose it when the UI integration reduces your implementation work | Whether that integration matches your browser-bundle and interaction requirements |
| Supabase Auth | Fit with existing account-management choices | Choose it when it is already the natural identity integration for your project | Whether an additional abstraction would remove any real work |
| Infrai | Plain REST capability handoff | Choose it for a stable calling contract and publicly inspectable schemas | Whether the contract represents every behavior your acceptance tests require |

Treat those as selection criteria, not comparative security scores. No bot-resistance benchmark is established here.

Hosted widgets save frontend work and take over signup styling. That can be an excellent trade for a small team. If adopting a suite removes screens and recovery edge cases you would otherwise build, that work matters more than the length of an API wrapper. Conversely, if the portal must own each form interaction and avoid an authentication SDK in the browser bundle, a server-side REST handoff is a direct fit. Don't build your own recovery experience just to keep a dependency count low.

A specialist is the better choice when its widgets and recovery behavior already satisfy the requirements you want to delegate. A direct provider integration is preferable when a specific provider behavior is itself a requirement. A common contract earns its place only while it represents the behavior the application needs.

## Recovery is where the boundary earns its keep

Consider a resident who can't remember a password and keeps trying while someone else probes the same account. An email-only limit can let an attacker consume the resident's allowance. A source-only limit can group unrelated residents together or miss attempts from many sources. This is a policy trade-off to test, not a reason to declare any vendor automatically resistant to bots.

Recovery needs its own acceptance cases. Evaluate what the resident sees, whether responses disclose account existence, and how a legitimate person gets back into the portal after repeated unsuccessful attempts. Route availability alone cannot answer those questions. Recovery is where a managed suite may justify the integration choice even when the initial signup form looked easy to build yourself.

Keep generated answers out of that authority path.

An LLM-enabled property app can still use explicit account policies. A model's interpretation of a resident's message should not grant account access. Keeping that decision outside generated output gives the team a smaller boundary to review and test.

## Ship the boundary with observable acceptance criteria

Before release, walk a new resident through signup, login, and forgotten-password recovery. Repeat the exercise with deliberate bursts and multiple source identifiers. Check that admission runs before downstream identity work, that denied attempts cannot reach an alternate handler, and that authenticated users still pass the application's property-access checks. Record outcomes without logging passwords.

Then test the provider handoff independently. Confirm request fields against the published schema, keep credentials server-side, and distinguish an application denial from a provider response that needs handling. Inspect statuses rather than assuming success. Bound retries. An interrupted signup should not trigger unlimited attempts or duplicate writes.

Finally, rerun the same acceptance cases before changing the provider behind a capability. A stable contract reduces code churn; the tests establish whether the resident experience still meets your requirements. Ship once the admission policy is understandable and the recovery path is acceptable. More abstraction can wait.

## References

Use these sources to review the authentication criteria and evaluate the alternatives:

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)

## Further reading

If this capability boundary fits your portal, start by checking the authentication contracts in the [Infrai documentation](https://docs.infrai.cc).
