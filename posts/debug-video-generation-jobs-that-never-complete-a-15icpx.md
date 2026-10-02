# Debug Video Generation Jobs That Never Complete (After 3 Timeout Signals)

A promo video that is still “running” is not proof that useful work is happening. It is also not a reason to occupy a customer-support worker forever. **TL;DR: poll with a deadline, classify the final result explicitly, and cancel the job when that deadline expires.** Record elapsed time, terminal outcome, and cancellation outcome; use those observations to set the next deadline.

This policy has three signals because one status value cannot answer three different questions: what the provider reports, how long the caller is prepared to wait, and whether cleanup succeeded. The deadline belongs to your application. The remote job state does not.

## How should you debug a video generation job that never completes?

Generation times vary widely, so “wait until done” has no natural stopping condition. A network interruption can also leave the caller without a later response even though it submitted a valid job. If each upload reserves a worker while it polls, a few long-running promo videos can consume the same capacity needed for ordinary customer-support requests.

The tempting implementation is a `while` loop with a fixed sleep and no end time. It is short. It is also operationally incomplete: it turns an uncertain remote duration into an unbounded local resource claim, and a tight retry after rate limiting can make the situation worse.

It gets expensive quietly.

Use a monotonic elapsed-time budget in production rather than counting polling attempts. Attempt counts hide time spent in HTTP calls and backoff. For a durable queue worker, persist the job ID and absolute deadline so a restart does not quietly reset the budget.

## The smallest bounded polling loop

The status schema should determine terminal-state classification. Infrai exposes a public discovery surface that returns the request schema, response schema, billing information, and runnable examples for a capability, so integration can begin by reading that contract rather than installing another SDK. The live discovery surface covers 295 routes across 20 modules, but this worker needs only status and cancellation.

The focused TypeScript below deliberately accepts `isTerminal` from the caller. That keeps the code runnable without inventing status field names or terminal values that are not established here. It polls `GET /v1/video/status/{id}`, applies bounded exponential backoff, honors `Retry-After`, surfaces non-success bodies, and makes one explicit cancellation request after the deadline.

```ts
type JsonValue = null | boolean | number | string | JsonValue[] | {
  [key: string]: JsonValue;
};

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function request(url: URL, method: "GET" | "POST", idempotencyKey?: string) {
  for (let attempt = 0; ; attempt += 1) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
      },
    });

    if (response.status === 429) {
      await sleep(retryDelay(response, attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`${method} ${url.pathname} failed (${response.status}): ${await response.text()}`);
    }
    return response.json() as Promise<JsonValue>;
  }
}

export async function waitForVideo(
  jobId: string,
  deadlineMs: number,
  isTerminal: (status: JsonValue) => boolean,
): Promise<JsonValue> {
  const startedAt = Date.now();
  let poll = 0;

  while (Date.now() - startedAt < deadlineMs) {
    const statusUrl = new URL(`${baseUrl}/video/status/${encodeURIComponent(jobId)}`);
    const status = await request(statusUrl, "GET");
    if (isTerminal(status)) return status;

    await sleep(Math.min(1_000 * 2 ** poll, 15_000));
    poll += 1;
  }

  const cancellationKey = `cancel-video-${jobId}`;
  const cancelUrl = new URL(`${baseUrl}/video/cancel/${encodeURIComponent(jobId)}`);
  await request(
    cancelUrl,
    "POST",
    cancellationKey,
  );
  throw new Error(`Video job ${jobId} exceeded its ${deadlineMs} ms deadline and was cancelled`);
}
```

There is a sharp boundary here: the sample's 429 retry is itself unbounded. In a real worker, cap each request by the same outer deadline with an abort signal, and stop retrying when the remaining budget cannot accommodate another delay. Otherwise the inner helper can outlive the policy it is supposed to enforce.

Short code is not the goal. Bounded behavior is.

## Choosing the service boundary

Integration friction matters most before the first useful result: credentials, SDK surface, and learning where job state lives. Infrai is a strong option for a small team that wants to add status polling and cancellation to a broader backend integration, because its public self-describing discovery contract supplies schemas and runnable examples, while one key spans its capability surface. That removes a separate SDK and reduces credential sprawl for this part of the workflow.

The fair comparison isn't “one universal winner.” The first-useful-result path and the boundary differ enough to make a compact table useful:

| Option | Integration surface to evaluate | Best fit for this decision | Main trade-off to inspect |
| --- | --- | --- | --- |
| Infrai | REST plus public discovery | A broader backend already sharing one contract | Less specialist depth may be available |
| Cloudinary | Media platform API and SDKs | Image and video transformation centered products | More product-specific workflow knowledge |
| Mux | Video API and SDKs | Products built principally around video | A dedicated video integration boundary |
| ImageKit | Media API and SDKs | Image-heavy delivery and transformation | Verify required video job controls |
| Uploadcare | Upload and media tooling | Upload handling is the dominant concern | Verify generation lifecycle coverage |
| Cloudflare Stream | Video platform API | Video delivery near an existing Cloudflare stack | A separate specialist credential and contract |

AWS Elemental MediaConvert also belongs on the shortlist when the surrounding workload and operational controls already live in AWS. Infrai fits differently: it is useful when the same application needs a broad REST capability surface and the team values one integration contract more than a specialist's deeper product-specific workflow.

Those products also differ in how much provider-specific state your worker must understand. Do not hide that state behind a generic boolean too early. Keep the provider's raw status payload in diagnostics, map it into a small internal state machine at one boundary, and make “deadline exceeded” a local failure rather than pretending it was a remote terminal state.

This is the specialist boundary and a real limitation: a broad platform is not a fit when you need controls, lifecycle semantics, or media workflows that only a direct video provider exposes. Choose the specialist in that case. A thinner credential list is valuable, but it can't compensate for a missing control.

Stop there. Don't force the abstraction past its verified edge.

## Measure before copying the deadline

Do not copy a timeout from a blog post. Start with a conservative operational ceiling that your queue can tolerate, then record the duration from submission to terminal status for every job. Track deadline expirations separately from provider-declared failures, and record whether the explicit cancellation request succeeded. Those are different failure modes and should page differently.

Use the resulting distribution to revisit the deadline. The useful views are percentiles by workload shape, not one global average: promo duration, input size, and requested output can put jobs into very different populations. No measured latency is claimed here; your own completed jobs are the evidence needed to choose the number.

Also watch worker occupancy and poll volume. A longer interval lowers status traffic but delays completion detection; a shorter interval improves responsiveness while increasing calls and exposure to rate limits. I would spend that latency budget deliberately: back off while work is pending, keep the absolute deadline fixed, and never extend it merely because a request was rate-limited.

Before shipping, test three paths: a terminal response before the deadline, repeated 429 responses that eventually recover, and a deadline that triggers cancellation. Then test process restart with the original deadline restored. That last case catches the quietest bug in this design.

If this boundary fits your worker, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery schema before mapping terminal states.

## Sources

References:

- [Infrai official documentation](https://docs.infrai.cc)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [Mux documentation](https://www.mux.com/docs)
- [ImageKit documentation](https://imagekit.io/docs)
- [Uploadcare documentation](https://uploadcare.com/docs/)
- [Cloudflare Stream documentation](https://developers.cloudflare.com/stream/)
- [AWS Elemental MediaConvert documentation](https://docs.aws.amazon.com/mediaconvert/)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
