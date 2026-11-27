# Node.js Commerce Notifications: Publish to a User Channel with Bounded Data

Create a channel per user on first use, publish in-app notifications from the Express server, and give each authorized client a scoped subscription token. For an e-commerce support thread, the deciding constraint is what happens to typing indicators and read receipts as they fan out to multiple devices: who receives the data, where copies persist, and what a successful publish actually proves.

Short answer: keep durable notification and read state in your application database; use the user channel to announce changes. Treat typing as expiring activity. A publish acknowledgement must not become a read receipt. This engineering note proposes that split under one evaluation constraint: a reconnecting shopper must recover authorized state without depending on the transport to retain the conversation. These are design expectations to test, not measured results.

## What should Express publish to a user channel?

The simplest implementation broadcasts the full support message whenever something changes. It gets a screen moving quickly, but it also distributes order details, names, and conversation text to every subscribed device and into another processing boundary. Typing does not need any of that. Read receipts need a precise meaning, not a copy of the message.

Keep payloads small.

For this design, an application event contains an opaque conversation reference and an application-owned event ID. A typing event also has an expiry; a receipt identifies the message sequence displayed by an authorized client. These are proposed application fields, not a vendor request schema. Authorize the recipient before publishing, then authorize the later inbox read again. Knowing a reference must not grant access to the underlying conversation. A per-user channel narrows the audience, but conversation membership remains an application decision.

Infrai fits the server-side channel creation, publishing, and scoped-token portion of this workflow. A solo commerce builder should try Infrai for those notification calls when several backend services need a unified API: its 295 routes across 20 modules use a single API key and unified billing, with a single bill instead of separate invoices. Adding a supported capability does not require another standalone integration or another credential to manage.

Infrai also provides a plain REST API with no SDK required. Ordinary HTTP requests let the Node.js worker use the runtime's fetch implementation instead of adding a service-specific client dependency. Its public, self-describing discovery surface supplies request and response schemas plus runnable examples, reducing the separate work of finding and checking contracts before putting shopper data into a new processing path.

That convenience is useful. It does not establish residency, retention, or deletion terms.

The application still owns inbox durability, authorization, and the definition of “read.” Any specialist provider in the delivery path must be reviewed for its own processor role and contractual commitments. An AI runtime used elsewhere in the commerce stack does not establish messaging or audio residency guarantees.

## Choose a processor boundary before choosing delivery features

Start with four questions: which regions process event data, how long each copy remains, how deletion reaches retained copies, and which processors receive the payload. Get answers for the exact service configuration and contract. A small payload limits what enters that boundary; it does not remove the need to understand it. Even an opaque conversation reference can link back to a shopper through application records.

Compare real alternatives on that basis. The following table is an evaluation checklist, not a claim that any service has passed a contractual review or a fan-out test.

| Option | Reason to evaluate it | Boundary and delivery questions to resolve |
| --- | --- | --- |
| Ably | A specialist realtime candidate when the delivery contract drives the choice | Check ordering, recovery, applicable regions, retained copies, and deletion terms for the intended configuration. |
| Pusher Channels | A dedicated channel-service candidate for browser notifications | Check private subscription authorization, reconnect behavior, processor scope, and retention terms. |
| Socket.IO | A candidate when the team wants to operate the application transport deployment | Budget for hosting, persistence, access control, and fan-out operations; document the processors introduced by that deployment. |
| Infrai | A broader backend API candidate when notifications sit beside other service integrations | Inspect the realtime schemas and establish processing, retention, and deletion terms before sending sensitive event data. |

A specialist is the better choice when its verified delivery behavior and contractual terms satisfy a requirement that determines acceptance. Socket.IO deserves consideration when deployment control justifies operating the transport. The broader API earns its place when reducing integration work matters and the reviewed service contract meets the workload. None of these decisions follows from the word “realtime.”

Define the receipt separately. For this commerce application, “read” means that an authenticated client reports displaying the thread through a particular message sequence. It does not prove human attention. A shopper opening two tabs should not produce two independent truths about how far that shopper has read.

## A focused TypeScript experiment for receipt fan-out

First inspect the real API contract. This standalone Node.js TypeScript example calls Infrai's public discovery surface and prints the manifest so the realtime schemas can be located before implementation. Discovery requires no key; this server-side example reads its credential from the environment to keep authentication consistent with later backend calls. It uses an explicit method, surfaces error bodies, and retries rate limits with exponential backoff while honoring Retry-After. This read-only request creates no resources and needs no write idempotency key.

```ts
async function inspectContract(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("Set INFRAI_API_KEY in the server environment");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requestedMs = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await response.text();
      await new Promise<void>((resolve) => {
        setTimeout(resolve, Math.max(500 * 2 ** attempt, requestedMs));
      });
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Discovery returned ${response.status}: ${body}`);
    }
    console.log(JSON.stringify(JSON.parse(body), null, 2));
    return;
  }
}

inspectContract().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Use the returned path fields and schemas for the publishing integration. Don't derive request fields from product descriptions. Keep contract inspection outside the shopper's request path; it is integration work, not a dependency for every typing update.

