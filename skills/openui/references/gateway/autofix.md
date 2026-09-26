# Repair Output with Autofix

Autofix repairs invalid OpenUI Lang after any model generates it. The app keeps its provider, inference setup, and agent framework. Only invalid output reaches the API, and failures are tracked in Reliability Monitoring automatically.

Use Autofix for self-hosted apps and any provider or framework that does not generate through Gateway. Generation through Gateway with `generateSystemPrompt({ cloud: true, library })` is already corrected in the stream; do not add Autofix on top of it.

## Set Up the Server Package

`@openuidev/server` validates locally and calls Autofix only when the output is invalid. It needs a server-side `THESYS_API_KEY` ([authentication handoff](quickstart.md#complete-authentication-with-the-user)) and the spec from `openui generate --spec` for the same library the client renders; the spec's `schema` field powers the local check.

```ts
// lib/autofix.ts
import { createAutofix } from "@openuidev/server/openai"; // or "@openuidev/server/vercel"
import library from "./generated/library-spec.json";

export const autofix = createAutofix({
  apiKey: process.env.THESYS_API_KEY!,
  library,
});
```

The `/openai` import returns `autofix.completions` for Chat Completions; the `/vercel` import returns `autofix.ai` for Vercel AI SDK UI message streams.

## Wrap a Stream

Chat Completions with the OpenAI SDK, paired with `openAIAdapter()` on the client:

```ts
import OpenAI from "openai";
import { autofix } from "@/lib/autofix";

const model = new OpenAI();

export async function POST(request: Request) {
  const { messages } = await request.json(); // authenticate and validate first
  const source = await model.chat.completions.create(
    { model: process.env.OPENAI_MODEL!, messages, stream: true },
    { signal: request.signal },
  );

  return autofix.completions
    .stream({ stream: source, messages, signal: request.signal })
    .toResponse();
}
```

Vercel AI SDK, paired with `useChat` or `vercelAIAdapter()`:

```ts
import { streamText, toUIMessageStream } from "ai";
import { autofix } from "@/lib/autofix";

export async function POST(request: Request) {
  const { messages } = await request.json();
  const result = streamText({ model, system: systemPrompt, messages, abortSignal: request.signal });

  return autofix.ai
    .stream({ stream: toUIMessageStream({ stream: result.stream }), signal: request.signal })
    .toResponse();
}
```

The wrapper forwards provider events unchanged and inserts the corrected program before the stream completes. Consume the wrapper once, through either `chunks` (native SDK events) or `toResponse()` (SSE for the matching client). Its `result` promise settles afterward: `null` when no repair was needed, otherwise the Autofix result. Persist `result.content` when present instead of the joined deltas. A repair that fails throws an `AutofixError` with `code: "fix_failed"`.

## Fix a Completed Generation

When the app already holds the full text:

```ts
const result = await autofix.completions.fix({ generation, messages, signal }); // or autofix.ai.fix
if (result.content !== null) {
  // "already_valid" or "fixed": render and persist result.content
} else {
  // "fix_failed": fall back to text or an error state; result.unfixedErrors lists the problems
}
```

`messages` is the conversation before the generation. The package trims it to the API's context limit.

## Call the API Directly

Without the package, parse the settled output and call the API only when it is invalid (see [reliability.md](../reliability.md#correct-invalid-output) for the validity check).

- `POST https://api.thesys.dev/v1/autofix`, or point an OpenAI SDK at base URL `https://api.thesys.dev/v1/autofix`. Authenticate with `Authorization: Bearer $THESYS_API_KEY`.
- `messages`: first a `system` turn containing `generateSystemPrompt({ cloud: true, library })`, then the conversation for context, and last an `assistant` turn with the generation to fix.
- `model` is ignored, and `stream: true` returns `400`.
- The response is a chat completion. `choices[0].message.content` is the whole corrected generation (not a patch), and `fix_summary.status` is `already_valid`, `fixed`, or `fix_failed`, with `fixed_errors` and `unfixed_errors` listed.

Limits: the generation can be up to 100,000 characters. Context can hold up to 20 usable turns and 8,000 characters of text; larger context returns `400`, so trim older history first. Each generation gets up to two repair attempts. Error codes match the browser parser's: `unknown-component`, `missing-required`, `null-required`, `type-mismatch`, `excess-args`, `inline-reserved`, `incomplete`, `unresolved`, `orphaned`, and `missing-root`.

## Cost and Data

A call that runs a fix is charged a flat amount from Gateway credits; an already-valid generation is free. Rates are on [Pricing](https://www.openui.com/docs/gateway/pricing-credits). The conversation sent is passed to the correction model and is not stored beyond request logs.

## Verify

1. The spec passed to `createAutofix` is regenerated from the library the client renders.
2. A valid response streams unchanged and reports no repair.
3. A deliberately invalid canned response comes back fixed, and the persisted message is the fixed content.
4. A `fix_failed` result shows the app's fallback, not a broken render.
5. Requests appear in [Reliability Monitoring](https://console.thesys.dev/reliability).
6. `THESYS_API_KEY` stays on the server.

## First-Party References

- `https://www.openui.com/docs/autofix`
- `https://www.openui.com/docs/api-reference/server`
- `https://github.com/thesysdev/openui/tree/main/examples/miscellaneous/autofix`
