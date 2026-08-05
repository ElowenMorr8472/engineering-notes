# Picking a Long-Context Chat API for SaaS Support Without Guessing at Quality

Bottom line: for a SaaS support chatbot, start on the smallest chat model that clears your quality bar on real tickets, cap every prompt with a hard token budget, and escalate only the conversations that earn it. Long context is a safety net, not a plan — the API bill tracks what you actually send, and most support threads never need 100k tokens of history behind them.

The three models people line up for this comparison — GPT-4.1 mini, Claude 3.5 Haiku, Gemini 1.5 Flash — are close enough on support-shaped work that the winner is rarely decided by the model. It's decided by what you put in the prompt.

Here's the flow in my app, and it's deliberately boring. The widget posts a message; my server pulls the last few turns from Postgres, adds a system prompt and two or three retrieved doc chunks, counts the tokens, trims the oldest exchanges until the prompt fits a budget, calls the model, then writes the reply and the measured spend back onto the conversation row. Everything that costs money is decided before the request goes out.

That trimming step is the part people skip.

## The token budget guard that goes in front of every call

I don't let a support turn reach a provider until something has told me how big it is. Counting locally with tiktoken is the cheap version — one dependency, no extra round trip, and it runs in the same request:

```ts
import { get_encoding } from "tiktoken";

const enc = get_encoding("cl100k_base");
const BASE_URL = process.env.CHAT_BASE_URL ?? "";   // any OpenAI-compatible /v1 base
const API_KEY = process.env.CHAT_API_KEY ?? "";
const MODEL = "gpt-5.4-mini";
const PROMPT_BUDGET = 12_000;

type Turn = { role: "system" | "user" | "assistant"; content: string };

const countTokens = (turns: Turn[]) =>
  turns.reduce((n, t) => n + enc.encode(t.content).length + 4, 0);

function trimToBudget(turns: Turn[]): Turn[] {
  const kept = [...turns];
  // keep the system prompt and the newest exchanges, drop the oldest pair first
  while (kept.length > 3 && countTokens(kept) > PROMPT_BUDGET) kept.splice(1, 2);
  return kept;
}

export async function reply(history: Turn[], attempt = 0): Promise<string> {
  const res = await fetch(`${BASE_URL}/chat/completions`, {
    method: "POST",
    headers: { authorization: `Bearer ${API_KEY}`, "content-type": "application/json" },
    body: JSON.stringify({ model: MODEL, messages: trimToBudget(history), max_tokens: 600 }),
  });

  if (res.status === 429 && attempt < 4) {
    const after = Number(res.headers.get("retry-after"));
    await new Promise((r) => setTimeout(r, after ? after * 1000 : 2 ** attempt * 500));
    return reply(history, attempt + 1);
  }
  if (!res.ok) throw new Error(`chat ${res.status}: ${(await res.text()).slice(0, 200)}`);

  const body = await res.json();
  return body.choices[0].message.content;
}
```

Two honest caveats about that snippet. The cl100k encoding is an approximation once you point it at anything outside OpenAI's family — as far as I can tell it runs a little under on Claude-style tokenizers, so I keep maybe 15% headroom in the budget rather than trusting the number to the digit. And if you'd rather not ship a tokenizer at all, some gateways will count for you: Infrai answers POST /v1/ai/tokens/count over plain HTTP on the same key as its OpenAI-compatible chat surface, so counting and calling stay one integration instead of two, in whatever language your backend happens to be written in.

The reason I trust the guard at all is a night I'd rather not repeat. I had a summarizer that compressed every conversation past 20 turns and wrote the summary back to the row, and I called it without awaiting it inside the same serverless handler that returned the reply — so the function froze the instant it sent the response, and the write never ran. Every request came back 200. Users got answers, the dashboards were green, nothing errored anywhere. Six hours later the daily cost line was roughly 3x what it should have been, because every long thread was still replaying its full history on every single turn. I found it by diffing prompt_tokens per conversation against the summary column: 41 conversations with a null summary and 30k-token prompts. The lesson wasn't about model choice. It was that a 200 tells you the call succeeded, not that your side of the work happened.

