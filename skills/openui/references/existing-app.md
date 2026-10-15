# Add generative UI to an existing app

Use this when the user has an app and wants its model responses to render as UI. Three pieces change: a component library, the backend's model call, and the chat UI's assistant message. Everything else stays.

## 1. Inventory the app

Before editing, find:

- the framework, router, and package manager;
- the chat UI, if any: assistant-ui, CopilotKit, a hand-built message list, or none;
- the backend route that calls the model, and the client it uses (OpenAI SDK, Anthropic SDK, LangChain, Vercel AI SDK, or none);
- how messages are stored, and how users are authenticated;
- whether the app already generates UI (its own prompt and renderer, or a self-hosted OpenUI setup).

Then settle what the user wants:
- **Replace or run beside.** Decide whether Gateway replaces the current model path or runs beside it. Ask only when the request leaves this open and it matters. Never silently delete a working backend.
- **UI generation.** An app that already generates its own UI keeps its prompt and validation unless the user asks to switch to OpenUI.
- **Stored history.** Moving code to Gateway does not move stored history. Treat any data migration as a separate task; see [gateway.md](gateway.md#conversations-api).

## 2. Define the component library

Create `src/lib/openui.tsx` for the browser and `src/lib/prompt-options.ts` for the server. Start with the built-in library:

```tsx
// src/lib/openui.tsx
export { openuiLibrary as library } from "@openuidev/react-ui";
```

```ts
// src/lib/prompt-options.ts — safe to import on the server
export { openuiPromptOptions as promptOptions } from "@openuidev/react-ui/genui-lib/prompt-options";
```

If the app already has OpenUI files at other paths, keep those paths.

Use a custom library when the app has its own design system or domain components; see [component-library.md](component-library.md).

Install the packages with the project's package manager: `@openuidev/react-lang`, `@openuidev/react-ui`, and `@openuidev/lang-core`. Leave out `@openuidev/react-ui` if the app uses only its own components.

Serialize the library for the server:

```bash
npx @openuidev/cli@latest generate src/lib/openui.tsx --spec --out src/lib/openui.spec.json
```

Regenerate the spec whenever the library changes, for example from a `prebuild` script.

## 3. Point the backend at Gateway

- **The app calls a model with the OpenAI client:** keep it and change three things: the key, the base URL, and the model id.
- **The app uses another client** (Anthropic SDK, LangChain, Vercel AI SDK with another provider): switch the model call to the OpenAI client, or to the framework's OpenAI-compatible model with a custom base URL:
  - LangChain: `new ChatOpenAI({ model, apiKey, configuration: { baseURL } })`.
  - Vercel AI SDK: `createOpenAI({ apiKey, baseURL }).chat("openai/gpt-5")`. Use `.chat(...)`: calling the provider directly uses the Responses API.
- **The app has no backend route:** write one with the OpenAI client in the app's framework.

```ts
import OpenAI from "openai";
import { generateSystemPrompt, type LibrarySpec } from "@openuidev/lang-core";
import library from "@/lib/openui.spec.json";
import { promptOptions } from "@/lib/prompt-options";

const gateway = new OpenAI({
  apiKey: process.env.THESYS_API_KEY,
  baseURL: "https://api.thesys.dev/v1/embed",
});

const completion = await gateway.chat.completions.create({
  model: "openai/gpt-5",
  messages: [
    {
      role: "system",
      content: generateSystemPrompt({
        cloud: true,
        library: library as LibrarySpec,
        promptOptions,
        instructions: appPrompt,
      }),
    },
    ...messages,
  ],
  stream: true,
});
```

- Model ids are `{provider}/{model}`: `gpt-5` becomes `openai/gpt-5`. An env var holding a bare id needs the prefix too.
- If the app already has a system prompt, pass it as `instructions` (`appPrompt` above). Remove lines that ask for Markdown or plain-prose answers; they conflict with OpenUI Lang.
- Keep the existing message history, streaming, and tool loop. If the route did not stream, make it stream, so the UI builds up as it arrives.
- If the route wraps the stream with Autofix (`createAutofix`), remove the wrapper: Gateway corrects the output while it streams. `toResponse()` returned Chat Completions SSE; return the same format yourself (see [gateway.md](gateway.md#stream-to-the-browser)), so the browser's adapter keeps working.
- If the app is a self-hosted OpenUI app, replace its full prompt (`library.prompt()`, or `generateSystemPrompt` without `cloud`) with the `cloud: true` prompt, and remove any home-grown parse-and-retry correction loop. Send exactly one OpenUI prompt per request.
- Once the Gateway path passes its checks, remove `OPENAI_API_KEY` and other provider keys the app no longer uses from `.env.example`.

The user adds `THESYS_API_KEY` to the app's environment; see "Never handle API keys" in [SKILL.md](../SKILL.md#never-handle-api-keys). [gateway.md](gateway.md) covers models, BYOK, history, tools, and security.

If the user explicitly declines Gateway for inference, follow [autofix.md](autofix.md) instead of this step.

## 4. Render in the chat UI

The chat UI keeps its message list, composer, threads, and transport. Only the assistant message changes.

| The app has | Change |
| --- | --- |
| assistant-ui | The assistant message's text part: pass an OpenUI component as `Text` to `MessagePrimitive.Parts` |
| CopilotKit | `CopilotChat`'s `messageView.assistantMessage.markdownRenderer` |
| A hand-built chat | The assistant message component |
| No chat UI | Add `AgentInterface`; see [agent-interface.md](agent-interface.md) |

Whatever the chat UI, the renderer needs:

- **The same library as the server.** Import `library` from `src/lib/openui.tsx`.
- **The styles,** when using the built-in library. Import `@openuidev/react-ui/styles/index.css` once, from the global stylesheet. With Tailwind v4, import `@openuidev/react-ui/layered/styles/index.css` instead.
- **`isStreaming`,** true until the response stream closes.
- **`onAction`.** Without it, buttons and forms in the generated UI do nothing.

assistant-ui:

```tsx
"use client";

import { MessagePrimitive, useAui, type TextMessagePartProps } from "@assistant-ui/react";
import { BuiltinActionType, Renderer, type ActionEvent } from "@openuidev/react-lang";
import { library } from "@/lib/openui";

function toModelMessage(event: ActionEvent) {
  const lines = [`User clicked: ${event.humanFriendlyMessage}`];
  if (event.formState) lines.push(`Form values: ${JSON.stringify(event.formState)}`);
  return lines.join("\n");
}

function OpenUIText({ text, status }: TextMessagePartProps) {
  const aui = useAui();
  const onAction = (event: ActionEvent) => {
    if (event.type === BuiltinActionType.ContinueConversation) {
      aui.thread.append({ role: "user", content: [{ type: "text", text: toModelMessage(event) }] });
    } else if (event.type === BuiltinActionType.OpenUrl) {
      window.open(String(event.params?.url), "_blank", "noopener,noreferrer");
    }
  };
  return (
    <Renderer
      response={text}
      library={library}
      isStreaming={status.type === "running"}
      onAction={onAction}
    />
  );
}

export function AssistantMessage() {
  return (
    <MessagePrimitive.Root>
      <MessagePrimitive.Parts components={{ Text: OpenUIText }} />
    </MessagePrimitive.Root>
  );
}
```

CopilotKit: render the same way in a `markdownRenderer` component that receives `{ content }`. In `onAction`, call `agent.addMessage({ id: crypto.randomUUID(), role: "user", content: toModelMessage(event) })` from `useAgent()`, then `copilotkit.runAgent({ agent })` from `useCopilotKit()`. Both hooks come from `@copilotkit/react-core/v2`.

A hand-built chat: render `<Renderer response={content} library={library} isStreaming={isStreaming} onAction={onAction} />` in the assistant message. In `onAction`, call the chat's existing send function. Show `event.humanFriendlyMessage` in the message list, and send `toModelMessage(event)` to the backend.

### Vue, Svelte, and Angular

The steps above use React. For other frameworks:

- **The built-in libraries are React-only.** Define a small custom library with `@openuidev/vue-lang`, `@openuidev/svelte-lang`, or `@openuidev/angular-lang` (see [component-library.md](component-library.md)).
- **Matching examples:** start from `examples/app-frameworks/vue`, `svelte`, or `angular` in the OpenUI repo. They show the renderer, action handling, and how the server builds the prompt from the same library.
- **The spec:** `openui generate` loads React library files. For other frameworks, follow the example: keep the component schemas in a shared file, build the library on the server with `createLibrary` from `@openuidev/lang-core`, and pass `library.toSpec()` to `generateSystemPrompt`.
- **No built-in stylesheet:** style the components with the app's own CSS.
- **Version match:** use the `@openuidev/lang-core` version that the renderer package depends on (check its `package.json`). `@openuidev/angular-lang` 0.4 depends on `lang-core` ^0.4, not 1.0. A mismatched `lang-core` on the server produces a prompt the renderer cannot parse.
- **Actions:** the renderer emits the same action events as React. Handle `continue_conversation` and `open_url` the same way.

In Next.js App Router, put `"use client"` at the top of any file that renders `Renderer` or `AgentInterface`. In Vite or strict TypeScript, CSS imports need `/// <reference types="vite/client" />` or `declare module "*.css";`.

## 5. Keep history correct

- Send earlier assistant turns back as the OpenUI Lang they were generated as, not as rendered text or a summary.
- A click in the generated UI arrives as a new user turn. Send the form values with it, as `toModelMessage` does.
- To keep form values after a reload, store them with the message and pass them back: `<Renderer initialState={message.formState} onStateUpdate={(state) => saveFormState(message.id, state)} />`.

## 6. Verify

1. Run the project's typecheck and build.
2. Send a message and confirm the UI builds up while it streams.
3. Click a button and submit a form in the generated UI; confirm each sends a new turn with the form values.
4. Reload a thread and confirm earlier responses still render.
5. Confirm `THESYS_API_KEY` appears in no client file.

## Sources

- [Choose a chat UI](https://www.openui.com/docs/build-agents/chat-ui)
- [assistant-ui](https://www.openui.com/docs/build-agents/assistant-ui), [CopilotKit](https://www.openui.com/docs/build-agents/copilotkit), [Custom chat UI](https://www.openui.com/docs/build-agents/custom-chat-ui)
- [Set up UI generation](https://www.openui.com/docs/build-agents/ui-generation)
- [Conversation history](https://www.openui.com/docs/build-agents/conversation-history)
- [Migrate to OpenUI Gateway](https://www.openui.com/docs/gateway/migrate-to-openui-gateway)
