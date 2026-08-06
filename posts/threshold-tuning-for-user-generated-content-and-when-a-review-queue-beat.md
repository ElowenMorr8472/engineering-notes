# Threshold tuning for user-generated content, and when a review queue beats a hard block

Use a three-way router — allow, review, block — the moment your moderation model can hand back per-category confidence; otherwise reach for a single hard threshold and accept that a slice of legitimate user content dies with it. Most false positives I've labeled weren't the model being stupid. They were a policy that never defined "harassment", collapsed into one yes/no gate, running against content the policy author had never seen.

The eval constraint shaped this more than the router did.

I had about 1,400 flagged items sitting in my own logs and zero labeling budget, so I hand-labeled 300 of them across two evenings and called that my eval set. Small. Skewed hard toward my own product's slang. It still told me the one thing no vendor benchmark could tell me: 18% of what my first version blocked was ordinary conversation, and roughly half of those were people quoting abuse that had been aimed at them.

## Where the false positives actually come from

Four patterns covered nearly everything in that 300. Slang and reclaimed slurs inside a community that uses them affectionately. Quoted abuse, where someone pastes the thing that was said to them into a report and the classifier scores the paste rather than the intent. Medical and harm-reduction language, which sits close enough to self-harm content that a generic category will happily grab it. And consensual adult context in a product that permits it, scored against a category written for a product that doesn't. None of those are hallucination — the model scored exactly what my policy asked it to score, and my policy said "harassment" with no definition attached and no carve-outs. So I rewrote each category as one sentence with an explicit exclusion baked in ("quoting abuse inside a report is not harassment"), left the prompt structure alone, and re-ran the same eval. Wrong blocks went from 18% to 6%. No threshold changed. Policy writing beat prompt engineering by a margin I honestly did not expect, and it's the practice I'd hand to anyone starting from scratch on user-generated content.

The remaining 6% is why you need a queue.

## How should a moderation review queue decide what to allow, block, or escalate?

Two numbers per category, not one. Below the low number, allow. Between the two, hand it to a human. Above the high number, block outright. Keeping those numbers in a plain object rather than baked into the prompt is what let me push the sexual-minors block threshold down near zero while making harassment extremely forgiving, without touching a single word of the classifier instructions or re-running my eval on the categories I hadn't changed.

The other thing that goes in the JSON is context, not just scores. A `quoted` boolean costs one extra field and kills the single largest false-positive bucket I had, because it lets the router keep a high harassment score while still declining to block — the same score means something different when the author is the target rather than the source.

Here's the routing layer I run. One POST to `/v1/chat/completions` with a strict JSON schema, nothing exotic:

```ts
import OpenAI from "openai";

type Verdict = { action: "allow" | "review" | "block"; reason: string };

// Two thresholds per category. Anything between them goes to a human.
// quoteExempt marks the categories where "the author is reporting this" is a real defence.
const POLICY: Record<string, { review: number; block: number; quoteExempt: boolean }> = {
  sexual_minors:  { review: 0.05, block: 0.15, quoteExempt: false },
  self_harm:      { review: 0.30, block: 0.90, quoteExempt: false },
  violent_threat: { review: 0.35, block: 0.85, quoteExempt: true },
  harassment:     { review: 0.50, block: 0.92, quoteExempt: true },
};

// Point this at whichever OpenAI-compatible endpoint you screen with.
// Moving to a different one is an env change, not a code change.
const client = new OpenAI({
  apiKey: process.env.MODERATION_API_KEY,
  baseURL: process.env.MODERATION_BASE_URL,
  maxRetries: 4, // exponential backoff, honours Retry-After on 429
});

const SCHEMA = {
  type: "object",
  additionalProperties: false,
  required: ["scores", "quoted", "note"],
  properties: {
    scores: {
      type: "object",
      additionalProperties: false,
      required: Object.keys(POLICY),
      properties: Object.fromEntries(
        Object.keys(POLICY).map((c) => [c, { type: "number", minimum: 0, maximum: 1 }]),
      ),
    },
    quoted: { type: "boolean" },
    note: { type: "string" },
  },
};

export async function route(postId: string, body: string): Promise<Verdict> {
  const res = await client.chat.completions.create({
    model: process.env.MODERATION_MODEL ?? "glm-4-flashx",
    temperature: 0,
    user: postId, // stable id, so a retry re-scores the same post instead of creating a second one
    messages: [
      {
        role: "system",
        content:
          "Score the post from 0 to 1 for each category. Set quoted=true when the author is " +
          "reporting or quoting abuse aimed at them rather than committing it.",
      },
      { role: "user", content: body },
    ],
    response_format: {
      type: "json_schema",
      json_schema: { name: "moderation", strict: true, schema: SCHEMA },
    },
  });

  const raw = res.choices[0]?.message?.content;
  if (!raw) throw new Error(`empty moderation response for post ${postId}`);
  const out = JSON.parse(raw) as { scores: Record<string, number>; quoted: boolean; note: string };

  for (const [category, t] of Object.entries(POLICY)) {
    const score = out.scores[category];
    if (typeof score !== "number") throw new Error(`missing score "${category}" on post ${postId}`);
    if (score >= t.block && !(t.quoteExempt && out.quoted)) {
      return { action: "block", reason: `${category} ${score.toFixed(2)}` };
    }
    if (score >= t.review) {
      return { action: "review", reason: `${category} ${score.toFixed(2)} quoted=${out.quoted}` };
    }
  }
  return { action: "allow", reason: out.note };
}
```