## How do I compare answer quality and long context for a SaaS support chatbot?

Not by chatting with each one for twenty minutes and picking my favourite. I keep 200 real tickets with the resolution a human actually gave, replay them against each candidate nightly, and score three things: did it resolve without a human, did it invent a policy we don't have, did it escalate when it should have. Then I divide the month's spend by resolved conversations. Cost per resolved ticket is the number that survives contact with a board meeting; cost per million tokens is not.

| Option | How you call it | Where it earns its place | Main limitation |
| --- | --- | --- | --- |
| GPT-4.1 mini (OpenAI) | REST, or the OpenAI SDK | Tool calls and strict JSON replies | Drifts on long, meandering threads |
| Claude 3.5 Haiku (Anthropic) | REST, separate SDK and key | Tone and refusals on angry tickets | Its own tokenizer, so your counts won't line up |
| Gemini 1.5 Flash (Google) | REST, separate SDK and key | When you truly must send a whole manual | Another auth flow, another invoice |
| A small open model you host (Ollama, vLLM) | OpenAI-compatible, on your box | Predictable spend, no per-token surprises | You own the uptime and all the eval work |
| An OpenAI-compatible gateway | One base URL and one key | Trying the others without new integrations | You inherit its routing decisions |

Gateways are how I stopped rewriting the client every time I wanted to test a model. Infrai's is a plain REST API — no SDK to install, anything that can send an HTTP request can call it — and the per-call cost, vendor and latency come back as metadata on the response, which happens to be exactly what I needed to compute cost per resolved conversation without building a billing pipeline first.

Quality on this class of work is closer between the three than the benchmarks suggest. On my ticket set the gap that mattered wasn't reasoning, it was instruction-following under a long system prompt.

## Where each of these stops being the right pick

The catch with long context is that it's the most expensive way to remember anything. Sending 60k tokens of history on every turn to save yourself writing a summarizer is a decision you'll pay for daily, forever, and it degrades answer quality too — retrieval accuracy in the middle of a very long prompt is not what the marketing pages imply.

If you're under a few hundred conversations a month, ignore most of this and stick with whichever provider you already have a key for. The integration work costs more than you'll save.

A gateway isn't a good fit either when your product depends on a vendor's newest tool-use mode the week it ships; go direct, accept the extra key, and revisit later. Same story for adjacent capabilities: text chat gateways don't cover realtime voice sessions, and none of the options here give you a dedicated moderation endpoint — you get moderation by asking a chat model for a JSON verdict, which is fine but is a second call you have to budget for. And self-hosting only wins if someone on your team genuinely wants to own a GPU at 3am.

## What I'd actually ship next week

Put the model id behind an env var on day one, because you will change it. Log prompt_tokens, completion_tokens and the model on every turn into the same row as the conversation, so cost per resolved ticket is a SQL query and not a project. Alert on cost per conversation rather than total spend — total spend goes up when you grow, which is the whole point, and it hides the regressions. Move summarization onto a queue with its own retry so a frozen handler can't swallow it the way mine did. Keep the 200-ticket regression set and re-run it whenever the prompt changes, not just when the model does. Then check the model catalog every quarter or so, because availability and prices move under you, and the small model that was good enough last quarter is usually cheaper or better this one.

None of this requires picking the perfect model up front. It requires being able to swap the one you picked in an afternoon.

## References

- OpenAI Chat Completions API reference — https://platform.openai.com/docs/api-reference/chat
- Anthropic Messages API reference — https://docs.anthropic.com/en/api/messages
- Gemini API documentation — https://ai.google.dev/gemini-api/docs
- openai/tiktoken, the official BPE tokenizer — https://github.com/openai/tiktoken
- OpenRouter documentation (model routing over an OpenAI-compatible surface) — https://openrouter.ai/docs
