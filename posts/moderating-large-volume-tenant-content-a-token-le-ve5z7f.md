# Moderating Large-Volume Tenant Content: A Token Ledger for Batch Review

Short answer: the cheapest reliable way to moderate a large backlog is to put a token ledger and a hard review-queue budget in front of batch LLM classification, then spend human attention only on uncertain tenant content.

For a property-management assistant, this is more useful than starting with a model comparison. A resident message may be a routine question, a threat, a privacy-sensitive disclosure, or an urgent maintenance report. The assistant still needs to answer ordinary questions from a private knowledge base, but moderation should decide what is safe to classify automatically, what needs a person, and what must stay out of retrieval until reviewed.

The ledger is the control point. It records the complete prompt, the allowed output, the policy version, and the result of every retry. That turns an apparently cheap bulk job into a bounded engineering decision.

## How should a large-volume batch use token counting before a review queue?

Start by separating intake from answer generation. Store the tenant message with a stable ID, remove irrelevant markup, and attach only the property and policy context needed for classification. Count that full request before dispatch. A classifier should return a small, validated decision such as `allow`, `review`, or `hold`, plus a short reason and the source ID. It should not write directly to enforcement or to the private knowledge base.

The flow is straightforward: select a bounded batch, estimate its maximum input and output tokens, submit it, validate one result per ID, and write uncertain results to an idempotent review queue. Only after a moderation decision should an allowed message enter retrieval and answer generation. This ordering keeps an angry or private message from being treated as ordinary search context, while urgent cases can follow a separate human escalation path.

No magic cutoff.

The boundary between `allow` and `review` depends on the cost of a false allow, the cost of a false hold, and the number of reviewers available. I'm not sure a confidence value can set that boundary by itself. Use a labeled sample of actual tenant messages, compare model decisions with reviewer decisions, and record the policy version used for each result.

## A runnable token ledger and queue pass

Before sending work to any provider, I want a local calculation that answers two questions: what is the upper-bound cost of this batch, and how much work will land in front of a reviewer? The tokenizer is deliberately outside this example because tokenization varies by model. Pass in counts produced by the tokenizer selected for the actual request.

```ts
type BatchItem = {
  id: string;
  inputTokens: number;
  maxOutputTokens: number;
};

type Rates = {
  inputPerMillion: number;
  outputPerMillion: number;
};

type ModerationResult = {
  id: string;
  decision: "allow" | "review" | "hold";
  reason: string;
};

function estimateMaximumCost(items: BatchItem[], rates: Rates): number {
  const totals = items.reduce(
    (sum, item) => ({
      input: sum.input + item.inputTokens,
      output: sum.output + item.maxOutputTokens,
    }),
    { input: 0, output: 0 },
  );

  return (
    (totals.input / 1_000_000) * rates.inputPerMillion +
    (totals.output / 1_000_000) * rates.outputPerMillion
  );
}

function selectReviewItems(
  results: ModerationResult[],
  maximumQueueSize: number,
): ModerationResult[] {
  const unique = new Map<string, ModerationResult>();

  for (const result of results) {
    if (result.decision === "review" || result.decision === "hold") {
      unique.set(result.id, result);
    }
  }

  const selected = [...unique.values()];
  if (selected.length > maximumQueueSize) {
    throw new Error("Review queue budget exceeded");
  }
  return selected;
}

const items = JSON.parse(process.env.BATCH_ITEMS_JSON ?? "[]") as BatchItem[];
const results = JSON.parse(
  process.env.MODERATION_RESULTS_JSON ?? "[]",
) as ModerationResult[];
const rates: Rates = {
  inputPerMillion: Number(process.env.INPUT_RATE_PER_MILLION),
  outputPerMillion: Number(process.env.OUTPUT_RATE_PER_MILLION),
};

if (
  !Number.isFinite(rates.inputPerMillion) ||
  !Number.isFinite(rates.outputPerMillion)
) {
  throw new Error("Set current input and output rates");
}

console.log({
  estimatedMaximumCost: estimateMaximumCost(items, rates),
  reviewItems: selectReviewItems(
    results,
    Number(process.env.MAXIMUM_REVIEW_ITEMS ?? 0),
  ),
});
```

This is a budget gate, not a promise that the estimate equals the invoice. It uses the maximum permitted output, so actual usage can be lower. Record the estimate beside the batch ID before submission and actual input and output usage afterward. Include the policy prompt, schema framing, and expected retry volume; counting only the tenant's words is how a small-looking batch gets an unexpectedly large bill.

