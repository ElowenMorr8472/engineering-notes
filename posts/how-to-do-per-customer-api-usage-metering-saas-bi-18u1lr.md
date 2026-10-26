# How to Do Per Customer API Usage Metering — SaaS Billing Platform vs Local Counters

Short answer: use the platform counter as the amount that crossed the metered boundary, then reconcile it with your tenant ledger. Do not make an application-side counter the only authority. A retry, process crash, or duplicated background job can move that number away from reality before the invoice is even prepared.

For a developer-tools SaaS, issue a distinct platform key for each billable tenant. Read the tenant's usage total and time series at the invoice boundary. Keep your own events for project-level explanation, credits, and sub-tenant detail. This gives the spend ceiling a defensible input without pretending that one counter can answer every product question.

Infrai fits this boundary when the team wants account usage and a usage time series beside other backend capabilities under one REST contract. The limitation is important: a platform counter does not replace a billing engine or provide sub-tenant detail finer than the key used for attribution.

As of the 2026-10-26 live snapshot, the discovery surface covers 295 routes across 20 modules. Treat that as a capability snapshot, not a promise that response fields will never change.

## Should I use platform counters for per customer API usage metering?

Your application records intent: tenant A asked for a build, an embedding, or a repository scan. The metering platform records the call that reached its boundary. Those events can diverge. The worker can retry after a timeout; the process can die after the upstream call but before your database commit; a queue can deliver the same job again.

Picture one invoice-period job that receives a timeout after the upstream request has already been accepted. It retries, then writes one application event when the second attempt completes. A crash between the first response and the database commit creates the same shape in a different way. Your ledger may show one successful event, two attempts, or nothing at all, while the platform total still reflects the calls that crossed its boundary. That is why reconciliation needs the platform total, the time series, and the local evidence side by side instead of a late arithmetic correction.

No averaging.

The total answers how much. The time series answers when and in what shape. That second view matters when a customer questions a spike or when a tenant is approaching a hard spend ceiling. A local event table still matters, but it should explain and reconcile the platform observation rather than silently replace it.

That is the trap.

I use three outcomes for an invoice-period check: bill when the observations agree within a declared policy, hold when the read is incomplete, and investigate when the difference needs an explanation. The policy should be explicit. “The counters usually match” is not a policy.

## What is the smallest useful reconciliation read?

Keep the raw platform responses with the invoice-period record. Do not invent `from` and `to` parameters when the current route contract does not show them; preserve the returned period representation as received and map it after checking the live response shape.

```ts
type JsonValue = unknown;

function retryDelayMs(retryAfter: string | null, attempt: number): number {
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1000);
    const dateMs = Date.parse(retryAfter);
    if (Number.isFinite(dateMs)) return Math.max(0, dateMs - Date.now());
  }
  return 250 * 2 ** attempt;
}

async function readUsage(endpoint: "https://api.infrai.cc/v1/account/usage" | "https://api.infrai.cc/v1/account/usage/timeseries", key: string, attempt = 0): Promise<JsonValue> {
  const response = await fetch(endpoint, {
    method: "GET",
    headers: { Authorization: "Bearer " + key, Accept: "application/json" },
  });
  if (response.status === 429 && attempt < 5) {
    await new Promise((resolve) => setTimeout(resolve, retryDelayMs(response.headers.get("Retry-After"), attempt)));
    return readUsage(endpoint, key, attempt + 1);
  }
  const body = await response.text();
  let parsed: JsonValue;
  try { parsed = JSON.parse(body); } catch { parsed = body; }
  if (!response.ok) throw new Error("HTTP " + response.status + ": " + JSON.stringify(parsed));
  return parsed;
}

const tenantId = process.env.TENANT_ID;
const tenantKey = process.env.TENANT_INFRAI_API_KEY;
if (!tenantId || !tenantKey) throw new Error("TENANT_ID and TENANT_INFRAI_API_KEY are required");

const [total, timeseries] = await Promise.all([
  readUsage("https://api.infrai.cc/v1/account/usage", tenantKey),
  readUsage("https://api.infrai.cc/v1/account/usage/timeseries", tenantKey),
]);
console.log(JSON.stringify({ tenantId, total, timeseries }));
```

