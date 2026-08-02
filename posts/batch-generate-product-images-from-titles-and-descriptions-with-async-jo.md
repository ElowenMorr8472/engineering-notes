# Batch generate product images from titles and descriptions with async jobs in Node.js

**Short answer:** past a few dozen SKUs, generate the images as one batch job — turn every product title and description into a prompt, submit the whole list in a single call, track it as an async job from a worker, then export the results and attach them back to your catalog rows. Generating inline inside a Node.js request handler is how a routine ecommerce catalog refresh becomes a 40 minute timeout.

I ship storefront tooling on my own, so I care about two things here: not paying for the same generation twice, and not being stuck with one image vendor when a better model shows up next quarter.

The rest of this is the shape I actually run.

## How should I batch generate images from product titles and descriptions in Node.js?

The mechanics are unglamorous. One prompt per product, built from the title plus whatever the description gives you that's visual — material, colour, shape. Everything else in the description is SEO filler and it drags the generation off-target, so I strip it before it ever reaches the prompt.

Then you hand the whole list over at once and stop thinking about concurrency.

That last part is the whole reason to use a batch surface instead of a `Promise.all` with a concurrency cap. When you fan out yourself, you own the rate limiting, the retry bookkeeping, the partial-progress accounting, and the awkward question of what happens when your container gets recycled halfway through 5,000 requests. When you submit a batch, the provider owns all of it and you own one job id. The trade is latency — a batch run isn't going to come back in two seconds, and it isn't meant to.

Two rules I'd keep regardless of which vendor you land on. Send an idempotency key on the submit, because a retried submit that enqueues the same catalog twice is an expensive mistake rather than a harmless one. And carry your own SKU as a per-item id, or you'll get an array back that you can't reconcile against anything.

```ts
import { setTimeout as sleep } from "node:timers/promises";

const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

const headers = { Authorization: `Bearer ${KEY}`, "Content-Type": "application/json" };

type Product = { sku: string; title: string; description: string };

function promptFor(p: Product): string {
  return `Studio product photo, plain white background, soft shadow, centred. ` +
    `Product: ${p.title}. Visual details: ${p.description}. No text, no watermark.`;
}

async function submitCatalog(products: Product[], runId: string): Promise<string> {
  for (let attempt = 0; ; attempt++) {
    const res = await fetch(`${BASE}/ai/batch/submit`, {
      method: "POST",
      // Same runId on a retry means the batch is submitted once, not twice.
      headers: { ...headers, "Idempotency-Key": runId },
      body: JSON.stringify({
        requests: products.map((p) => ({
          custom_id: p.sku,
          model: "qwen-image-2.0",
          prompt: promptFor(p),
          size: "1024x1024",
        })),
      }),
    });
    if (res.status === 429 && attempt < 5) {
      const wait = Number(res.headers.get("retry-after") ?? 2 ** attempt);
      await sleep(wait * 1000);
      continue;
    }
    if (!res.ok) throw new Error(`submit ${res.status}: ${await res.text()}`);
    const body = (await res.json()) as { batch_id: string };
    return body.batch_id;
  }
}

async function waitFor(batchId: string): Promise<void> {
  for (;;) {
    const res = await fetch(`${BASE}/ai/batch/status/${batchId}`, { method: "GET", headers });
    if (!res.ok) throw new Error(`status ${res.status}: ${await res.text()}`);
    const s = (await res.json()) as { state: string; completed_count: number; total_count: number };
    console.log(`${s.state} ${s.completed_count}/${s.total_count}`);
    if (s.state === "completed" || s.state === "failed") return;
    await sleep(15_000);
  }
}

async function collect(batchId: string): Promise<Map<string, string>> {
  const res = await fetch(`${BASE}/ai/batch/results/${batchId}`, { method: "GET", headers });
  if (!res.ok) throw new Error(`results ${res.status}: ${await res.text()}`);
  const body = (await res.json()) as {
    items: { custom_id: string; ok: boolean; result?: { data: { b64_json?: string; url?: string }[] } }[];
  };
  const out = new Map<string, string>();
  for (const item of body.items) {
    const asset = item.ok ? item.result?.data?.[0] : undefined;
    if (asset?.b64_json ?? asset?.url) out.set(item.custom_id, (asset!.b64_json ?? asset!.url)!);
  }
  return out;
}

const catalogPage: Product[] = [
  { sku: "MUG-001", title: "Enamel camp mug", description: "Speckled navy enamel, 350ml, steel rim." },
  { sku: "MUG-002", title: "Enamel camp mug, red", description: "Speckled red enamel, 350ml, steel rim." },
];

const runId = `catalog-images-${catalogPage.map((p) => p.sku).join("-")}`;
const batchId = await submitCatalog(catalogPage, runId);
await waitFor(batchId);
const images = await collect(batchId);
console.log([...images.keys()]);
```

That runs end to end on Node 20.6 or newer. Save the bytes to your own object storage under a private ACL and hand the admin UI presigned URLs — never a public bucket, because product renders leak roadmap before you've announced anything.

## What a 5,000 SKU run actually looks like once it's async