The trade-off is deliberate: a compact schema and a maximum queue size make the run easier to control, but they can hide nuance if the policy is too narrow. Keep the raw source available to an authorized reviewer and expand the schema only when review data shows a recurring decision the current contract cannot express.

For example, a resident asks, “The boiler is noisy again. Can someone come tomorrow?” The moderation record should retain the message ID and policy version, classify the content, and send an allowed item to retrieval over the building's private handbook. A message containing a medical detail or a threat should route to the review queue instead of becoming searchable context. The answer service then receives a reviewed, scoped request rather than an opaque raw transcript.

The useful implementation detail is what happens between those two ends. An intake worker first copies the message into a short-lived classification record and freezes the policy version for that record; it should not silently reclassify an old message when the rubric changes. A batch builder then attaches the building identifier, the minimum relevant policy text, and the stable content ID, while excluding unrelated conversation history that would increase tokens without helping the decision. The ledger is written before dispatch, so an operator can see the expected exposure even if the batch is delayed. When results arrive, a validator checks that each selected ID appears once, that no unknown ID was introduced, and that the decision is one of the permitted values. An allowed message can proceed to retrieval with its scope intact. A review or hold result gets a queue row containing the original reference, the reason, and the state transition needed by the reviewer. If a reviewer changes the decision, store that human outcome beside the model outcome rather than overwriting the evidence. That makes later threshold evaluation possible and prevents a policy edit from erasing why the original route was chosen. The same records also make retries inspectable: a repeated delivery is a new transport event, not a new moderation case. This is where the cost estimate becomes useful operational data instead of a spreadsheet guess.

Stable IDs matter here. If a rate-limited request is retried, the same ID should update the same moderation record and queue item. Otherwise, one resident message can occupy several review slots. HTTP retry semantics are a protocol concern, too: use an idempotency strategy in the application and honor server guidance such as `Retry-After`. Don't let a retry loop inflate both token usage and reviewer workload.

## Where the budget leaks in a property-management assistant

The obvious term is model inference: input tokens multiplied by the current input rate, plus output tokens multiplied by the current output rate. The less obvious term is repeated context. A long safety rubric pasted into every small request can outweigh the tenant text, especially when the backlog contains short messages.

| Control | Record | Failure it exposes |
| --- | --- | --- |
| Token ledger | Prompt, output limit, estimate, actual usage | A policy or retry change that expands the batch |
| Stable identity | Content ID, batch ID, policy version | Duplicate or stale moderation decisions |
| Queue budget | Queue size, age, reviewer status | More human work than the team can clear |
| Result validation | Expected IDs, returned IDs, allowed enums | Partial or malformed batch output |

Keep the policy compact, but don't remove the context a reviewer needs. Version the rubric and schema. If the ledger shows a jump after a policy revision, the team can inspect a concrete change instead of arguing about a monthly total.

The queue has a budget, too. A batch that is inexpensive to classify can still be operationally expensive if it creates more cases than reviewers can clear. Track queue depth, oldest-item age, batch age, parse failures, missing IDs, and reviewer agreement next to token totals. Backpressure non-urgent imports when the queue grows; urgent maintenance or safety reports should use their own escalation path.

One practical rule: sample automatic allows and holds as well as borderline cases. Reviewing only escalations hides confidently wrong decisions. Your mileage may vary by property, language mix, and policy, so calibrate the routing threshold against local review data rather than copying a number from another application.

## What should ship before automatic enforcement and retrieval?

Ship the observable path first. Persist the content ID, batch ID, policy version, schema version, token estimate, actual usage, moderation decision, reviewer status, and final action. Validate that every selected input has exactly one known result and that every enum is valid before any retrieval or enforcement step.

Then test the failure modes deliberately: duplicated delivery, missing results, malformed JSON, delayed reviewers, a changed policy, and a retry after rate limiting. A replay must not create a second queue item or apply an enforcement action twice. A distribution shift in labels should pause automatic action until a reviewer checks whether traffic, source data, or policy changed.

This approach is not suitable when nobody can review samples or respond to a growing queue. The trade-off is that a strict queue budget may leave some content waiting instead of producing an automatic decision. In that case, keep moderation advisory, narrow the content surface, or choose a simpler deterministic filter before adding batch classification. The model call is the easy part. The accountability loop is the product.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
