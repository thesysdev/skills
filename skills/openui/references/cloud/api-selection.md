# Choose an OpenUI Cloud API

Read this reference before selecting a Cloud endpoint, history model, stream adapter, prompt mode, or integration runbook. Prefer the host application's existing protocol unless the user requests migration or needs a capability available only on another surface.

## API Matrix

| Surface | Endpoint | Use when | History owner | Runbook |
| --- | --- | --- | --- | --- |
| Responses | `POST https://api.thesys.dev/v1/embed/responses` | Building a new agent app, using Responses already, or needing hosted tools, persistent Cloud conversations, or artifacts inside the agent stream | Cloud with `conversation` plus `store: true`; otherwise the app can send full `input` or use `previous_response_id` | [responses.md](responses.md) |
| Embed Chat Completions | `POST https://api.thesys.dev/v1/embed/chat/completions` | Preserving an existing `chat.completions.create()` application, using plain text passthrough, or retaining app-owned messages and function tools | Application resends `messages` | [chat-completions.md](chat-completions.md) |
| Artifact Chat Completions | `POST https://api.thesys.dev/v1/artifact/chat/completions` | Generating or explicitly editing a standalone slide deck or report outside an agent conversation | Application stores the program and sends its current version on edits | [artifacts.md](artifacts.md) |
| Conversations | `https://api.thesys.dev/v1/conversations` | Creating, listing, updating, or deleting persistent Responses threads and items | Cloud | [conversations.md](conversations.md) |

Responses is the recommended starting point for new agent applications, not a mandatory migration target for existing Chat Completions applications. Do not use Embed Chat Completions with the Conversations API.

## Choose the State Model Independently

Responses supports three history patterns:

- Full `input` history owned by the application.
- `previous_response_id` chains stored responses without a named conversation.
- `conversation` plus `store: true` creates a persistent thread managed through the Conversations API.

Embed Chat Completions always uses application-owned `messages` history. Artifact Chat Completions receives the current artifact program explicitly for edits.

Do not introduce Cloud Conversations merely because the application calls the Cloud generation endpoint. Persistence changes require their own authorization, storage, and migration decisions.

## Match the Client Contract

| Server response | Agent Interface stream adapter | Message format |
| --- | --- | --- |
| Responses SSE | `openAIResponsesAdapter()` | `openAIConversationMessageFormat` for Cloud conversations; verify the installed format for other Responses history modes |
| Raw Chat Completions `data:` SSE | `openAIAdapter()` | `openAIMessageFormat` |
| OpenAI SDK Chat Completions `.toReadableStream()` | `openAIReadableStreamAdapter()` | `openAIMessageFormat` |

For Responses with `conversation: threadId` and `store: true`, send only the latest user turn. Do not also resend full history. For Chat Completions, retain the complete relevant `messages` array. Preserve the upstream event shape when proxying either stream.

The Conversations API is the storage plane used by `useOpenuiCloudStorage()`. Browser access requires a short-lived frontend token minted by the application server for an authenticated `user_id` and optional stable `app_id`; never expose `THESYS_API_KEY` to browser code. This storage path is for Responses conversations, not Embed Chat Completions history. Follow [conversations.md](conversations.md) when selecting it.

## Choose the Prompt Mode

Use `generateSystemPrompt()` from `@openuidev/lang-core` for current prompt compilation.

For managed Cloud generative UI:

- Pass `{ cloud: true }`.
- Responses: put the generated prompt in `instructions`.
- Embed Chat Completions: put it in a `role: "system"` message.
- With the built-in `chatLibrary`, omit `library`.
- With a custom library, pass the serialized `library` spec and optional trusted `instructions`/`promptOptions`.

For application-owned/self-hosted generative UI sent through Embed Chat Completions, omit `cloud: true` and compile the full prompt from the application's serialized `library` and `promptOptions`. The application then owns validation and correction.

For plain text passthrough, omit the managed generative UI prompt and preserve the application's trusted system/developer instructions.

Read [build-component-library.md](../build-component-library.md) before defining or migrating a custom library. Never pair a custom runtime library with the built-in model-facing prompt or a stale serialized spec.

## Tools and Artifacts

- Responses supports hosted web search, image search, remote MCP, managed artifacts, and app-owned function tools.
- Embed Chat Completions accepts app-owned function tools but does not execute them; the application runs the standard tool loop.
- Artifact Chat Completions is for standalone slide/report programs and explicit program-based edits; follow [artifacts.md](artifacts.md).
- Use Responses plus `artifactTool()` when artifacts should live inside a persistent agent conversation.

## First-Party References

- `https://www.openui.com/docs/openui-cloud/api/overview`
- `https://www.openui.com/docs/openui-cloud/api/responses`
- `https://www.openui.com/docs/openui-cloud/api/chat-completions`
- `https://www.openui.com/docs/openui-cloud/api/artifacts`
- `https://www.openui.com/docs/openui-cloud/api/conversations`
- `https://www.openui.com/docs/openui-cloud/build/component-library`
