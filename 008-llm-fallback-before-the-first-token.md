# LLM fallback before the first token

*Les Grimoires devlog · September 2026*

**TL;DR**

- We keep a ranked list of LLM candidates (provider + model + key), editable from an admin page, no redeploy.
- On quota, overload, no credits or bad key, a turn **silently moves to the next candidate**, as long as nothing has been streamed yet.
- Trick: **hold the SDK's writes** to the HTTP response until the first token, then replay them.
- Every fallback is logged and attached to the message. A model change mid-game must be explainable.

## Why

Free tiers and cheap models fail often: rate limits, overloads, credits gone. A player shouldn't see "error" when another model could answer.

But once text is streaming, switching model means the player sees two answers glued together. So: **fallback only before the first token.**

## The loop

```js
const RETRYABLE = new Set(['rate_limit', 'overloaded', 'insufficient_credits', 'auth']);

export async function withFallback(reasoningEffort, run, onFailure) {
  let lastError;
  for (const candidate of llmCandidates()) {
    try {
      return await run(buildModel(candidate, reasoningEffort));
    } catch (err) {
      lastError = err;
      const { code, message } = classifyLLMError(err);
      if (!RETRYABLE.has(code)) throw err;   // our bug, not the model's: stop here
      onFailure?.({ provider: candidate.provider, model: candidate.model, code, message });
    }
  }
  throw lastError;
}
```

- Only **model-side** errors move on. A bad prompt or a bug is rethrown at once, instead of burning every key.
- Order: by priority (cascade) or random (spread the load).

## Holding the stream

We use the Vercel AI SDK's `streamText` with Fastify. The SDK writes straight to `reply.raw`. We wrap `write`, `writeHead` and `end`:

```js
let live = false, dropped = false, held = [];

const hold = (fn, args, ret) => {
  if (live) return fn(...args);           // candidate answered: write for real
  if (!dropped) held.push([fn, args]);    // not yet: keep it
  return ret;
};

raw.write     = (chunk, ...a) => hold(rawWrite, [chunk, ...a], true);
raw.writeHead = (...a)        => hold(rawWriteHead, a, raw);
raw.end       = (...a)        => hold(rawEnd, a, raw);

const flush = () => {
  live = true;
  reply.hijack();
  for (const [fn, args] of held) fn(...args);
  held = [];
};
```

And in `streamText`:

```js
onChunk() { if (!live) flush(); }   // first chunk: this candidate holds, no way back
```

If the candidate fails before its first chunk, its held writes are dropped and the next one starts clean.

## Details that bit us

- **Keep-alive pings.** Mobile networks cut idle connections, so we send an SSE comment every 15 s. But **only once live**: before that, pings would be buffered and replayed, and early headers would kill the JSON error response.
- **Tab closed mid-turn.** Without an `AbortController` on `close`, the model keeps generating into the void. Billed tokens for nobody.
- **The model you asked for isn't always the one served.** With an OpenRouter preset, read `response.modelId` in `onStepFinish` (`onFinish` runs too late to fill the metadata).
- **Some endpoints refuse `reasoning: none`.** Retry the same model at `low` before moving on.

## Make every switch visible

A silent fallback is a model change nobody can explain later ("why did the tone change?"). So each refused candidate:

- goes into the message metadata (`fallbacks`);
- is logged as an error in the call journal;
- shows up in our session checker and in the debug panel, next to the model actually served.

## The limit

**Mid-stream failure goes to the client.** Replaying it would mean replaying the whole turn: tool calls, state changes, dice. Not worth it.

## Takeaways

- Fallback is cheap and safe **before** the first token, messy after. Draw the line there.
- Hold the transport, not the SDK: wrapping `write`/`end` works with any streaming library.
- Only retry errors that are the model's fault.
- Log every switch. Invisible fallbacks become unexplained bugs.

---

*Les Grimoires is an AI game master for gamebooks. Closed alpha (in French): [lesgrimoires.fr](https://lesgrimoires.fr)*