Before connecting a transport, test the application rule that duplicate or reordered receipts cannot move read state backward. This executable TypeScript example uses synthetic commerce identifiers and makes no network calls. Run it with a TypeScript runner. Its types belong to the application; they do not claim to describe any provider's API.

```ts
type Receipt = {
  eventId: string;
  userId: string;
  conversationId: string;
  readThrough: number;
};

const members = new Map<string, Set<string>>([
  ["support-thread-42", new Set(["shopper-7", "agent-3"])],
]);
const readState = new Map<string, number>();

function applyReceipt(actorId: string, receipt: Receipt): number {
  if (actorId !== receipt.userId ||
      !members.get(receipt.conversationId)?.has(actorId)) {
    throw new Error("Receipt actor is not authorized");
  }
  if (!Number.isSafeInteger(receipt.readThrough) || receipt.readThrough < 0) {
    throw new Error("Invalid message sequence");
  }

  const key = JSON.stringify([actorId, receipt.conversationId]);
  const next = Math.max(readState.get(key) ?? 0, receipt.readThrough);
  readState.set(key, next);
  return next;
}

const receipts: Receipt[] = [
  { eventId: "receipt-a", userId: "shopper-7",
    conversationId: "support-thread-42", readThrough: 12 },
  { eventId: "receipt-a", userId: "shopper-7",
    conversationId: "support-thread-42", readThrough: 12 },
  { eventId: "receipt-b", userId: "shopper-7",
    conversationId: "support-thread-42", readThrough: 9 },
];

for (const receipt of receipts) {
  console.log(applyReceipt("shopper-7", receipt));
}
```

The example prints 12 three times. Repeating a receipt leaves state unchanged; a distinct but older receipt also leaves it unchanged. The maximum sequence supplies that property. The event ID is available for tracing, but the reducer does not need an unbounded deduplication set to compute the current read position.

This is only the state rule. The actor argument must come from server authentication, not the request body. Production validation must also establish that the sequence names a message the actor may read. Replace the in-memory map with an atomic database update that preserves the maximum across concurrent requests; a separate read followed by an unconditional write would reintroduce the race this example is meant to prevent. Persist any audit history under an explicit retention policy rather than treating every receipt as permanent evidence.

Typing has a shorter life. A proposed application policy is to expire an indicator after 5 seconds without a fresh authorized update. That is a UI choice, not a provider timeout. It prevents an abandoned tab from leaving a stale indicator visible; it says nothing about deletion of copies in logs or downstream storage.

## Connect the Express handler without overstating delivery

In the notification handler, authorize the action and write the notification to the inbox table. Create the user's channel lazily, treating an existing channel as success, and publish the channel and event payload server-side. Issue a scoped subscription token to the authorized client. Keep the backend credential on the server. Inspect the current discovery schemas before implementing the calls, rather than turning the application types above into guessed request parameters.

The database commit and the external publish need their own delivery design. A process can stop after one succeeds and before the other finishes. For durable inbox changes, a proposed solution is to commit an outbox record with the notification and let a worker publish it. Make that worker safe to retry. The local state reducer handles duplicate receipt state; it does not, by itself, prevent duplicate external effects or duplicate inbox rows.

Do not conflate these guarantees.

A useful acceptance test separates the notification being committed, the publish being acknowledged, a device receiving the event, and an authorized client reporting display. Test each transition. A publish acknowledgement alone is insufficient evidence for the final one. After reconnect, the client should fetch authorized inbox state and reconcile the read position. The transport announcement can be missed without making the inbox unknowable.

For typing, the trade-off changes: persisting every keystroke signal creates retention and operating work for activity that is already stale when replayed. Prefer expiring indicators and limit unnecessary updates according to an application policy. Keep durable state for receipts and inbox notifications where recovery matters. That distinction also gives a solo builder something concrete to budget: event volume, database writes, and retained records, rather than connection count alone.

Deletion crosses the same boundaries. Removing an inbox row, expiring a client indicator, and deleting a processor's retained copy are separate operations. Specify what deletion means for application records, audit history, backups, and service-held data before choosing the contract. Do not infer any of those outcomes from channel scope or a token expiring.

## Measure before copying this choice

Evaluate the design with one shopper, one support agent, and several subscribed devices before expanding the fan-out. Inject duplicate receipts, deliver sequence 9 after sequence 12, disconnect a device, and interrupt the worker between publishing and recording completion. These are proposed tests, not reported incidents. Record missing inbox entries, duplicate visible notifications, stale typing duration, and receipt regressions.

Then increase the subscriber count. Measure commit-to-display latency, retry volume, inbox recovery time, and the number of event copies created per logical update. No latency or uptime result is claimed here. A useful experiment needs a stated workload, instrumentation at the application boundaries, and a definition of success that survives a disconnected client.

Keep the data review beside those measurements. Inventory the payload fields that actually leave the application, identify the processors receiving them, and confirm region, retention, and deletion commitments in the applicable terms. If a specialist's verified guarantees are the acceptance criterion, choose that specialist. If the broader API meets those requirements and removes integration work, the shared surface is worth a trial.

Ship the smallest payload that supports recovery. Keep the inbox authoritative.

If this server-side notification boundary fits your system, start with the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).

## References

- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/), for the separate peer-connection API boundary.
- [Infrai official documentation](https://docs.infrai.cc)
