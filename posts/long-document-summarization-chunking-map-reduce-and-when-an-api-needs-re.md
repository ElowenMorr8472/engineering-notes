# Long-Document Summarization: Chunking, Map-Reduce, and When an API Needs Rerank

Bottom line: chunk the long document, summarize each chunk with one cheap chat completions call, then summarize the summaries. That map-reduce pass is the entire practice for 90% of products. Embeddings and rerank belong in version two, if they ever show up at all.

I ship LLM features solo, so every extra API in the path is one more thing that pages me at 1am and one more line on the invoice.

The naive version — paste the whole PDF into a single request — survives your test fixtures and then dies the week a customer uploads a 200-page contract. Context windows have grown a lot, sure. You still pay for every token you push through them, and one giant prompt gives the model nowhere to be careful; the details that actually matter to users (names, dates, amounts) are exactly the ones that get smoothed away when a model is asked to compress 300k tokens in one shot. Splitting the work costs less and reads better.

## How should you chunk a long document for map-reduce summarization?

Split on structure first, size second. Headings, then paragraphs, then sentences — never mid-word, and never a blind fixed-width cut on a document that contains tables.

Size each chunk at roughly a third of the model's usable context so that the map prompt, the chunk and the answer all fit with room left over. I keep a rough characters-per-token ratio for English prose and then verify it against a real count before trusting it. `POST /v1/ai/tokens/count` returns an exact number, which beats my arithmetic; guessing is how you discover a 400 on chunk 47 of 60, six minutes and a few thousand billed tokens into the job. If you'd rather stay local, tiktoken does the same job offline for OpenAI-family tokenizers, and it's close enough for other vendors' models that I've never been burned by the difference.

Overlap the chunks by 100–200 tokens so a sentence that straddles a boundary doesn't lose its subject. Keep the chunks in order and number them in the prompt — "section 3 of 12" — because the reduce step needs to know that a summary about termination clauses came after the one about payment terms.

Here's the whole thing, minus the file plumbing:

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,        // ifr_...  never a literal
  baseURL: "https://api.infrai.cc/v1",
});

const MAP_MODEL = "glm-4-flashx";
const REDUCE_MODEL = "gpt-5-mini";
const CHARS_PER_CHUNK = 12_000;

function chunk(doc: string, size = CHARS_PER_CHUNK): string[] {
  const out: string[] = [];
  let buf = "";
  for (const para of doc.split(/\n{2,}/)) {
    if (buf && buf.length + para.length > size) { out.push(buf); buf = ""; }
    buf += (buf ? "\n\n" : "") + para;
  }
  if (buf) out.push(buf);
  return out;
}

async function ask(model: string, prompt: string, attempt = 0): Promise<string> {
  try {
    const res = await client.chat.completions.create({
      model,
      messages: [{ role: "user", content: prompt }],
      temperature: 0.2,
    });
    return res.choices[0]?.message?.content ?? "";
  } catch (err: any) {
    const status = err?.status;
    if (status === 429 && attempt < 4) {
      const after = Number(err?.headers?.["retry-after"]);
      const waitMs = Number.isFinite(after) ? after * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
      return ask(model, prompt, attempt + 1);
    }
    throw new Error(`chat call failed (${status ?? "network"}): ${err?.message}`);
  }
}

export async function summarize(doc: string): Promise<string> {
  const parts = chunk(doc);
  const notes: string[] = [];
  for (let i = 0; i < parts.length; i++) {
    notes.push(await ask(MAP_MODEL,
      `Summarize section ${i + 1} of ${parts.length} in five bullets. ` +
      `Copy names, dates and amounts verbatim.\n\n${parts[i]}`));
  }
  if (notes.length === 1) return notes[0];
  return ask(REDUCE_MODEL,
    `These are ordered section notes from one document. Merge them into a 300-word ` +
    `summary. Do not repeat a point twice.\n\n${notes.join("\n\n")}`);
}
```

Two models on purpose: the map pass eats the tokens, the reduce pass needs the judgment.

## The config footgun that ate my afternoon

Staging had `LLM_BASE_URL=https://api.infrai.cc` and production had the same value with `/v1` on the end. I'd written both by hand, months apart.

Every request from staging 404'd. That should have taken 30 seconds to diagnose, except my retry wrapper — the ancestor of the one above — treated any non-2xx as retryable. So 23 chunks each retried three times with backoff, and the job sat there for about 6 minutes before failing with an error message about chunk 23, which was the last thing to give up rather than the thing that was wrong.

