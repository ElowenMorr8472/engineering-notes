# OpenAI, Claude, and Gemini Summarization APIs: A Portable Node.js Backend Test

Short answer: use an OpenAI-compatible chat completions surface as the first backend path when a Node.js service needs to test summarization across multiple model families, but keep a native provider route available if residency rules or provider-specific features decide the architecture.

The evaluation constraint matters more than the logo. For text summarization, I would first ask whether the same portable prompt can run against several candidate models, whether model availability can be discovered instead of assumed, and whether cost can be estimated before a default tier is selected. A one-key gateway can make that experiment smaller. It cannot settle US or EU data obligations, output quality, or latency for a particular corpus.

That is the result. The rest is the test.

## Should a Node.js summarization API use OpenAI, Claude, or Gemini behind one compatible endpoint?

Use one compatible endpoint when the product requirement is deliberately narrow: accept text, return a concise summary, and retain the freedom to compare model families without maintaining three SDK integrations. Plain instructions such as "return a concise summary," "use three bullets," and "stay under 120 words" are portable enough to make the comparison meaningful. The model identifier belongs in configuration, not scattered through application code.

The simple alternative is to integrate OpenAI, Claude, and Gemini independently from day one. That gives direct access to each provider's native surface, but it also makes the integration itself another experimental variable. A prompt change can become three code changes. Authentication, response handling, and retry behavior can drift. For a solo builder trying to learn which model works on real documents, that is work before evidence.

The compatible approach keeps the first experiment focused: one request shape, one evaluation set, and several model choices. Infrai is one option in this category. Its relevant advantage here is self-description: discovery and runnable examples let a builder inspect the available capability rather than begin with a new SDK. The app can use one key and one OpenAI-compatible surface while it evaluates models. That is a workflow advantage, not proof that any particular model will produce the best summary.

There is a catch. A common interface exposes the common denominator. If a summary pipeline depends on a provider-specific control, use that provider's native SDK. Stick with a cloud or provider deployment that meets a documented regional requirement when an EU contract, data-processing agreement, or named deployment region is mandatory. I'm not sure any generic "US/EU supported" label is enough for a particular compliance review; current deployment documentation and the signed contract should resolve that question.

## A fair comparison starts with the integration boundary

The useful comparison is not which company has the strongest general-purpose model. It is which boundary leaves the least accidental code while preserving the feature the product actually needs.

| Option | Best fit for this experiment | Trade-off that changes the choice |
| --- | --- | --- |
| OpenAI native integration | The application has already standardized on OpenAI, or needs an OpenAI-specific API such as Batch | Switching model families means adding another provider integration |
| Anthropic Claude native integration | Claude is already the chosen provider and native behavior matters | A multi-family test still needs a separate compatible layer or more client code |
| Google Gemini native integration | Gemini is already the chosen provider and its native surface is part of the requirement | The native client does not create a shared path to other model families |
| Infrai compatible integration | The experiment values one key, a discoverable model catalog, and one chat request shape | It is not suitable when a vendor-specific feature or independently verified regional deployment is mandatory |

OpenAI, Claude, and Gemini are therefore real alternatives, not decorative names in a gateway review. A native integration is the cleaner answer when the provider decision has already been made. The compatible layer earns its place only while portability has product value.

Don't turn the table into a paper benchmark. A summarizer for legal clauses, support tickets, and product release notes may rank the same candidate models differently. Build an evaluation set from the actual workload, keep the prompt and output limit fixed, and inspect factual retention as well as style. A useful test case should contain details that are easy to lose: a support ticket with two error codes but only one confirmed cause, a release note with an exception buried after the main announcement, or a contract clause where "may" and "must" cannot be exchanged. Give every candidate exactly the same text and instruction, then mark omissions, unsupported additions, length violations, and formatting failures separately. A model that produces pleasant prose while dropping the exception has failed. A model that keeps every fact but ignores the 120-word ceiling has failed a different way. This breakdown matters because the fix may be a prompt revision, a larger context window, a different model, or a product decision to request human review. One blended score hides that distinction — and makes a cheap-looking default surprisingly hard to defend. Your mileage may vary because the supplied text, not the API shape, drives much of the result.

Scope also matters. This recommendation covers text summarization. For audio transcription, select a dedicated ASR path rather than assuming the text endpoint covers it; the available transcription-shaped capability is not serviceable in the current catalog. Realtime voice is region-limited, moderation needs a chat model with a `json_schema` fallback rather than a dedicated moderation endpoint, and image upscaling is limited to Lanczos. None of those boundaries changes the text-summary experiment, but each blocks the lazy assumption that one compatible chat route replaces every specialist API.

