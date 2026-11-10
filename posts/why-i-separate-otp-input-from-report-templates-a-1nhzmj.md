# Why I Separate OTP Input From Report Templates — A 2FA Login Trade-off

Keep OTP verification in the authentication service and give the customer support report template ownership of presentation only. The deciding constraint is that editing an email attachment template must never change who can complete a 2FA login or receive a generated report.

TL;DR: For a Node.js application, validate OTP input on the server, bind the challenge to the pending login, and consume a successful challenge through one atomic state transition. My design choice is to keep report generation and attachment delivery behind that transition. This note proposes an implementation and its acceptance checks; it does not report a production experiment or measured performance.

## What does the report template actually own?

Consider a customer support application that generates a case report and sends it as an email attachment. A support editor should be able to revise the subject, explanatory text, and attachment label. Those are useful template responsibilities. The editor should not be able to select the authenticated account, set the OTP expiry, reset the attempt allowance, or choose the destination from text produced by a model. Keep those decisions in application code and trusted account records.

That is the ownership boundary.

The tempting implementation puts everything into one email workflow: render a login code, accept the submitted value, and attach the latest report after a match. I would reject that design at review because it gives a presentation change too much authority. A successful code comparison would say nothing about whether the selected attachment belongs to the account, and a replacement-code branch could accidentally allocate a fresh attempt allowance. The simpler workflow therefore fails the acceptance constraint before any latency benchmark matters. These are failure cases to test, not claims about a particular deployment. To make that review concrete, let a template preview specify another account's case identifier and a different destination address; require the application to ignore both fields when resolving authorization. A presentation-only preview can render those strings, but it must not turn them into delivery instructions. Otherwise the editor effectively owns the data-access policy, regardless of which team owns the authentication module.

I would choose two separately versioned templates and a server-owned authentication policy. One template presents the login challenge; the other presents the authorized report delivery. The extra ownership boundary adds release coordination, but lets a support editor fix confusing attachment copy without changing login policy. For a small team, that is a reasonable trade when generated reports may contain customer-specific information. If only engineers edit templates, the same boundary can live inside one repository. A separate service is optional.

## How should an application handle OTP input during a 2FA login?

The example below is a proposed contract for a server-issued, six-digit challenge. Six digits and five failed attempts are illustrative policy choices, not a claim that one policy fits every threat model. Authenticator-generated codes require a verifier appropriate to that mechanism; this example deliberately models a challenge issued and stored by the application.

Treat the submitted code as a string. A value such as `004281` should retain its leading zeros. Reject malformed input rather than coercing numbers, trimming unexpected characters, or accepting a partial match. The pending-login identifier must come from the server's authenticated login context, and the client must not choose the account associated with it.

```ts
type PendingLogin = {
  id: string;
};

type VerificationResult =
  | { ok: false }
  | { ok: true; sessionId: string };

interface LoginStore {
  // Atomically check binding, expiry, attempts, and the code verifier;
  // then consume the challenge and persist a single session.
  // Record a failed attempt under the same transaction or lock.
  verifyAndCreateSession(input: {
    pendingLoginId: string;
    code: string;
  }): Promise<VerificationResult>;
}

async function verifyOtp(
  input: unknown,
  pendingLogin: PendingLogin,
  store: LoginStore,
): Promise<VerificationResult> {
  if (typeof input !== "string" || !/^[0-9]{6}$/.test(input)) {
    return { ok: false };
  }

  return store.verifyAndCreateSession({
    pendingLoginId: pendingLogin.id,
    code: input,
  });
}
```

The interface is the hard part of this example. Its implementation must use a transaction or equivalent atomic operation, including across application workers. A separate read, comparison, and delete sequence does not satisfy this contract: two requests could both observe an unused challenge. Never replace the interface with an in-memory map and assume the example has become a complete authentication system.

