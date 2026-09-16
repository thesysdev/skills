# Integrate OpenUI Gateway with Chat Completions

Read [the shared Gateway integration guide](../integration.md) first. Use this runbook when an existing application calls `chat.completions.create()`, owns a `messages` array, or should retain app-owned conversation persistence and function-tool execution.

The central invariant is protocol preservation: moving model access or UI generation to OpenUI Gateway does not require moving the application to Responses or Gateway Conversations.

## Supported Shapes

Choose whether Gateway should compile and correct OpenUI Lang or forward the application's existing prompt:

| Mode | System prompt | Output owner |
| --- | --- | --- |
| Gateway generative UI | `generateSystemPrompt({ cloud: true, library })` | Gateway uses the supplied library spec and validates/repairs OpenUI Lang |
| Existing prompt / plain text | Preserve the application's system/developer messages | Gateway provides model routing and provider fallbacks without OpenUI Lang correction |

Do not infer the mode from the endpoint alone. Preserve the existing output mode unless the user asks to change it.

## Preserve Application-Owned History

Chat Completions is message-based. Keep the application's existing persistence and resend the relevant `system`, `developer` when supported by the selected model, `user`, `assistant`, and `tool` messages on every turn.

- Do not send only the latest message.
- Do not add `conversation`, `previous_response_id`, or `store: true` Responses semantics.
- Keep the host persistence layer when changing the model endpoint.
- Preserve existing compaction, truncation, tool-result, and attachment behavior after checking Gateway/model compatibility.

If the product wants server-side history injection via `conversation`, choose Responses and treat that as a separate protocol/storage migration. A framework can also manage Gateway storage independently of its Chat Completions model call; preserve an already-configured path and verify its writes and reloads. Verify the storage integration separately from the model call.

## Configure Gateway Generative UI

Use the stock OpenAI SDK and keep the key on the server:

```ts
import { generateSystemPrompt } from "@openuidev/lang-core";
import librarySpec from "./generated/library-spec.json";
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
        library: librarySpec,
        instructions: trustedApplicationInstructions,
      }),
    },
    ...conversationMessages,
  ],
  tools: functionTools,
  stream: true,
});
```

Move existing trusted system behavior into the helper's `instructions` or another currently documented trusted-instruction seam. Do not blindly append every prior system message, and never include user-authored content there. Avoid sending duplicate Gateway prompt configurations on later turns.

Generate `librarySpec` from the same library the client renders. Follow [build-component-library.md](../../build-component-library.md) for the export and generation steps.

## Relay the Stream

Preserve the response format expected by the existing client: raw Chat Completions `data:` SSE and the OpenAI SDK's `.toReadableStream()` serialization are different formats. When a framework wraps the model call, preserve its browser transport.

For client adapter configuration, follow [Agent Interface stream wiring](../../agent-interface.md#match-the-browser-stream).

## Keep Function Tools in the Application

Chat Completions accepts function tools but does not execute them. Preserve or implement the standard application loop:

1. Send the complete relevant messages plus function declarations.
2. Read `tool_calls` from the assistant message.
3. Validate the requested name and arguments against the application's allowlist.
4. Execute authorized tools on the application server.
5. Append the assistant tool-call message and one `role: "tool"` result for each call.
6. Repeat with a bounded iteration count until the model returns the final answer.

Do not attach Responses-only hosted `web_search`, `image_search`, or remote MCP declarations to this endpoint. If the application needs those inside an agent turn, migrate intentionally to Responses.

## Keep the Existing Prompt

For ordinary model traffic, pass the application's trusted system/developer messages directly. Do not call `generateSystemPrompt()` or add Gateway's OpenUI Lang configuration. This path uses Gateway's model routing and provider fallbacks; it does not enable OpenUI Lang correction.

If the application already generates its own UI prompt, retain that prompt and its existing validation behavior. Opt into Gateway generative UI only when requested, using the explicit library configuration above.

## Adapt the Server Route

Preserve the host's framework and existing route contract. The route should:

- Authenticate and rate-limit before calling Gateway.
- Bound and validate the full messages array rather than casting `req.json()` directly.
- Allow only roles and content parts the product actually supports, including complete assistant/tool-call pairs.
- Preserve the existing storage owner and load authoritative history server-side when possible instead of trusting arbitrary browser-supplied history.
- Keep model selection behind a server-maintained allowlist.
- Forward the abort signal and the chosen raw-SSE or readable-stream shape without protocol conversion.
- Return upstream failures through the host's existing error contract without logging secrets or sensitive message content.

## Verify

1. Run the shared checks in [the Gateway integration guide](../integration.md#shared-verification).
2. Confirm the request uses `/v1/embed/chat/completions` and the expected `chat.completions.create()` shape.
3. Confirm every turn includes the intended history and exactly one Gateway prompt configuration when OpenUI Lang generation is enabled.
4. Confirm the route preserves the stream format expected by the existing client.
5. Reload a persisted thread and confirm its intended storage owner restores it; separately verify any explicit framework-to-Gateway storage integration.
6. Exercise a multi-step function tool and confirm assistant tool calls plus every tool result remain in history.
7. Confirm Responses-only hosted tool declarations are absent.
8. For generative UI, run representative prompts repeatedly and inspect both stream rendering and settled parser errors.

## First-Party References

- `https://www.openui.com/docs/gateway/api/chat-completions`
- `https://www.openui.com/docs/gateway`
- `https://www.openui.com/docs/gateway/generate-openui-lang`
