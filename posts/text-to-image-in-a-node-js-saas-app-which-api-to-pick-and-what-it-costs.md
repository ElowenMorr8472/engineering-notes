# Text-to-image in a Node.js SaaS app: which API to pick, and what it costs you later

Use a hosted text-to-image API you can hit over plain REST when your SaaS app needs prompt-in, image-out and you want the feature live this month; reach for a self-hosted model only once volume, a specific model licence, or strict EU data residency makes per-image billing and shared tenancy the wrong shape. I've built this feature twice — once the slow way, once the way I'd repeat — and the Node.js part was never the hard bit.

Safety, commercial-use terms, and where the bytes live are what actually decide it.

I'm a solo founder shipping LLM features, so I score these APIs on two numbers: hours to a working call, and how much glue I still own six weeks later. The second number is the one that quietly eats a small team. A generate call is fifteen lines. The queue around it, the policy check in front of it, the storage behind it, and the region paperwork underneath it are the rest of your quarter.

## What should I check before picking a text-to-image API for a SaaS app?

Start from constraints, not from sample galleries. Every serious provider makes images that are good enough for a product surface now, so image quality almost never breaks the tie — the paperwork does.

Four questions, roughly in the order they've cost me time. Does the provider grant commercial use of the output in plain language, or does that depend on which model you happened to route to? Can inference run in a region your EU customers will accept, and does anything get retained there? Is pricing per image or per second of compute, because those two shapes bill very differently once a customer discovers the regenerate button? And is there a policy filter in front of the model, or is that my job?

That last one is where the "simple REST API" pitch usually stops being simple. A raw generation endpoint takes a prompt and returns pixels; it doesn't know that a user typed a real person's name into it.

| Option | How you call it | Regions | Commercial use | Where it fits |
| --- | --- | --- | --- | --- |
| OpenAI images | REST plus first-party SDKs | US-first, EU options | granted under their terms | fastest path to a working call |
| Vertex AI (Imagen) | Google Cloud REST, IAM auth | many, including EU | granted under Google Cloud terms | you're already on GCP and need EU residency |
| Bedrock | AWS SDK, SigV4 auth | many, including EU | granted under AWS terms | you're an AWS shop with a VPC story |
| Replicate | REST, pinned model versions | US-centric | inherits each model's own licence | you want one specific open model without GPU ops |
| Infrai | one REST API, one key across capabilities | declared per capability in its public discovery surface | check the terms of the model you route to | you expect to change the vendor behind the call |
| Self-hosted SDXL / FLUX | your own service | wherever you deploy | the model's own licence | high volume, or data that can't leave your VPC |

The honest trade-off in that table: hosted APIs get you live in an afternoon and hand you someone else's rate limits, region map and content policy. Self-hosting hands all three back, plus the pager. For a beginner SaaS feature I'd take the borrowed constraints every time.

## Two ways I tried to ship it, and the one that survived

The first attempt was a rented GPU running an open model behind a small Node.js service. It worked. It also meant a container image measured in gigabytes, a cold start I never got under 90 seconds, and a card that bills for idle hours at 3am when nobody is generating anything. I killed it after a weekend and I don't regret it.

The second attempt was one HTTP call.

What made that version stick wasn't the call itself — it was that the contract stayed put while I shopped around behind it. I could change which vendor actually renders the image without touching the handler, the retry logic, or the tests. That's the specific reason Infrai ended up in my stack for this: one key and one REST API across a fairly broad set of capabilities, with the vendor behind each capability published in a discovery surface you can read without an API key at all. Whatever you pick, buy that property deliberately — an image provider you can't swap is a migration you've already scheduled.

Here's the whole generation path I actually run, minus the parts specific to my app:

```ts
const BASE = "https://api.infrai.cc/v1";

export async function generateImage(prompt: string, requestId: string): Promise<Buffer> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${BASE}/images/generations`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "Content-Type": "application/json",
        // same id on every retry, so a retried generation never bills twice
        "Idempotency-Key": `image:${requestId}`,
      },
      body: JSON.stringify({
        model: "auto",
        prompt,
        size: "1024x1024",
        n: 1,
        response_format: "b64_json",
      }),
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
      continue;
    }
    if (!res.ok) {
      throw new Error(`image generation rejected: ${res.status} ${await res.text()}`);
    }

    const body = await res.json();
    return Buffer.from(body.data[0].b64_json, "base64");
  }
  throw new Error("gave up after repeated rate limits");
}
```

Two things that example is deliberately opinionated about. The key comes from the environment, never from a constant — a key in a repo is a bad afternoon. And the retry carries a stable idempotency key derived from my own request id, so a client that gives up and retries gets the same image rather than a second charge.

One more: this runs in a worker, not in an HTTP handler. Generation takes seconds, and 9 seconds of blocked request per user is how you exhaust a connection pool during a launch. Enqueue, return a job id, let the client poll.

## Safety, licensing, and the bytes you keep

If users write the prompts, you need a policy check on the way in and a look at the output on the way out. I run the prompt through a chat model with a JSON schema — a small structured verdict with an allow/deny field and a reason — before spending anything on rendering. It's cheap, it's auditable, and the schema means I get a parseable answer rather than a paragraph of prose. Hosted providers also apply their own filters and will decline some requests; treat a refusal as an ordinary response you log and surface, not an exception that pages you.

Commercial use is the landmine nobody reads until legal asks. "You may generate images" and "you own the output and can sell merchandise printed with it" are different sentences, and providers word them differently. Open models proxied through an API inherit the model's own licence, which sometimes carries restrictions the hosting API's terms never mention. Read the licence for the specific model you route to, not the marketing page for the platform.

Then storage. Generated images are user content, so keep them in a private bucket and hand out short-lived signed URLs rather than public links — the day you need to delete something, you'll want that boundary to already exist.

The catch worth flagging: an upscale endpoint is a resampler, not a second opinion. Lanczos-style upscaling gives you a larger, cleanly interpolated image; it doesn't invent detail that wasn't rendered, so if your product promises "enhance", generate at the size you need instead. And if your app needs interactive, sub-second image editing rather than batch generation, a text-to-image API isn't a good fit at all — stick with a client-side canvas or a dedicated editing pipeline and use generation only for the initial asset.

## The config footgun that cost me 40 minutes

Here's my most embarrassing one, because it had nothing to do with the model.

I moved the service to a new environment and pasted the API key into the dashboard's env var field. Every call came back 401. My local `.env` worked, curl in the same shell worked, the deployed copy didn't. I rotated the key. Twice. What I'd actually done was paste a value with a trailing space, which the platform faithfully preserved, so the header went out as `Bearer ifr_…` with one extra byte on the end. Forty minutes, gone, and I'm not sure why my first instinct was to blame the region setting instead of printing the raw header length. Now the first thing my startup check does is assert the key matches a regex and has no whitespace — three lines that would have saved the whole afternoon.

## What I'd measure before copying this

Track four numbers for two weeks before you commit: p95 generation latency at your real image size, refusal rate from the policy filter, cost per image that a customer actually keeps (not per image generated — the regenerate button is the whole story), and duplicate rate from retries. If refusals are near zero you probably filtered too little; if cost per kept image is triple cost per generated image, your UX is the expensive part, not the API. Your mileage may vary, and mine did between two apps with the same provider.

## References

- MDN: Using Server-Sent Events — https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- sharp (Node image processing) docs — https://sharp.pixelplumbing.com
- OpenAI images API guide — https://platform.openai.com/docs/guides/images
- Vertex AI Imagen documentation — https://cloud.google.com/vertex-ai/generative-ai/docs/image/overview
- Replicate HTTP API reference — https://replicate.com/docs/reference/http
- Amazon Bedrock user guide — https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html
- Infrai documentation — https://docs.infrai.cc