Specify the storage behavior before wiring the handler. Use the server's clock for expiry. Bind the stored verifier to its challenge and pending account, use a protected verifier representation suitable for a small code space, and use an appropriate constant-time comparison when comparing secret-derived byte values. Keep verification secrets outside the template configuration. Do not store a short code as an unprotected reusable credential or put it in a log.

For the proposed five-attempt policy, the fifth failed comparison should exhaust the challenge, and subsequent submissions should fail even if the code matches. Apply request limits before this function as well: malformed requests must not become an unlimited path through the endpoint. Define replacement-code behavior explicitly, including invalidation of older challenges and a broader limit that cannot be refreshed simply by asking for another code.

One success only.

## The attachment needs its own authorization check

A completed login establishes a session; the report-delivery operation still needs to authorize access to the requested report. Resolve the report owner and recipient using trusted application records at delivery time. An attachment filename, model-generated summary, or hidden form field is not an authority to send customer data.

For an illustrative fixture, use case `SUP-204`, report revision `3`, and template revision `7`. Those labels are example data. Keep them separate so the delivery worker can identify exactly which authorized artifact it is sending after a template release. If the report is regenerated, make the revision change explicit instead of silently substituting a different attachment into an existing queued job.

This is where the choice costs something.

The trade-off is retention work: freezing a report revision and template revision makes the job less flexible, and storing the artifact requires a deletion policy. In exchange, a retry can reuse the approved attachment instead of invoking the model again. That is the token-cost decision I would make first: define what can be reused, then measure generation frequency and storage needs. There is no useful universal saving percentage to attach to that decision.

Caching has limits.

This approach is not suitable for reports whose authorization or contents must be freshly evaluated on every delivery. For those reports, rebuild the artifact after a new authorization check and accept the additional generation work. A frozen revision cannot stand in for current permission.

Give each logical delivery a persistent identifier and record its progress. Handle a timeout as an uncertain result requiring reconciliation, rather than immediately assuming the email was never accepted and sending another copy. Keep attachment delivery out of the login request's critical path; the login result should not wait for report generation or mail transmission.

## Sender authentication cannot verify a login code

SPF specifies how a receiving system can evaluate whether a sending host is authorized to use a domain in the SMTP envelope identity. It does not define OTP verification, bind a challenge to an application session, or authorize a report attachment. Its result cannot replace the application checks above.

Keep mail-sending configuration and login acceptance criteria in separate review checklists. A correctly configured sender identity is useful delivery infrastructure; the application still owns the challenge lifecycle and customer-data authorization.

## What to measure before copying this choice

Start with a small acceptance exercise. Submit the same valid challenge concurrently from two requests and require exactly one session creation. Submit an expired challenge, use one pending login's challenge against another, exhaust the five-attempt allowance, and request a replacement before trying the old code. Test the expiry boundary with a controlled server clock. These tests check the storage contract, not just the input regex.

Then edit the report template and verify that none of those outcomes changes. Try an authorized session against another account's report and require denial before generation or delivery. Simulate a worker restart and an uncertain send outcome. The system should preserve the chosen report revision and expose enough state to reconcile the delivery without silently generating a new artifact.

For deployment, review authentication policy changes separately from copy changes and retain explicit template revisions. Record outcome categories, exhausted challenges, delivery retries, generation counts, and time spent waiting for each stage. Keep submitted codes and attachment contents out of telemetry; use restricted correlation identifiers when tracing a case across the workflow. Measure authentication latency separately from report preparation so a slow generation step cannot disappear inside a single average.

My decision rule is conditional: keep templates responsible for wording whenever support staff can edit the delivery experience, and keep login policy and attachment authorization under application ownership. Before adopting the design, prove the concurrency behavior and measure the cost of generation retries. Those results determine whether the boundary earns its operational overhead.

## References

- RFC 7208: Sender Policy Framework (SPF), for the scope of sender authorization in email: https://datatracker.ietf.org/doc/html/rfc7208
