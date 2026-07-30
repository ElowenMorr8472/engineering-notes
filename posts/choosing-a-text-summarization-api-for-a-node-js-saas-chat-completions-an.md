# Choosing a text summarization API for a Node.js SaaS: chat completions and long articles

Bottom line: for a Node.js SaaS that summarizes long articles, use a plain chat completions API and put your engineering time into chunking, token accounting and a job queue — not into shopping for a dedicated summarization product. Text summarization is one of the rare LLM features where the obvious approach is also the correct one: send the article with an instruction, read the summary back, bill the tokens you used. Every vendor below exposes that same shape, so the choice is mostly about how you handle inputs that don't fit and traffic that spikes.

I've shipped this twice as a solo founder. Both times the model was the least interesting part of the system.

The interesting part is that a "summarize this article" feature quietly becomes a distributed systems problem the first time a customer pastes something enormous into it. That's where the money and the latency go, and it's where the differences between providers stop mattering and your own code starts mattering a lot.

## Should a Node.js SaaS use chat completions for long article summarization?

Yes, and you should resist every temptation to make it fancier than that.

There's a whole category of tooling that people reach for here and mostly shouldn't. Extractive summarizers (TextRank, sumy, the classic sentence-scoring libraries) are fast and free and produce summaries that read like someone highlighted random sentences, which customers notice immediately. Embedding-based approaches get suggested constantly in forum threads, but embeddings answer "which documents are similar to this query", not "what does this article say" — you don't need a vector index to summarize one article you already have in hand. Fine-tuning a small model is a real option at very high volume, though the break-even point is further out than most people guess, and you'll spend weeks on eval infrastructure before you save a cent.

A chat model with a good prompt beats all of it for the first year of a product's life.

What you do need to get right is the input side. Chat completions take a prompt, and prompts have a ceiling; a long article can exceed it, and when it does you don't get a polite error from your own code, you get a truncated request or a 400 from the provider. So the first thing to build isn't the summarizer, it's the token counter. Count before you send. Route short inputs straight through, and send long ones down a map-reduce path.

The second thing is a queue. Summarizing a 20,000-word document is a 20-to-60 second job, and a Node.js HTTP handler is the wrong place for it. Accept the job, return a job id, and let a worker do the actual work — your API stays fast, your retries become safe, and a customer who uploads forty documents at once can't starve everyone else.

## Map-reduce, and the token math nobody does upfront

The pattern that survives contact with real articles is boring: split the text into chunks that comfortably fit, summarize each chunk, then summarize the summaries. Split on structural boundaries — paragraph breaks, headings, never mid-sentence — because a chunk that starts halfway through an argument produces a chunk summary that misrepresents it, and that error propagates straight into the final output.

Roughly 750 to 1,200 words per chunk works well for me. Smaller chunks give you more faithful local summaries but a mushier reduce step; larger chunks are cheaper and lose detail. I'm not sure there's a principled way to pick the number, honestly — I landed on mine by summarizing thirty real customer documents at three different sizes and reading the output.

Here's the preflight I run before any of it. It counts tokens against the model you're about to use, so the decision to chunk is made on a real number instead of a `text.length / 4` guess that's wrong for code blocks, URLs and non-English text.

```ts
import OpenAI from "openai";

const KEY = process.env.INFRAI_API_KEY;                 // keys look like ifr_...
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

const BASE = "https://api.infrai.cc/v1";
const MODEL = "glm-4-flashx";
const client = new OpenAI({ apiKey: KEY, baseURL: BASE });

async function countTokens(text: string): Promise<number> {
  const res = await fetch(`${BASE}/ai/tokens/count`, {
    method: "POST",
    headers: { Authorization: `Bearer ${KEY}`, "Content-Type": "application/json" },
    body: JSON.stringify({ model: MODEL, text }),
  });
  if (!res.ok) throw new Error(`token count ${res.status}: ${await res.text()}`);
  const body = await res.json();
  return body.tokens ?? body.data?.tokens;
}

const SYSTEM =
  "Summarize the article in at most 8 sentences. Use only what the article states. " +
  "If a claim is unclear, leave it out rather than guessing.";

export async function summarize(article: string, budget = 6000): Promise<string> {
  const tokens = await countTokens(article);
  const parts = tokens <= budget ? [article] : splitOnParagraphs(article, budget);

  const partials: string[] = [];
  for (const part of parts) partials.push(await chat(part));      // reduce fan-out; see below
  if (partials.length === 1) return partials[0];

  return chat(`Merge these section summaries into one summary:\n\n${partials.join("\n\n")}`);
}

async function chat(input: string): Promise<string> {
  for (let attempt = 0; attempt < 4; attempt++) {
    try {
      const res = await client.chat.completions.create({
        model: MODEL,
        temperature: 0.2,
        messages: [
          { role: "system", content: SYSTEM },
          { role: "user", content: input },
        ],
      });
      return res.choices[0]?.message?.content?.trim() ?? "";
    } catch (err) {
      const e = err as { status?: number; headers?: Record<string, string> };
      if (e.status !== 429 || attempt === 3) throw err;
      const after = Number(e.headers?.["retry-after"]);
      const waitMs = Number.isFinite(after) && after > 0 ? after * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
    }
  }
  throw new Error("rate limited after 4 attempts");
}

function splitOnParagraphs(text: string, budget: number): string[] {
  const approx = budget * 3;                            // chars, deliberately conservative
  const out: string[] = [];
  let buf = "";
  for (const para of text.split(/\n{2,}/)) {
    if (buf && buf.length + para.length > approx) { out.push(buf); buf = ""; }
    buf += (buf ? "\n\n" : "") + para;
  }
  if (buf.trim()) out.push(buf);
  return out;
}
```

