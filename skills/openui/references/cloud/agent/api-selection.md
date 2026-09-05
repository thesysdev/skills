# Choose an Agent API

Use this reference only to choose the agent-generation protocol and its state model. Shared configuration, security, compatibility, and verification requirements live in [the Cloud integration guide](../integration.md).

## Choose the Generation Protocol

Responses and Embed Chat Completions are the two agent-generation choices:

| Protocol | Endpoint | Use when | Continue with |
| --- | --- | --- | --- |
| Responses | `POST https://api.thesys.dev/v1/embed/responses` | Building a new agent app, preserving an existing Responses integration, or needing hosted tools or artifacts inside agent turns | [responses.md](responses.md) |
| Embed Chat Completions | `POST https://api.thesys.dev/v1/embed/chat/completions` | Preserving an existing `chat.completions.create()` application, plain-text passthrough, app-owned messages, or app-run function tools | [chat-completions.md](chat-completions.md) |

Responses is the recommended starting point for new agent applications, not a mandatory migration target. Preserve an existing Chat Completions protocol unless the user requests migration or needs a Responses-only capability.

## Choose the State Model Separately

Responses supports three history patterns:

- Full `input` history owned by the application.
- A `previous_response_id` chain stored without a named conversation.
- A persistent named thread using `conversation` plus `store: true` and the Conversations API.

Read [conversations.md](conversations.md) only for the third pattern, including conversation/item CRUD, browser thread storage, frontend tokens, and `user_id`/`app_id` isolation. Conversations is a companion to Responses, not another generation protocol.

Embed Chat Completions always uses application-owned `messages` history. Do not combine it with the Conversations API.

## Match the Client Contract

| Server response | Agent Interface stream adapter | Message format |
| --- | --- | --- |
| Responses SSE | `openAIResponsesAdapter()` | `openAIConversationMessageFormat` for Cloud conversations; verify the installed format for other Responses history modes |
| Raw Chat Completions `data:` SSE | `openAIAdapter()` | `openAIMessageFormat` |
| OpenAI SDK Chat Completions `.toReadableStream()` | `openAIReadableStreamAdapter()` | `openAIMessageFormat` |

For Responses with `conversation: threadId` and `store: true`, send only the latest user turn. Do not also resend full history. For Chat Completions, retain the complete relevant `messages` array. Preserve the upstream event shape when proxying either stream.

## Choose the Prompt Mode

Use `generateSystemPrompt()` from `@openuidev/lang-core` for current prompt compilation.

For managed Cloud generative UI:

- Pass `{ cloud: true }`.
- Responses: put the generated prompt in `instructions`.
- Embed Chat Completions: put it in a `role: "system"` message.
- With the built-in `chatLibrary`, omit `library`.
- With a custom library, pass the serialized `library` spec and optional trusted `instructions`/`promptOptions`.

For application-owned/self-hosted generative UI sent through Embed Chat Completions, omit `cloud: true` and compile the full prompt from the application's serialized `library` and `promptOptions`. The application then owns validation and correction.

For plain-text passthrough, omit the managed generative UI prompt and preserve the application's trusted system/developer instructions.

Read [build-component-library.md](../../build-component-library.md) before defining or migrating a custom library. Never pair a custom runtime library with the built-in model-facing prompt or a stale serialized spec.

## Keep Standalone Artifacts Separate

Artifact Chat Completions is a specialized endpoint for standalone slide and report programs. It is not an agent-generation protocol and does not belong in the Responses-versus-Chat-Completions decision. Read [artifacts.md](../artifacts.md) for that workflow.

When an artifact should live inside an agent conversation, use Responses with `artifactTool()` instead.

## First-Party References

- `https://www.openui.com/docs/openui-cloud/api/overview`
- `https://www.openui.com/docs/openui-cloud/api/responses`
- `https://www.openui.com/docs/openui-cloud/api/chat-completions`
- `https://www.openui.com/docs/openui-cloud/api/conversations`
- `https://www.openui.com/docs/openui-cloud/build/component-library`
