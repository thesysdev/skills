# Correct output without Gateway (Autofix)

Use this only when the user has declined OpenUI Gateway for inference and keeps calling their provider directly. Autofix repairs invalid OpenUI Lang after the model has generated it. The provider, keys, and agent framework stay as they are.

Autofix needs:

- `THESYS_API_KEY` on the server, created by the user (see [SKILL.md](../SKILL.md#never-handle-api-keys));
- the library spec from `openui generate --spec`, for the same library the browser renders (see [component-library.md](component-library.md)).

For pricing, link https://www.openui.com/docs/gateway/pricing-credits.

If the user declines Autofix too, keep their provider as it is and tell them they can add error correction later through Gateway or Autofix.

Install `@openuidev/server` with the project's package manager.

## Prompt the model

Without Gateway, the app sends the full OpenUI prompt itself:

```ts
import { generateSystemPrompt } from "@openuidev/lang-core";
import { library } from "@/lib/openui";
import { promptOptions } from "@/lib/prompt-options";

const systemPrompt = generateSystemPrompt({ library: library.toSpec(), promptOptions });
```

## Wrap the stream with `@openuidev/server`

`createAutofix` comes from `@openuidev/server/openai` for the OpenAI SDK (Chat Completions) and from `@openuidev/server/vercel` for the Vercel AI SDK. Create it once:

```ts
import { createAutofix } from "@openuidev/server/openai";
import library from "@/lib/openui.spec.json";

export const autofix = createAutofix({ apiKey: process.env.THESYS_API_KEY!, library });
```

`stream()` passes the provider's events through. If the finished program is invalid, it inserts the repaired program before the stream ends.

```ts
// OpenAI SDK route; the browser reads it with openAIAdapter()
const source = await openai.chat.completions.create(
  { model: "gpt-5.5", messages: [{ role: "system", content: systemPrompt }, ...messages], stream: true },
  { signal: request.signal },
);
return autofix.completions.stream({ stream: source, messages, signal: request.signal }).toResponse();
```

```ts
// Vercel AI SDK route; the browser reads it with useChat or vercelAIAdapter()
import { streamText, toUIMessageStream } from "ai";
import { createAutofix } from "@openuidev/server/vercel";

const result = streamText({ model, system: systemPrompt, messages, abortSignal: request.signal });
return autofix.ai
  .stream({ stream: toUIMessageStream({ stream: result.stream }), signal: request.signal })
  .toResponse();
```

- Consume either `chunks` or `toResponse()`, once.
- `result` settles afterwards. It is `null` when no repair ran. Otherwise, persist `result.content`, not the joined stream text.
- A repair that fails throws with `code: "fix_failed"`.
- The Responses API is not supported by the helper.

For text that is already complete, use `fix()`:

```ts
const result = await autofix.completions.fix({ generation, messages, signal });
if (result.content !== null) {
  // "already_valid" or "fixed": render or store result.content
} else {
  // "fix_failed": show the app's fallback; result.unfixedErrors lists the problems
}
```

## Call the API directly

For other stacks, parse first and call Autofix only when the finished program is invalid:

```ts
import { createParser, type ParseResult } from "@openuidev/lang-core";

function isValid({ root, meta }: ParseResult) {
  return !meta.incomplete && meta.errors.length === 0 &&
    meta.unresolved.length === 0 && meta.orphaned.length === 0 && root !== null;
}

const parsed = createParser(spec.schema, spec.root).parse(generation);
```

Then `POST https://api.thesys.dev/v1/autofix` (OpenAI-compatible; an OpenAI client with base URL `https://api.thesys.dev/v1/autofix` works):

- `messages[0]`: `{ role: "system", content: generateSystemPrompt({ cloud: true, library: spec }) }`. This is how the API receives the library.
- the conversation before the generation;
- last: `{ role: "assistant", content: generation }`.

The response is a chat completion. `choices[0].message.content` is the whole corrected program, to replace the original. `fix_summary.status` is `already_valid`, `fixed`, or `fix_failed`, with `fixed_errors` and `unfixed_errors`. Streaming is not supported.

Limits: the generation can be up to 100,000 characters; the context before it, up to 20 turns and 8,000 characters of text. Trim older turns before sending.

## Verify

1. A valid response renders unchanged.
2. A deliberately broken response (an unknown component, a missing reference) comes back fixed.
3. A failed fix shows the app's fallback instead of a broken UI.
4. `THESYS_API_KEY` appears in no client file.

## Sources

- [Autofix API](https://www.openui.com/docs/autofix)
- [`@openuidev/server`](https://www.openui.com/docs/api-reference/server)
- [Reliability](https://www.openui.com/docs/openui-lang/reliability)
- [Example](https://github.com/thesysdev/openui/tree/main/examples/miscellaneous/autofix)
