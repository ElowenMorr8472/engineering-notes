# JSON extraction from long documents in Node.js: chunking, token limits, timeouts

## TL;DR

If your text-to-JSON extraction call times out on a long document, don't go shopping for a bigger context window. Split the document into chunks that line up with your schema, extract JSON per chunk under a hard timeout, validate, then merge. Embeddings and a rerank pass earn their place only when you need a few fields out of a large pile of text — for whole-document extraction they mostly buy you cost and latency.

I build LLM features solo, so my bias is out in the open: p95 latency, the monthly token bill, and never writing an extraction layer I can't point at a different provider next quarter.

## Why one extraction call on a long document times out

Two limits are in play, and most people blame the wrong one.

The input token limit gets all the attention, but what usually kills the request is output. Generation is serial. One object covering a 60-page contract can run to five or six thousand output tokens, and at the throughput I actually see from Node — somewhere around 40 to 80 tokens per second on hosted mid-size models — that's well over a minute of streaming before the closing brace shows up. Your platform has opinions about that: a serverless function capped at 30 seconds, an ingress with a 60 second idle timeout, a load balancer that drops the socket and tells nobody. The SDK's own default timeout is normally far more generous than the gateway sitting in front of your app, so the connection dies long before the client would have given up on its own.

There's a second failure that looks like a timeout and isn't. When the response hits the output cap mid-object you get truncated JSON, `JSON.parse` throws, and a naive retry pays for the entire chunk again. Check `finish_reason` before you parse anything.

Accuracy sags too. Models recall the start and the end of a very long input better than the middle, which is exactly where page 30 of the invoice lives, so a single-shot extraction over 80 pages quietly drops records rather than erroring.

## Should I chunk a long document or just raise the token limit?

Chunk it. And **chunk to your schema, not to a fixed 512-token window.**

If you're pulling line items out of invoices, the boundary is a page or a table. If you're pulling parties and dates out of a contract, it's a clause heading. A record split across two chunks becomes two half-records, and merging half-records is where extraction pipelines go to die. I carry a tail of overlap between chunks and give every record a natural key so the duplicates collapse on merge.

```ts
import OpenAI from "openai";
import { z } from "zod";

const client = new OpenAI(); // reads OPENAI_API_KEY

const LineItem = z.object({
  description: z.string(),
  qty: z.number(),
  unitPrice: z.number(),
});
type LineItem = z.infer<typeof LineItem>;

const jsonSchema = {
  type: "object",
  additionalProperties: false,
  required: ["items"],
  properties: {
    items: {
      type: "array",
      items: {
        type: "object",
        additionalProperties: false,
        required: ["description", "qty", "unitPrice"],
        properties: {
          description: { type: "string" },
          qty: { type: "number" },
          unitPrice: { type: "number" },
        },
      },
    },
  },
};

async function extractChunk(chunk: string): Promise<LineItem[]> {
  const res = await client.chat.completions.create(
    {
      model: "gpt-5-mini",
      messages: [
        { role: "system", content: "Extract invoice line items from the text. Return an empty items array when the text has none. Never guess a number." },
        { role: "user", content: chunk },
      ],
      response_format: {
        type: "json_schema",
        json_schema: { name: "line_items", strict: true, schema: jsonSchema },
      },
    },
    { signal: AbortSignal.timeout(20_000) }, // per-request deadline, separate from the batch budget
  );

  const choice = res.choices[0];
  if (choice.finish_reason === "length") throw new Error("output truncated - chunk is too large");
  const parsed = JSON.parse(choice.message.content ?? '{"items":[]}');
  return z.array(LineItem).parse(parsed.items);
}
```

The batch around it is boring on purpose: a fixed number of workers pulling from a queue, one deadline per request, and a failed chunk logged instead of thrown so 47 good chunks aren't lost to one bad page.

```ts
function chunkBySection(text: string, maxChars = 6_000, overlap = 800): string[] {
  const sections = text.split(/\n(?=#{1,3}\s|\f)/); // headings, or form-feed page breaks
  const chunks: string[] = [];
  let buf = "";
  for (const s of sections) {
    if (buf.length + s.length > maxChars && buf) {
      chunks.push(buf);
      buf = buf.slice(-overlap); // carry a tail so a split record survives
    }
    buf += s;
  }
  if (buf.trim()) chunks.push(buf);
  return chunks;
}

async function extractDocument(text: string, concurrency = 4): Promise<LineItem[]> {
  const queue = [...chunkBySection(text).entries()];
  const out: LineItem[] = [];

  await Promise.all(
    Array.from({ length: concurrency }, async () => {
      for (let job = queue.shift(); job; job = queue.shift()) {
        const [i, chunk] = job;
        try {
          out.push(...(await withRetry(() => extractChunk(chunk))));
        } catch (err) {
          console.error({ chunk: i, err: String(err) }); // report the gap, keep the batch alive
        }
      }
    }),
  );

  const seen = new Set<string>();
  return out.filter((it) => {
    const key = `${it.description.toLowerCase().trim()}|${it.qty}|${it.unitPrice}`;
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  });
}
```