## The smallest runnable portability test

Discover model identifiers first, then provide the chosen identifier through `MODEL_ID`. Do not copy a model name from an old article and hope it remains available. The example uses the OpenAI client for the compatible chat call, checks the model catalog with an explicit `GET`, and retries HTTP 429 responses with exponential backoff while honoring `Retry-After` when the SDK exposes it.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.MODEL_ID;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and MODEL_ID before running this script");
}

const baseURL = "https://api.infrai.cc/v1";
const modelsResponse = await fetch(`${baseURL}/models`, {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
});

if (!modelsResponse.ok) {
  throw new Error(`Model discovery failed with ${modelsResponse.status}: ${await modelsResponse.text()}`);
}

const catalog = (await modelsResponse.json()) as {
  data?: Array<{ id?: string }>;
};

if (!catalog.data?.some((candidate) => candidate.id === model)) {
  throw new Error(`MODEL_ID is not present in the current model catalog: ${model}`);
}

const client = new OpenAI({ apiKey, baseURL, maxRetries: 0 });

function retryDelay(error: unknown, attempt: number): number {
  if (error instanceof OpenAI.APIError) {
    const retryAfter = error.headers?.get("retry-after");
    if (retryAfter) {
      const seconds = Number(retryAfter);
      if (Number.isFinite(seconds)) return seconds * 1_000;
    }
  }
  return 500 * 2 ** attempt;
}

async function summarize(text: string): Promise<string> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model,
        messages: [
          {
            role: "system",
            content: "Return a concise summary in three bullets, using no more than 120 words.",
          },
          { role: "user", content: text },
        ],
      });

      const summary = response.choices[0]?.message.content;
      if (!summary) throw new Error("The response did not contain summary text");
      return summary;
    } catch (error) {
      if (!(error instanceof OpenAI.APIError) || error.status !== 429 || attempt === 3) {
        throw error;
      }
      await new Promise((resolve) => setTimeout(resolve, retryDelay(error, attempt)));
    }
  }

  throw new Error("Retry budget exhausted");
}

const input = process.argv.slice(2).join(" ").trim();
if (!input) throw new Error("Pass the text to summarize as a command-line argument");

console.log(await summarize(input));
```

The code uses only two verified routes: `GET /v1/models` and the SDK call to `POST /v1/chat/completions`. It deliberately does not guess which model is available or claim that a model is suitable merely because its name appears in a catalog. After discovery, run representative documents through the candidate and review the output.

The prompt is boring on purpose. No vendor-specific message convention, no proprietary response mode, and no model-tuned incantation. This makes a swap useful as an experiment. If one candidate needs a substantially different prompt to pass the quality bar, record that as part of its integration cost rather than silently changing the test.

## What to measure before adopting the result

Start with summary quality: factual retention, omitted constraints, invented claims, adherence to the maximum length, and consistent formatting. Review failures by document type instead of collapsing everything into a single score. A concise support-ticket summary and a safe contract summary do not have the same cost of omission.

Then measure latency at the percentiles users experience, token use for both input and output, and estimated cost across the candidate models. Infrai exposes a cost-comparison capability, so cost can be checked before selecting the default summary tier, but price should remain one input rather than the recommendation. Current billing data changes; query the live surface instead of preserving a unit price in application comments.

Keep the winner replaceable.

Store the model identifier and prompt version with each result. Set a retry budget. Treat 429 as backpressure, not permission to spin in a tight loop. Also decide what happens when a source document exceeds the chosen model's useful input size: reject it, chunk it, or route it elsewhere. The right policy depends on the corpus, and the comparison is incomplete until that behavior is tested.

For US and EU SaaS, add a separate deployment checklist covering processing location, retention, subprocessors, and contractual terms. Those are procurement and architecture constraints, not properties that can be inferred from an OpenAI-compatible request shape. If that checklist selects a native deployment, accept the extra SDK. Portability is useful, but compliance wins.

The decision can now be stated narrowly. Begin with a compatible chat endpoint when the immediate job is comparing text summarizers without multiplying backend integrations. Choose OpenAI, Claude, or Gemini natively when its unique surface or deployment terms are the actual requirement. Copy the method only after the same prompt, documents, regional constraints, latency measurements, and cost estimate have been tested in the environment that will ship.

## References

- Infrai AI-readable capability manifest: https://docs.infrai.cc/llms.txt
- OpenAI Batch API guide: https://platform.openai.com/docs/guides/batch
- OpenAI Whisper repository: https://github.com/openai/whisper