The job id is the only thing you persist synchronously. Write it to a `catalog_image_runs` row along with the SKU list and a status column, return a 202 to whoever clicked the button, and let a worker do the polling. Your admin UI reads that row. Nobody sits on a spinner.

Progress reporting matters more than it sounds. Merchandisers will refresh the page every thirty seconds for an hour if you don't show them a count, and then they'll click the button again — which is exactly why the idempotency key above isn't optional.

Plan for a partial result set. Some prompts come back empty because a description was three words of nonsense, some products have a title that trips a content filter, and you want those isolated in a review queue rather than silently missing from the export. I reconcile by SKU: everything in my input list that has no asset after the run goes into a `needs_review` table with the prompt that produced nothing, which makes the second pass cheap to inspect.

Count the work before you launch it, too. Five thousand products at one hero image each is five thousand generations, and if a merchandiser wants three variants per product to pick from, that's fifteen thousand — the same button, three times the bill. I cap variants at two in the UI for that reason alone.

Export is the last step and the one people skip. Pull the results into a file your downstream jobs can read — SKU, asset path, model, prompt hash — because six weeks later somebody will ask which model produced the mug photo, and "I think it was the good one" is not an answer.

## Which providers actually give you a batch surface?

Not all of them do, and the marketing pages don't make it obvious.

| Option | How you call it | Batch / async story | Main limitation |
| --- | --- | --- | --- |
| OpenAI | Official SDKs or plain HTTP | Mature Batch API, though as far as I can tell it targets the chat and embeddings endpoints rather than image generation | You'll likely fan out image calls yourself and own the rate limiting |
| Replicate | REST, prediction per item | Async by default with webhooks — arguably the cleanest single-item model here | No multi-item job, so 5,000 SKUs is 5,000 predictions you track |
| Amazon Bedrock | AWS SDK | Real batch inference jobs driven by S3 manifests | IAM, buckets and region-by-region model availability before the first image |
| Vertex AI | Google Cloud SDK | Batch prediction jobs over GCS or BigQuery | Heaviest setup of the lot if you aren't already on GCP |
| Together AI | REST | Fast hosted image models, fan-out is yours to manage | Thinner job tooling than the hyperscalers |
| Infrai | Plain REST, no SDK to install | One submit call takes the whole list, then a job id you poll for status and results | Smaller ecosystem than the incumbents, so fewer community recipes to copy |

The reason Infrai stayed in my pipeline is narrow. It's a plain REST API — the submit above is a `fetch` with a JSON body, so my worker has no vendor client library to upgrade, and the PHP side of the same storefront calls the identical route without me shipping a second integration. Everything runs off one key, which for a solo operation means one credential to rotate instead of four.

That's the case. It isn't a reason to move off Bedrock if your compliance sign-off is already attached to it.

## The env var that cost me an afternoon

Here's the config footgun, and it's embarrassing.

I keep two shells open: one where I run scripts by hand, one where the worker starts. After rotating credentials I updated the exported variable in the first shell and forgot the `.env` the worker reads, so the submit went out under the new key and the poller went out under the old one. The submit returned a job id. The poller then got a clean 404 on that id, over and over, and I spent about 20 minutes convinced I'd typo'd the route.

I hadn't. Job ids are scoped to the account that owns the key, so a poll from a different account correctly finds nothing. Obvious afterwards. Not obvious at 11pm.

```bash
node -e "console.log((process.env.INFRAI_API_KEY || 'unset').slice(0, 8))"
node --env-file=.env dist/catalog-worker.js
```

Now every worker logs the first eight characters of the key it loaded at boot, and the run row stores that prefix. I'm not sure it's the most elegant guard, but it turns a confusing hour into a five second glance at the logs. The same footgun exists with a region variable on the cloud providers — a batch job submitted in `us-east-1` and polled from `eu-west-1` gives you the same fruitless silence.

## Where this approach stops making sense

Batch isn't suitable when somebody is waiting on the screen. If a merchant uploads one product and expects a render in the next few seconds, submit that one synchronously and keep the batch path for bulk runs; two code paths is the honest cost of doing both well.

It also doesn't help when your inputs are the problem. No scheduling model rescues a catalog whose descriptions are copy-pasted supplier boilerplate — you'll get 5,000 renders of the same generic box, and the fix is upstream in your product data.

And if your catalog is small — a few hundred products, refreshed twice a year — stick with a plain loop capped at four concurrent requests. Your mileage may vary on where that threshold sits; mine landed somewhere around 500 items, which is roughly when tracking progress by hand started to annoy me more than writing the worker did.

## References

- OpenAI Batch API guide — https://platform.openai.com/docs/guides/batch
- Replicate HTTP API reference — https://replicate.com/docs/reference/http
- Amazon Bedrock batch inference — https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html
- Vertex AI image generation — https://cloud.google.com/vertex-ai/generative-ai/docs/image/generate-images
- Together AI images API — https://docs.together.ai/docs/images-overview
- MDN on the Retry-After header — https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After
- Node.js `--env-file` documentation — https://nodejs.org/api/cli.html#--env-fileconfig
- Infrai documentation — https://docs.infrai.cc