Schema enforcement differs by provider, and it matters more than the model ranking does, because a shape you can trust is what lets you skip a repair pass:

| Runtime | How you pin the JSON shape | Fits when | Watch out for |
| --- | --- | --- | --- |
| OpenAI | `response_format` with a strict `json_schema` | you want the shape guaranteed, not requested | the strict dialect is a subset of JSON Schema, so exotic constructs get rejected up front |
| Anthropic Claude | a tool definition with `input_schema`, forced via `tool_choice` | long chunks, prompt caching across many extractions | adherence is strongly encouraged rather than hard-enforced, so validate the payload |
| Google Gemini | `responseMimeType: "application/json"` plus `responseSchema` | high chunk volume, wide input windows | the schema dialect is its own subset; keep types simple |
| Ollama (Llama, Mistral, Qwen locally) | `format` set to a JSON schema | private documents, free iteration, no per-token meter | small models drop fields as the chunk grows; keep chunks short |

## Where embeddings and rerank actually pay off

Retrieval isn't free, and I learned that the expensive way.

I had 1,100 PDFs chunked at 400 tokens with 50% overlap — about 4,800 chunks per rebuild — and my embedding cache key included the build id. Every deploy re-embedded the whole corpus from scratch. Six deploys in one afternoon, and the embeddings line on that month's bill came to $137 for a job that should have run exactly once and cost single digits. What made it invisible was that nothing failed: no timeout, no error, just a quiet loop doing honest work I'd already paid for. The fix took ten minutes — hash the chunk text, use the hash as the cache key, store the vector next to it. I'm not sure why I ever keyed a cache on a value that changes on every build.

Retrieval earns its keep when the answer is small and the haystack is large. Embed once, pull the top 20 or 30 chunks for a query, then run a rerank model over that shortlist and keep 5 to 8. Cohere Rerank and the open bge-reranker family both do this well, and the win is precision: fewer chunks into the extraction call means fewer input tokens, lower latency, and a smaller blast radius when something is wrong.

If you need every line item on every page, retrieval will silently miss some. Stick with the full chunk sweep there and pay for the tokens. Rerank is a filter for questions with narrow answers, not a substitute for exhaustive extraction.

## Timeouts, retries, and the parts that bite in production

Give every request a deadline and the whole document a budget. `AbortSignal.timeout()` is built into Node now, and passing the signal to the SDK means a stalled chunk releases its worker instead of holding the batch hostage.

Retry only what's safe to repeat. RFC 9110 draws that line at idempotency, and a POST to a completions endpoint isn't idempotent by spec — repeating it is safe here only because extraction is a pure function of the chunk and the natural key stops a duplicate record from landing twice. Retry on 429, on 5xx-class responses, and on socket resets (the ones that surface as ECONNRESET). Never on a 400: the schema is wrong, and trying again just burns tokens at the same rate.

```ts
async function withRetry<T>(fn: () => Promise<T>, tries = 3): Promise<T> {
  let lastErr: unknown;
  for (let i = 0; i < tries; i++) {
    try {
      return await fn();
    } catch (err) {
      lastErr = err;
      const status = (err as { status?: number }).status ?? 0;
      if (status === 400 || status === 401 || status === 422) throw err; // deterministic, don't pay twice
      await new Promise((r) => setTimeout(r, 2 ** i * 500 + Math.random() * 250));
    }
  }
  throw lastErr;
}
```

The vendor-neutrality part is worth ten minutes of your time. Keep the provider call behind one function, or put a self-hosted gateway like LiteLLM in front of it, and switching between OpenAI, Anthropic, Gemini and a local Ollama box becomes a config change rather than a rewrite of the extraction layer. The catch is that a gateway is one more service to run and one more hop of latency; if you only ever call one provider, that trade-off doesn't pay.

Log per chunk: chunk hash, input and output tokens, latency, `finish_reason`, retry count. A rising share of `length` finishes is your early warning that chunks have grown past the output budget, and without those numbers a token spend regression looks like absolutely nothing until the invoice lands.

None of this is clever. It's a queue, a schema, and a stopwatch — and it's the difference between an extraction endpoint that answers in 12 seconds and one that dies at the gateway with the user's upload still in flight. Your mileage may vary by document shape; contracts and invoices behave nothing alike, and I retune chunk boundaries every time a new format shows up.

## References

- RFC 9110: HTTP Semantics (idempotency and retry semantics) — https://www.rfc-editor.org/rfc/rfc9110
- MDN: `AbortSignal.timeout()` — https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static
- OpenAI structured outputs — https://platform.openai.com/docs/guides/structured-outputs
- Anthropic tool use (schema-shaped output) — https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview
- Gemini API structured output — https://ai.google.dev/gemini-api/docs/structured-output
- Ollama structured outputs — https://ollama.com/blog/structured-outputs
- Cohere Rerank — https://docs.cohere.com/docs/rerank-overview
- LiteLLM (self-hosted LLM gateway, open source) — https://github.com/BerriAI/litellm
- Zod — https://github.com/colinhacks/zod
- Liu et al., "Lost in the Middle: How Language Models Use Long Contexts" — https://arxiv.org/abs/2307.03172
