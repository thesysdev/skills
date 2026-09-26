# Gateway Chat Completions API

Use this guide when an app calls `chat.completions.create()`, owns its `messages` array, or runs its own function tools. Read [overview.md](overview.md) first for endpoints, the prompt, and route security. Moving model access to Gateway does not require moving the app to Responses.

```text
POST https://api.thesys.dev/v1/embed/chat/completions
```

## Choose the Output Mode

| Mode | System message | Result |
| --- | --- | --- |
| Generated UI | `generateSystemPrompt({ cloud: true, library })` | Gateway validates and repairs OpenUI Lang against the library |
| Plain model traffic | The app's existing system or developer messages | Model routing and provider fallbacks, without OpenUI Lang correction |

Keep the app's current mode unless the user asks to change it. An app that builds its own UI prompt keeps that prompt and its own validation until the user opts into generated UI through Gateway.

## Send the Request

```ts
import { generateSystemPrompt } from "@openuidev/lang-core";
import OpenAI from "openai";
import librarySpec from "./generated/library-spec.json";

const gateway = new OpenAI({
  apiKey: process.env.THESYS_API_KEY,
  baseURL: "https://api.thesys.dev/v1/embed",
});

const stream = await gateway.chat.completions.create(
  {
    model,
    messages: [
      {
        role: "system",
        content: generateSystemPrompt({
          cloud: true,
          library: librarySpec,
          instructions: trustedApplicationInstructions,
        }),
      },
      ...conversationMessages,
    ],
    tools: functionTools,
    stream: true,
  },
  { signal: req.signal },
);
```

Move the app's trusted system behavior into the helper's `instructions`, and send exactly one Gateway system message per request; strip earlier copies from stored history.

## Keep History in the App

Chat Completions sends history with every request. Resend the relevant `system`, `developer` (when the model supports it), `user`, `assistant`, and `tool` messages, and keep the app's existing persistence, compaction, and truncation.

- Never send only the latest message.
- Never add `conversation`, `previous_response_id`, or `store` from the Responses API.
- Load authoritative history on the server when possible instead of trusting history from the browser.

To keep threads in Gateway Conversations while staying on Chat Completions, write each completed turn with `storeChatCompletionHistory` ([conversations.md](conversations.md#store-chat-completions-turns)). For history injected by Gateway, migrate to [Responses](responses.md) as a separate change.

## Relay the Stream

Return the format the client expects. Raw Chat Completions SSE (`.asResponse()`) pairs with `openAIAdapter()`, and the SDK's `.toReadableStream()` output pairs with `openAIReadableStreamAdapter()`; the two are different formats. When a framework wraps the model call, keep its browser stream. See [Agent Interface stream wiring](../agent-interface.md#match-the-browser-stream).

## Run Function Tools

Gateway accepts function declarations but never runs them. Use the bounded loop in [self-hosted.md](../self-hosted.md#run-function-tools); it is the same with Gateway. Hosted web search, image search, and remote MCP are available only through Responses; do not declare them here.

## Route Rules

On top of the [shared route rules](overview.md#protect-the-server-boundary):

- Bound and validate the whole `messages` array instead of casting `req.json()`.
- Accept only the roles and content parts the product supports, with complete assistant tool-call and tool-result pairs.

## Verify

1. Run the [shared checks](overview.md#verify).
2. Requests go to `/v1/embed/chat/completions`, and each turn carries the intended history plus exactly one Gateway system message when generated UI is on.
3. The client receives the stream format its adapter expects.
4. A reloaded thread comes back from its storage owner, including tool calls and results.
5. A multi-step function tool keeps every call and result in history.
6. No hosted tool declarations are sent.

## First-Party References

- `https://www.openui.com/docs/gateway/api/chat-completions`
- `https://www.openui.com/docs/gateway/generate-openui-lang`
