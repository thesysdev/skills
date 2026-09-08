# Choose a Chat Generation API

Use this reference only to choose the generation protocol and state model for an interactive chat or agent application. Shared configuration, security, compatibility, and verification requirements live in [the Gateway integration guide](../integration.md).

## Choose the Generation Protocol

Responses and Embed Chat Completions are the two conversational generation choices:

| Protocol | Endpoint | Use when | Continue with |
| --- | --- | --- | --- |
| Responses | `POST https://api.thesys.dev/v1/embed/responses` | Building a new chat or agent app, preserving an existing Responses integration, or needing hosted tools or artifacts inside turns | [responses.md](responses.md) |
| Embed Chat Completions | `POST https://api.thesys.dev/v1/embed/chat/completions` | Preserving an existing `chat.completions.create()` application, plain-text passthrough, app-owned messages, or app-run function tools | [chat-completions.md](chat-completions.md) |

Responses is the recommended starting point for new agent applications, not a mandatory migration target. Preserve an existing Chat Completions protocol unless the user requests migration or needs a Responses-only capability.

## Choose the State Model Separately

Responses supports three history patterns:

- Full `input` history owned by the application.
- A `previous_response_id` chain stored without a named conversation.
- A persistent named thread using `conversation` plus `store: true` and the Conversations API.

Read [conversations.md](conversations.md) for the third pattern or a separately configured framework storage integration needing conversation/item CRUD, frontend tokens, or `user_id`/`app_id` isolation. Conversations is a storage companion, not another generation protocol.

Embed Chat Completions requires the application/framework to supply relevant `messages`; it does not apply Responses `conversation` or `previous_response_id` semantics. Preserve app-owned storage by default. If an existing framework intentionally uses Gateway storage separately, preserve and verify its write/reload path rather than forcing a protocol migration. See [framework scaffold contracts](../quickstart.md#work-from-the-generated-app).

## Match the Client Contract

| Server response | Agent Interface stream adapter | Message format |
| --- | --- | --- |
| Responses SSE | `openAIResponsesAdapter()` | `openAIConversationMessageFormat` for Gateway conversations; verify the installed format for other Responses history modes |
| Raw Chat Completions `data:` SSE | `openAIAdapter()` | `openAIMessageFormat` |
| OpenAI SDK Chat Completions `.toReadableStream()` | `openAIReadableStreamAdapter()` | `openAIMessageFormat` |

For Responses with `conversation: threadId` and `store: true`, send only the latest user turn. Do not also resend full history. For Chat Completions, retain the complete relevant `messages` array. Preserve the upstream event shape when proxying either stream.

This table covers direct provider-stream proxies. A framework may instead return a UIMessage, LangGraph, AG-UI, or Eve stream; select the browser adapter from that actual stream, not from the model API hidden behind the framework.

## Choose the Prompt Mode

Use `generateSystemPrompt()` from `@openuidev/lang-core` for current prompt compilation.

For managed Gateway generative UI:

- Pass `{ cloud: true }`.
- Responses: put the generated prompt in `instructions`.
- Embed Chat Completions: put it in a `role: "system"` message.
- With the built-in `chatLibrary`, omit `library`.
- With a custom library, pass the serialized `library` spec and optional trusted `instructions`/`promptOptions`.

For application-owned/self-hosted generative UI sent through Embed Chat Completions, omit `cloud: true` and compile the full prompt from the application's serialized `library` and `promptOptions`. The application then owns validation and correction.

For plain-text passthrough, omit the managed generative UI prompt and preserve the application's trusted system/developer instructions.

Read [build-component-library.md](../../build-component-library.md) before defining or migrating a custom library. Never pair a custom runtime library with the built-in model-facing prompt or a stale serialized spec.

## Keep Standalone Artifacts Separate

Artifact Chat Completions is a specialized, version-sensitive contract for standalone slide and report programs. Its former docs were removed from the live site; read [artifacts.md](../artifacts.md) and verify current availability before new integration work. It is not a peer conversational generation protocol.

When an artifact should live inside an agent conversation, use Responses with `artifactTool()` instead.

## First-Party References

- `https://www.openui.com/docs/gateway`
- `https://www.openui.com/docs/gateway/api/responses`
- `https://www.openui.com/docs/gateway/api/chat-completions`
- `https://www.openui.com/docs/gateway/api/conversations`
- `https://www.openui.com/docs/gateway/generate-openui-lang`