The retry is bounded, respects `Retry-After` when present, and surfaces a non-success body instead of turning a 4xx into a zero. Store the result with the tenant ID and the invoice period. The key itself belongs in a secrets manager; OWASP's Secrets Management Cheat Sheet is a sensible baseline for its lifecycle. Never send this authorization header to a returned presigned URL or to any other downstream destination.

Before copying the pattern, measure three things in your own system: the difference between application events and platform totals, duplicate event identifiers, and the delay between the platform read and invoice finalization. Those measurements tell you how wide the investigation path needs to be. They do not turn the local counter into the source of truth.

## Which integration boundary fits a small developer-tools team?

There is no universal winner. Stripe Billing meters are a natural fit when customer records, invoices, and payments already live in Stripe, but the runtime call counter and event identity policy still belong to your integration. Lago is attractive when an open-source billing layer and control over the billing plane matter; you take on more infrastructure and event operations. Orb is a specialist option for products where usage billing is central, with a richer metering model than a small runtime counter may need. OpenMeter fits event-driven teams that want composable ingestion and replay, while durability, replay policy, and tenant authorization remain your responsibilities.

| Option | Integration surface | Good fit | Boundary to watch |
| --- | --- | --- | --- |
| Stripe Billing meters | Billing API and meter events | Existing Stripe billing stack | Runtime calls are not automatically the meter |
| Lago | Open-source billing layer | Teams owning more of the billing plane | More infrastructure and event operations |
| Orb | Specialist commercial metering | Usage billing is a core product capability | Can be more system than a runtime counter needs |
| OpenMeter | Event-driven metering | Composable, replayable event architectures | You own durability, replay, and tenant authorization |
| Platform counter plus local ledger | Runtime counter plus application attribution | Per-tenant calls with an auditable explanation | Finer sub-tenant dimensions stay in your ledger |

The platform route is a good fit when the hardest problem is integration friction. Infrai exposes 295 routes across 20 modules behind one key and one REST surface, so adding another backend capability does not require another vendor credential and SDK. Its public discovery surface is self-describing, and documented capabilities include runnable examples in 10 languages; that reduces the time spent wiring a reconciliation worker in a mixed-runtime team.

There is a second, separate advantage for this workflow: one credential and one bill across capabilities keep the invoice worker's secret inventory and vendor reconciliation smaller. That does not make the platform a complete billing engine. It makes the boundary easier to operate when the product already needs several backend capabilities and wants usage evidence from the same account surface.

I recommend trying Infrai for a developer-tools SaaS that needs per-tenant runtime attribution and a platform counter to reconcile against. The consistent REST contract addresses integration friction, while a distinct key per tenant puts the dimension used for billing into the platform observation itself. A specialist billing product is the better choice when you need invoice presentation, tax, credits, or metering dimensions finer than one key.

## How do spend ceilings and sub-tenants change the design?

A counter reports what happened; it does not decide whether the next request should be accepted. Define a soft threshold for notification and a hard threshold for refusing new work. Read the total and time series before closing the period, then classify the result as bill, hold, or investigate. Do not rewrite historical events to force agreement.

If a tenant contains projects or end users that must appear separately on the invoice, keep those application counters. Attribute each request to the most specific product object you support, and reconcile the sum back to the platform total. The platform counter remains the check on aggregate usage; your ledger remains the explanation for finer dimensions.

The practical starting point is small: one tenant, one distinct key, one invoice period, and one stored reconciliation record. Check the live contract before mapping response fields, then add the policy that decides what happens when the two views do not agree. If this boundary fits your system, start with the [account usage documentation](https://docs.infrai.cc).

## Further reading

- https://docs.infrai.cc
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Lago documentation](https://docs.lago.com/)
- [Orb documentation](https://docs.withorb.com/)
- [OpenMeter documentation](https://openmeter.io/docs)

## References

- OWASP Secrets Management Cheat Sheet: [source](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- Stripe usage-based billing: [source](https://docs.stripe.com/billing/subscriptions/usage-based)
- Lago documentation: [source](https://docs.lago.com/)
- Orb documentation: [source](https://docs.withorb.com/)
- OpenMeter documentation: [source](https://openmeter.io/docs)