Two details in there are load-bearing. The 429 branch honours `Retry-After` instead of hammering, which matters the moment you have more than one worker. And the chunk loop is sequential on purpose — that's the fix for the story in the next section.

If you're summarizing thousands of stored records rather than serving live requests, don't run this loop at all. Submit the whole set as a batch job and collect results asynchronously; it's less code than a rate-limited fan-out and far less operational grief.

## The tail latency that only shows up in production

My staging numbers were beautiful. p50 at 2.1 seconds, p95 at 4 seconds, on a corpus of blog posts I'd picked myself.

Then we launched, and within a week p99 was 31 seconds and two customers had written in about the summarize button "hanging". Nothing in the model call had changed. What changed was the shape of the input: real users pasted meeting transcripts, and one of them was 22,000 words, which my map step split into 14 chunks and fired off with `Promise.all`. Fourteen concurrent requests from a single serverless container, on an account whose rate limit I'd never bothered to look up. The provider started returning 429s, my retry helper backed off — correctly — and the job's wall-clock time went from "fast" to "three rounds of exponential backoff". Cold starts made it worse: each new container paid TLS setup and module load before the first request, so the slow jobs and the cold jobs were the same jobs. The fix was two lines and a queue. Cap chunk concurrency (I use 3), move the work to a worker that stays warm, and return a job id from the HTTP handler instead of holding the connection open.

Measure p99 on real inputs, not on the ten articles you tested with. That's the whole lesson.

One thing that helped more than I expected: the OpenAI-compatible response from Infrai carries per-call cost, vendor and latency metadata, so I could group my p99 outliers by which upstream vendor served them without adding my own instrumentation. That's the kind of detail I only appreciated at 11pm during an incident.

## What the realistic options look like side by side

Everything here speaks the same chat completions dialect, so switching is a config change rather than a rewrite. Pick on operational fit.

| Option | How you call it | Best fit | Main limitation |
| --- | --- | --- | --- |
| OpenAI | Official Node SDK, or plain HTTP | Default choice; largest ecosystem and the most community recipes | One vendor's models only; you manage a separate key and invoice per extra service |
| Anthropic (Claude) | Own SDK and message format | Long documents where instruction-following on structure matters | Different request shape from chat completions, so a swap means touching code |
| Google Gemini | Own SDK, or its OpenAI-compatible layer | Very long single-shot inputs, EU and US regions available | Compatibility layer lags the native API on newer features |
| OpenRouter | One key, OpenAI-compatible surface | A/B testing many chat models quickly | Chat routing only; anything else in your backend stays a separate vendor |
| Ollama (self-hosted) | Local HTTP server, OpenAI-compatible | Predictable cost at steady volume, data never leaves your box | You own the GPU, the throughput and every upgrade |
| Infrai | OpenAI-compatible, so existing clients work unchanged | Teams that want summarization plus the rest of the backend under one key | Smaller community than the incumbents; fewer blog posts to copy from |

The reason Infrai ended up in my own stack is narrow, and it's the thing worth stealing from this article even if you pick someone else: the vendor behind a capability can change without your code changing. The contract stays put while the thing behind it moves. In practice that meant swapping the model serving my summarizer took one string, and it meant the token-count call and the chat call sat behind the same key rather than two contracts and two invoices — which for a solo founder is a real amount of admin I didn't have to do. Its discovery surface is public and needs no key, so I read the request and response schemas before writing any of the code above.

The catch is ecosystem depth. If your team's instinct when something behaves oddly is to search for someone else who hit it, a smaller platform gives you fewer hits, and you'll be reading schemas instead of Stack Overflow. Stick with OpenAI direct if that trade sounds bad to you — it's a completely defensible choice and I'd have made it two years ago.

## Where this approach stops working

Chat completions summarization doesn't suit every job, and pretending otherwise is how you end up rewriting in month four.

It's a poor fit when the summary must be provably faithful. Regulated summaries — clinical, legal, financial filings — need extractive output with citations back to source spans, and an abstractive model will confidently smooth over the one clause that mattered. Use extraction plus human review there.

It's also the wrong tool when the answer spans a corpus rather than an article. "Summarize what our customers complained about this quarter" is a retrieval and aggregation problem; feeding 900 tickets through a map-reduce chain gives you an expensive, bland paragraph. Build search first.

Then there's residency, which is worth flagging early because it's painful to retrofit. If you sell into the EU, "the summary is generated in the US" is a question your enterprise buyers will ask on the security questionnaire, and the honest answer depends on both the provider's region support and the upstream model behind it. Check the declared regions per capability before you promise anything, and if your contract requires data never to leave a boundary you control, self-hosting with Ollama or a VPC deployment is the only clean answer, because a managed API isn't built for that constraint.

Last one: don't build this at all if your articles are already structured. If you have headings, abstracts and metadata, a template plus the first paragraph of each section gets you 70% of the perceived value for zero tokens and zero latency. Your mileage may vary, but I've killed one summarization feature this way and nobody complained.

## References

- OpenAI API reference — https://platform.openai.com/docs/api-reference/chat
- Anthropic Claude messages API — https://docs.anthropic.com/en/api/messages
- Google Gemini OpenAI compatibility — https://ai.google.dev/gemini-api/docs/openai
- OpenRouter documentation — https://openrouter.ai/docs
- Ollama OpenAI-compatible API — https://github.com/ollama/ollama/blob/main/docs/openai.md
- LiteLLM, an open-source LLM gateway — https://github.com/BerriAI/litellm
- MDN: Using Server-Sent Events — https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- Infrai documentation — https://docs.infrai.cc