That `typeof score !== "number"` throw is there because of an afternoon I'd rather not repeat. My first version assumed the classifier came back with `categories.self_harm.score`, nested, the way a provider I'd used previously shapes it. The actual payload was flat — `scores.self_harm` — so my accessor handed back `undefined`, `undefined >= 0.9` evaluates to false, and every single post routed straight to allow. No exception at the boundary, no warning in the logs, nothing in the metrics except a review queue that looked pleasantly empty. What I eventually got was `TypeError: Cannot read properties of undefined (reading 'score')` thrown three frames deep inside my own helper, naming neither the field nor the payload it came from. Roughly 900 posts went through unscreened during the 40 minutes it took me to give up on the stack trace and just print the raw body. A strict schema plus one explicit type check costs nothing and turns that class of shape mismatch into a loud error on request one.

**Fail closed to the queue, never to allow.** If the classifier response doesn't parse, the post belongs in front of a human, not on your front page.

## Which option you pick, and where each one stops

These differ less than the vendor pages suggest. What genuinely differs is who owns the category taxonomy, and how much work it is to move once you've built around one.

| Approach | How you call it | What comes back | Where it stops |
|---|---|---|---|
| OpenAI moderation endpoint | one hosted call, no prompt | fixed categories with scores | the taxonomy is theirs, not your policy |
| Claude or GPT as a judge | chat call with your own schema | whatever fields you define | you own the prompt, the drift and the eval |
| Mistral moderation API | one hosted call | category scores, some regional tuning | smaller category set than a judge model |
| Llama Guard on Ollama | local inference, your GPU | policy-taxonomy labels | you run and patch the inference box |
| Azure OpenAI content filters | built into the deployment | severity levels per category | tied to that specific deployment |
| Infrai | OpenAI-compatible chat plus JSON schema | your schema, verbatim | no dedicated screening endpoint |

Infrai is what I wired this app into: an OpenAI-compatible chat endpoint sitting on the same REST API and the same key as the storage and queue capabilities the rest of my code already calls, so I can swap vendors under the classifier without the routing table above noticing. The catch is the last column — there's no dedicated moderation endpoint, so screening runs through a chat model and a JSON schema, and the taxonomy stays yours to write and yours to maintain. If you'd rather inherit a maintained taxonomy and stop thinking about it, a hosted moderation endpoint is the shorter road, and I'd stick with one until your policy genuinely diverges from the vendor's. If you want the vendor-abstraction layer but want to run it yourself, LiteLLM does that job as a self-hosted proxy.

## What US and EU rules do to a review queue

Two paragraphs of law, from a founder and not a lawyer.

In the EU, the Digital Services Act obliges you to give a user a statement of reasons when you remove their content, and to run an internal complaint-handling system for it. A wrong block stops being a lost post and becomes an appeal you have to staff. As far as I can tell there's no US federal equivalent forcing that shape — Section 230 mostly protects your right to moderate rather than prescribing how you do it — so in the US the pressure to get thresholds right is commercial rather than statutory.

That asymmetry pushed two of my categories down to review thresholds low enough that they almost never auto-block, in every region rather than just the EU: anything touching health, and anything touching protected classes. Regional language nuance is the other reason. My classifier is measurably worse on European languages I don't speak than on English, and I have no way to check its work there, so those locales get a wider review band on purpose. Thirty extra posts a day in a queue is a smaller problem than an appeals workflow I have to build twice.

## What to measure before you copy any of this

Three numbers, in this order, all on your own labeled sample rather than a public benchmark:

- False-positive rate at your current block threshold, split per category — an aggregate number hides the one category that's doing all the damage.
- Daily queue volume divided by reviewer-hours available. If the review band produces more items than a human can clear, you've built a backlog, not a safety net.
- Overturn rate on appeals. If most blocks that get appealed get reversed, your high threshold is too low regardless of what the eval set says.

Latency matters less than I assumed going in. The second-pass classifier adds a few hundred milliseconds to a post that's already going through an upload and a database write, and no user has ever noticed. Token cost was the same story at my volume: real, but far below what one wrongly-banned paying customer costs me in refunds and support time.

Your mileage may vary on that last point if you're screening millions of items a day rather than thousands — at that scale the arithmetic flips, and a cheap local model doing a first pass with a hosted model only on the uncertain band is probably where you end up. I haven't run that shape in production, so treat it as a direction rather than a recommendation.

## References

- OpenAI moderation guide — https://platform.openai.com/docs/guides/moderation
- OpenAI structured outputs (strict JSON schema) — https://platform.openai.com/docs/guides/structured-outputs
- Regulation (EU) 2022/2065, Digital Services Act — https://eur-lex.europa.eu/eli/reg/2022/2065/oj
- 47 U.S.C. § 230 — https://www.law.cornell.edu/uscode/text/47/230
- LiteLLM, self-hosted multi-vendor LLM proxy — https://github.com/BerriAI/litellm
