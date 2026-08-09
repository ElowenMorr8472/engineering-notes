# Diagnosing Long Recording Failures in Node.js: File Size, Multipart, and Backoff

A long recording crosses several time budgets before speech becomes text, so one larger client timeout rarely settles the design.

Short answer: treat the audio upload, transcription request, and result delivery as separate failure domains; retry only an operation whose outcome is known, and use a stable operation key when a retry could duplicate work.

That choice is less convenient than wrapping a file in `FormData` and awaiting one `fetch`. It is also easier to reason about after a caller disconnects. For a small, bounded clip, the direct multipart request is still a sensible baseline. For a large file or a recording with unpredictable duration, I would put the bytes in durable storage first and let a worker own transcription.

## Start with the boundary that ended the request

An API timeout is a symptom, not a location. The browser or Node.js caller can stop waiting. An application gateway can reject a body before the handler reads it. The upload can stop midway. The speech operation can continue after the caller loses the response. These outcomes look similar at the call site, but they don't share a safe retry policy.

Record the request phase alongside the error: preparing the multipart body, sending bytes, waiting for acceptance, processing audio, or fetching the transcript. Also record the operation key, attempt, elapsed time, uploaded byte count, declared file size, and terminal state. Those fields let an operator distinguish “nothing was accepted” from “the answer may exist, but this caller didn't receive it.” I'm not sure which timeout deserves adjustment until that boundary is visible.

Keep it concrete.

If the handler never observes the request, changing the speech worker's deadline cannot help. If all bytes arrive and the caller times out while waiting for transcription, resending the same multipart body may start the work again. If a worker loses its response, the correct next action depends on whether the remote operation can be found by a stable identifier. The operational question is always the same: what durable state exists at the moment uncertainty begins?

This is also where file size and audio duration must stay separate. File size affects upload and buffering pressure; duration affects how much audio must be processed. Compression can move those two measurements in different directions, so neither is a reliable substitute for the other. Capture both from the real workload instead of setting a timeout from one convenient sample.

## How should Node.js fetch handle multipart file size and retry backoff?

Use `fetch` as a transport boundary, not as the job ledger. Give each attempt an explicit deadline, retain the deepest available error cause, and return an outcome that forces the caller to choose between retry and reconciliation. Don't turn every rejection into the same “try again” branch.

The focused TypeScript example below models that rule. The endpoint is intentionally pseudonymous; replace it only with a documented endpoint whose idempotency behavior you have verified. `unknown` means the connection ended after submission may have started, so the worker must reconcile rather than automatically upload the long recording again.

```ts
type SubmitOutcome =
  | { kind: "accepted"; jobId: string }
  | { kind: "retryable"; reason: unknown }
  | { kind: "unknown"; reason: unknown };

type SubmitInput = {
  audio: Blob;
  filename: string;
  operationKey: string;
  deadlineMs: number;
};

export async function submitRecording(input: SubmitInput): Promise<SubmitOutcome> {
  const body = new FormData();
  body.set("audio", input.audio, input.filename);

  try {
    const response = await fetch("https://speech.example/transcriptions", {
      method: "POST",
      headers: { "Idempotency-Key": input.operationKey },
      body,
      signal: AbortSignal.timeout(input.deadlineMs),
    });

    if (!response.ok) {
      return { kind: "unknown", reason: new Error(`status ${response.status}`) };
    }

    const payload = (await response.json()) as { jobId: string };
    return { kind: "accepted", jobId: payload.jobId };
  } catch (reason) {
    return { kind: "retryable", reason };
  }
}

export function backoffMs(attempt: number, random: () => number): number {
  const capMs = 30_000;
  const windowMs = Math.min(capMs, 500 * 2 ** attempt);
  return Math.floor(random() * windowMs);
}
```

The split between `retryable` and `unknown` cannot be inferred perfectly from a generic thrown error — the application needs transport evidence and a documented remote contract. In a real adapter, classify only failures that prove submission did not begin as retryable. Everything else moves to reconciliation. The backoff function uses a growing randomized window and a cap as application policy; tune its inputs from observed recovery behavior and the service contract, not from a copied constant.

There is a catch. An idempotency header has no value unless the receiver promises to honor it for this operation, and a pseudonymous example cannot make that promise on a real service's behalf. If the API exposes a stable job identifier, persist it before polling. If it doesn't expose an idempotent submission or lookup contract, put uniqueness in your own job table and avoid concurrent submissions for the same operation key. A digest can help identify identical bytes, but processing settings belong in the identity too because the same recording may legitimately need a different transcript configuration.

## Move long audio out of the synchronous path

The ship-first version has three states worth preserving: stored audio, requested transcription, and available transcript. Upload the file once, validate the stored object, then enqueue a compact job containing an immutable object reference and operation key. A worker submits the audio or a storage reference according to the speech API's documented contract. It persists the remote job identifier before any status loop begins.

Now a dropped client connection doesn't erase the input. A repeated queue delivery can resolve to the existing local operation. Polling can have a different budget from uploading, and downstream prompt work starts only from a terminal transcript. That last boundary matters for cost: duplicated or partial text can flow into later model calls even when the speech operation was the original failure. The useful unit of accounting is the completed operation, with upload attempts and downstream usage attached to it.

The simple multipart path is not wrong. Stick with it when recordings are predictably small, every intermediary accepts the body, the caller can wait for the complete operation, and duplicate submission is controlled. Storage plus a queue is not suitable when that operational machinery costs more than the failure it prevents. It adds retention rules, cleanup, access control, queue visibility, and reconciliation work — all of which must be owned.

For long recordings, though, separating data transfer from control flow gives each failure a recoverable boundary. It also makes deployment less tense: a web process can restart without taking the only copy of an upload with it, while workers can drain under their own deadline policy.

## Measure before changing the timeout

Test the path with the largest accepted encoded file, the longest accepted recording, and a slow upload. Interrupt each phase deliberately: before any bytes are accepted, midway through transfer, after acceptance but before a response reaches the worker, and while the transcript is being retrieved. The acceptance criterion is not merely “a retry succeeded.” It is one terminal local operation, one attributable transcript, and no second submission while the first outcome remains unknown.

Watch distributions rather than one successful run: upload elapsed time, processing elapsed time, queued time, bytes, audio duration, attempts per operation, reconciliations, and terminal failures. Keep recognition evaluation beside transport evaluation. A change that shrinks an audio file can alter the text, and a transcript that arrives reliably can still be poor input for prompting or reranking.

This is the decision rule I would ship: increase a timeout only after locating the boundary and confirming that waiting longer fits the caller's budget. Move to durable upload and asynchronous work when the result must survive disconnection or the recording approaches an intermediary limit. Add retry backoff only after defining idempotency and the unknown-outcome state.

No magic number fixes this.

## References

- https://docs.cohere.com/docs/rerank-overview
- https://www.promptingguide.ai

## Sources

- https://docs.cohere.com/docs/rerank-overview
- https://www.promptingguide.ai