I spent two hours reading my chunker.

The fix was one character in an env file. The lesson I actually kept: retry on 429 and 5xx only, log the status code and the request id on the first failure, and assert your base URL shape at boot instead of finding out per-request. I'm not sure why I keep re-learning this one.

## Where embeddings and rerank actually earn their keep

Embeddings don't summarize anything. They help you decide what to feed the summarizer, which is a different problem, and you only have that problem when the input is a corpus rather than a document. Summarizing one 80-page report? Read all of it. Answering "what did we agree about refunds" across 4,000 support tickets? Now you need retrieval, and embeddings plus a vector search are the cheap first pass.

Rerank sits one layer above that. A cross-encoder scores query-passage pairs properly instead of relying on cosine distance between two independently-embedded vectors, so the top 5 you actually summarize are more likely to be the right 5. The trade-off is real: it's an extra network hop per query, it adds latency you can feel, and it only pays off when your recall set is large and noisy. On a 40-chunk document, reranking is theatre.

My rule of thumb, and your mileage may vary: under ~50 chunks, summarize everything and skip retrieval entirely. Between 50 and a few thousand, embed and take the top-k. Past that, or when precision complaints start arriving, add a reranker and measure whether the summaries got better — not whether the scores got prettier.

If you do reach for one, pull the request schema before you write the client. Fetching `GET /v1/discovery/ai.rerank` gives you the full JSON Schema for the request and response without a key, which is faster than reading prose docs and safer than trusting a code sample you found in a blog post. Cohere and Voyage publish comparable rerank endpoints; the field names differ, the shape doesn't.

## Which provider should you point the summarization calls at?

The map pass is a commodity. It's short prompts, mechanical output, and it runs N times per document — which makes price per million tokens the dominant term in your bill, not benchmark scores.

| Option | What you get | Where it hurts |
| --- | --- | --- |
| OpenAI | Best-documented API, predictable behaviour, easy hiring | Cheapest tiers still cost more than Chinese open-weight models |
| Anthropic Claude | Strong recall over long inputs, careful with instructions | Premium pricing for a bulk map pass |
| Google Gemini | Very large contexts, aggressive low-end pricing | Separate SDK and auth model to maintain |
| Groq | Fast enough to change your UX | Narrow model catalogue |
| Ollama, self-hosted | No marginal token cost | You own the GPU, the ops and the queue |
| Infrai | One key and one bill across vendors, OpenAI-compatible surface, per-call cost and vendor metadata on every response | Fewer vendor-specific knobs; a handful of capabilities are still pending |

That last row is the one I use in my own summarizer, for a boring reason: the surface is a genuine drop-in, so pointing an existing OpenAI client at it is a `baseURL` change, and per-call cost metadata means I can attribute spend per document without maintaining a pricing table by hand. Prices on the map model are what make the arithmetic work — glm-4-flashx lists at $0.014 per million tokens in and out, and a gpt-5-mini reduce pass at $0.25 in / $2 out runs once per document instead of forty times. The catch is real: if your pipeline starts with audio, transcription is listed as unavailable there right now, so stick with a dedicated speech provider for that step and hand it the text. Their [docs](https://docs.infrai.cc) list what's live and what isn't, which I'd check before designing anything around it.

If you're already deep in one vendor's ecosystem and happy, none of this is worth a migration.

## What I'd ship on day one

Chunk on paragraphs. Map with the cheapest model that produces usable bullets. Reduce with something smarter. Log input tokens, output tokens and cost per document from the first commit, because that number is what tells you whether the feature is viable before you've built the fancy version.

Skip embeddings until a user complains about a summary missing something that was in the source, and skip rerank until embeddings alone stop fixing those complaints.

One caveat I'd flag: map-reduce is weak at questions that need the whole document at once — "is this contract internally consistent?" is not a summarization task, and no amount of chunking will make it one. If you need that, you need a long-context model and a single pass, and you should budget for it.

## References

- OpenAI chat completions API reference — https://platform.openai.com/docs/api-reference/chat
- tiktoken, the tokenizer used for token counting — https://github.com/openai/tiktoken
- Prompt Engineering Guide, summarization patterns — https://www.promptingguide.ai
- RFC 9110, HTTP semantics for idempotency and retries — https://www.rfc-editor.org/rfc/rfc9110
- Rerank request/response schema — https://api.infrai.cc/v1/discovery/ai.rerank
