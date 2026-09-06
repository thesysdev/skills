# Integrate OpenUI Cloud with Chat Completions

Read [the shared Cloud integration guide](../integration.md) first. Use this runbook when an existing application calls `chat.completions.create()`, owns a `messages` array, or should retain app-owned conversation persistence and function-tool execution.

The central invariant is protocol preservation: moving model access or managed UI generation to OpenUI Cloud does not require moving the application to Responses or Cloud Conversations.

## Supported Shapes

The Embed Chat Completions endpoint supports three distinct modes:

| Mode | System prompt | Output owner |
| --- | --- | --- |
| Managed generative UI | `generateSystemPrompt({ cloud: true })` | Cloud assembles the managed component prompt and validates/repairs OpenUI Lang |
| Plain text passthrough | Existing application system/developer messages; omit the managed prompt sentinel | Cloud routes the selected model and returns text |
| Application-owned generative UI | `generateSystemPrompt({ library, promptOptions })` without `cloud: true` | Application owns the full prompt, validation, correction, and renderer contract |

Do not infer the mode from the endpoint alone. Preserve the existing output mode unless the user asks to change it.

## Preserve Application-Owned History

Chat Completions is message-based. Keep the application's existing persistence and resend the relevant `system`, `developer` when supported by the selected model, `user`, `assistant`, and `tool` messages on every turn.

- Do not send only the latest message.
- Do not add `conversation`, `previous_response_id`, or `store: true` Responses semantics.
- Do not replace the host database or `restStorage` with `useOpenuiCloudStorage()` merely to use the Cloud generation endpoint.
- Preserve existing compaction, truncation, tool-result, and attachment behavior after checking Cloud/model compatibility.

If the product explicitly wants Cloud-managed persistent threads, that is a protocol and storage migration. Read [responses.md](responses.md) and [conversations.md](conversations.md), then treat the migration as a separate user-visible change.

## Configure Managed Generative UI

Use the stock OpenAI SDK and keep the key on the server:

```ts
import { generateSystemPrompt } from "@openuidev/lang-core";
import OpenAI from "openai";

const embedClient = new OpenAI({
  apiKey: process.env.THESYS_API_KEY,
  baseURL: "https://api.thesys.dev/v1/embed",
});

const stream = await embedClient.chat.completions.create({
  model,
  messages: [
    {
      role: "system",
      content: generateSystemPrompt({
        cloud: true,
        instructions: trustedApplicationInstructions,
      }),
    },
    ...conversationMessages,
  ],
  tools: functionTools,
  stream: true,
});
```

Move existing trusted system behavior into the helper's `instructions` or another currently documented trusted-instruction seam. Do not blindly append every prior system message, and never include user-authored content there. Avoid sending duplicate managed prompts on later turns.

For a custom component library, follow [build-component-library.md](../../build-component-library.md): generate a serialized spec, pass it as `library` with `cloud: true`, and render with the matching runtime library.

## Match the Client Stream

For raw Chat Completions `data:` SSE:

```tsx
import {
  AgentInterface,
  fetchLLM,
  openAIAdapter,
  openAIMessageFormat,
} from "@openuidev/react-ui";
import { chatLibrary } from "@openuidev/thesys";
import "@openuidev/thesys/styles.css";

const llm = fetchLLM({
  url: "/api/chat",
  streamAdapter: openAIAdapter(),
  messageFormat: openAIMessageFormat,
});

export function CloudChat() {
  return <AgentInterface llm={llm} componentLibrary={chatLibrary} />;
}
```

Use `openAIReadableStreamAdapter()` instead when the server returns the OpenAI SDK stream through `.toReadableStream()`. Do not pair a Chat Completions stream with `openAIResponsesAdapter()` or `openAIConversationMessageFormat`.

If the application already owns its chat UI, keep it and render only generated OpenUI Lang with the appropriate `Renderer`; adopting `AgentInterface` is a separate choice.

## Keep Function Tools in the Application

Embed Chat Completions accepts function tools but does not execute them. Preserve or implement the standard application loop:

1. Send the complete relevant messages plus function declarations.
2. Read `tool_calls` from the assistant message.
3. Validate the requested name and arguments against the application's allowlist.
4. Execute authorized tools on the application server.
5. Append the assistant tool-call message and one `role: "tool"` result for each call.
6. Repeat with a bounded iteration count until the model returns the final answer.

Do not attach Responses-only hosted `web_search`, `image_search`, remote MCP, or `artifactTool()` declarations to this endpoint. If the application needs those inside an agent turn, migrate intentionally to Responses. For standalone slide/report generation, use Artifact Chat Completions as a separate call and follow [artifacts.md](../artifacts.md).

## Keep Plain Text and Application-Owned UI Intact

For plain text passthrough, retain the existing trusted prompt and omit `generateSystemPrompt({ cloud: true })`. The Cloud endpoint then acts as the compatible model gateway.

For application-owned generative UI, compile the complete prompt from the runtime library spec without `cloud: true`:

```ts
const systemPrompt = generateSystemPrompt({
  library: librarySpec,
  promptOptions,
});
```

In that mode the application—not Cloud's managed component path—owns prompt assembly, output validation/correction, and the renderer contract. Do not describe it as managed Cloud UI validation.

## Adapt the Server Route

Preserve the host's framework and existing route contract. The route should:

- Authenticate and rate-limit before calling Cloud.
- Bound and validate the full messages array rather than casting `req.json()` directly.
- Allow only roles and content parts the product actually supports, including complete assistant/tool-call pairs.
- Preserve the existing storage owner and load authoritative history server-side when possible instead of trusting arbitrary browser-supplied history.
- Keep model selection behind a server-maintained allowlist.
- Forward the abort signal and the chosen raw-SSE or readable-stream shape without protocol conversion.
- Return upstream failures through the host's existing error contract without logging secrets or sensitive message content.

## Verify

1. Run the shared checks in [the Cloud integration guide](../integration.md#shared-verification).
2. Confirm the request uses `/v1/embed/chat/completions` and the expected `chat.completions.create()` shape.
3. Confirm every turn includes the intended history and exactly one managed system prompt when managed UI is enabled.
4. Confirm `openAIMessageFormat` is paired with `openAIAdapter()` or `openAIReadableStreamAdapter()` as required by the returned stream.
5. Reload a persisted thread and confirm the application's existing storage—not Cloud Conversations—restores it.
6. Exercise a multi-step function tool and confirm assistant tool calls plus every tool result remain in history.
7. Confirm Responses-only hosted tool declarations are absent.
8. For generative UI, run representative prompts repeatedly and inspect both stream rendering and settled parser errors.

## First-Party References

- `https://www.openui.com/docs/openui-cloud/api/chat-completions`
- `https://www.openui.com/docs/openui-cloud/api/overview`
- `https://www.openui.com/docs/openui-cloud/build/component-library`
- `https://www.openui.com/docs/agent/reference/adapters-and-formats`
