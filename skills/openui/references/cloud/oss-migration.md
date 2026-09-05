# OSS to Cloud Migration

Use this runbook for application-code migration from OSS (open-source or self-hosted) OpenUI to the managed Cloud backend. Read [the shared Cloud integration guide](integration.md) and [Choose an agent API](agent/api-selection.md), then read the selected [Responses](agent/responses.md) or [Chat Completions](agent/chat-completions.md) runbook. Also read [Conversations](agent/conversations.md) when Cloud will own persistent threads.

## Contents

1. [Define the Migration](#define-the-migration)
2. [Classify the Existing App](#classify-the-existing-app)
3. [Preserve Chat Completions](#preserve-chat-completions)
4. [Migrate to Responses and Conversations](#migrate-to-responses-and-conversations)
5. [Handle Renderer-Only and Custom-Library Apps](#handle-renderer-only-and-custom-library-apps)
6. [Handle Dual Mode](#handle-dual-mode)
7. [Respect Unsupported Boundaries](#respect-unsupported-boundaries)
8. [Verify](#verify)

## Define the Migration

Distinguish these goals before removing code:

- **Replace:** Cloud becomes the only generation and storage backend.
- **Dual mode:** Keep the OSS path beside Cloud when a required capability lacks verified Cloud parity, or when the rollout needs a reversible comparison before replacement.
- **Code migration:** Rewire the application to Cloud and start with Cloud-owned conversations.
- **Data migration:** Import historical threads or artifacts into Cloud.

If the request says only “migrate to Cloud,” default to code migration and preserve historical data in place. Do not claim data migration unless a current first-party import API is documented and verified.

If the app uses OSS components, ask whether the user wants to keep them or switch to Cloud's `chatLibrary`. Treat backend, storage, and component-library migration as separate choices; moving to Cloud does not automatically authorize replacing the visible component set.

## Classify the Existing App

Inventory the project before editing:

| Shape                        | Typical signals                                                              | Migration character                                                           |
| ---------------------------- | ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| Self-hosted `AgentInterface` | `fetchLLM`, `ChatLLM`, `restStorage`, `openuiLibrary` or `openuiChatLibrary` | Mostly mechanical adapter/storage/library swap                                |
| Legacy flat-prop chat        | `apiUrl`, `threadApiUrl`, `processMessage`, old renderer names               | First migrate to current `AgentInterface`, then migrate backend               |
| Renderer-only GenUI          | `Renderer`, `library.prompt()`, app-owned messages and model stream          | Keep the renderer or adopt Agent Interface; choose Cloud state and generation APIs explicitly |
| Custom component library     | `defineComponent`, `createLibrary`, custom prompt options                    | Reuse the runtime library and generate the matching serialized spec for Cloud |
| Tool-heavy agent             | provider tool loop or AG-UI tool events                                      | Preserve the current protocol for app-owned tools; choose Responses only when hosted Cloud tools are required |

Record the current provider, model selector, message wire format, attachments,
storage ownership, user identity, route behavior, tools, component library,
artifacts, custom slots/theme, and tests.

## Preserve Chat Completions

If the existing application uses `chat.completions.create()` and the user did not request a protocol migration, keep its message and storage architecture:

1. Read [Chat Completions](agent/chat-completions.md).
2. Point the existing OpenAI-compatible client at the Cloud Embed base URL and replace provider-specific model ids with an allowed current `{provider}/{model}` value.
3. For managed generative UI, put `generateSystemPrompt({ cloud: true, ... })` from `@openuidev/lang-core` in the trusted system-message position. Preserve trusted application behavior through the helper's documented instructions option.
4. Keep the complete relevant `messages` history, including assistant tool calls and tool results.
5. Keep the host persistence layer. Do not add Cloud Conversations, `useOpenuiCloudStorage()`, or a frontend-token route.
6. Keep app-owned function tool execution and its bounded loop.
7. Pair raw Chat Completions SSE with `openAIAdapter()` or an SDK readable stream with `openAIReadableStreamAdapter()`, using `openAIMessageFormat`.

This is still a Cloud migration: model routing and, when selected, managed UI validation/correction move to Cloud while the application retains its established Chat Completions protocol and state owner.

## Migrate to Responses and Conversations

Apply this map only when the user chose Responses and Cloud Conversations. Do not treat it as the default transformation for an existing Chat Completions application.

| Self-hosted/open-source                              | OpenUI Cloud                                                                                           |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Provider API key and provider route                  | Server-only `THESYS_API_KEY` and Cloud Responses proxy                                                 |
| Full message history sent per turn                   | Latest message only plus `conversation: threadId`                                                      |
| `openAIReadableStreamAdapter()` or `openAIAdapter()` | `openAIResponsesAdapter()`                                                                             |
| `openAIMessageFormat`                                | `openAIConversationMessageFormat`                                                                      |
| In-memory, `restStorage`, or custom `ChatStorage`    | `useOpenuiCloudStorage({ token: "/api/frontend-token" })`                                              |
| `openuiLibrary`/`openuiChatLibrary`                  | `chatLibrary` from `@openuidev/thesys` when the user chooses Cloud components                          |
| `library.prompt(...)` in the provider route          | `generateSystemPrompt({ cloud: true, ... })` from `@openuidev/lang-core`; pass a generated spec when preserving a custom library |
| App-owned artifact loop/renderers                    | Managed `artifactTool({ artifacts: ["slides", "report"] })` plus managed renderers for stock artifacts |
| No browser storage credential                        | Short-lived frontend token scoped to the authenticated `user_id`                                       |

Preserve branding, theme, starters, slots, navigation, route placement, error boundaries, analytics, and authentication unless the user requests a redesign.

For the generation contract, read [Responses](agent/responses.md). For persistent named threads, frontend tokens, and identity, also read [Conversations](agent/conversations.md).

### Migrate AgentInterface apps to Responses

1. Read [the Cloud integration guide](integration.md), [Responses](agent/responses.md), and [Conversations](agent/conversations.md), then add only the missing Cloud packages and server-only configuration.
2. Replace the client `llm` with `fetchLLM({ streamAdapter: openAIResponsesAdapter(), messageFormat: openAIConversationMessageFormat })` or an equivalent direct `ChatLLM`. The server must forward only the latest allowlisted user message to the stored Cloud conversation.
3. In Next.js, move the Cloud UI into a separate client component and match the
   installed first-party template's dynamic-rendering boundary. Add an
   `ssr: false` client loader only when the production build requires it.
4. Replace self-hosted storage with `useOpenuiCloudStorage()` and add the frontend-token route.
5. If the user chose Cloud components, replace the stock OSS library with `chatLibrary` and register the managed report/presentation renderers. If they keep a custom library, generate its serialized spec, pass it to `generateSystemPrompt({ cloud: true, library, ... })`, and render with the matching runtime library.
6. Replace the provider `/api/chat` implementation with the Cloud proxy while preserving independent API authentication, conversation authorization, rate limiting, request validation, abort propagation, error handling, and the route URL expected by the client.
7. Keep the previous provider/storage code until the Cloud path builds and passes tests. Remove it only for an explicitly confirmed replacement migration.
8. Remove provider dependencies, environment variables, storage routes, and dead adapters only when no other application path uses them.
9. Update environment examples and deployment configuration without committing secrets.

Do not send both full history and a Cloud conversation id. Doing so can duplicate context. Do not leave `library.prompt()` in the managed Cloud route; `generateSystemPrompt({ cloud: true, ... })` from `@openuidev/lang-core` supplies the managed instructions for the built-in library or the provided serialized custom-library spec.

## Handle Renderer-Only and Custom-Library Apps

A Renderer-only app owns its messages, model stream, and layout. It can keep its client renderer while moving generation to the appropriate Cloud API, or adopt `AgentInterface` plus Cloud storage. Treat renderer, generation, and persistence as separate migration choices.

For an Agent Interface migration:

1. Introduce `AgentInterface` at the requested route or surface.
2. Move reusable branding and surrounding layout into `AgentInterface` props/slots.
3. Add the generation route from [Responses](agent/responses.md) and the frontend-token and storage wiring from [Conversations](agent/conversations.md).
4. Retain the old Renderer surface until behavior parity is verified; then remove it only for replacement migrations.

For a renderer-preserving or custom component-library migration:

- **Can the client render it?** Keep the existing `Renderer.library` or pass the library through `AgentInterface.componentLibrary`.
- **Will Cloud generation receive matching component instructions?** Generate a serialized spec with `openui generate --spec` and pass it to `generateSystemPrompt({ cloud: true, library, ... })` from `@openuidev/lang-core`.
- **Who owns history?** Use Responses with `conversation` plus `store: true` for Cloud persistence, or choose Embed Chat Completions when the application should continue resending its own history.

Regenerate the spec whenever the runtime library contract changes. Do not pair the built-in Cloud prompt with a custom client library; the model and renderer must use the same component contract. Read [build-component-library.md](../build-component-library.md) for the complete workflow.

## Handle Dual Mode

Keep each backend internally consistent. Define two complete configurations rather than mixing adapters and storage:

```text
selfHosted = full-history transport + provider route + self-hosted storage + OSS library prompt
cloudChatCompletions = full messages + Cloud Embed proxy + app storage/tools + Chat Completions adapters
cloudResponses = selected Responses history model + Cloud proxy + matching storage + Responses adapter
```

Select the mode on the server or through trusted deployment configuration. Do not expose secrets or allow an untrusted browser value to choose arbitrary upstream credentials. Namespace storage and routes when necessary to avoid thread-id collisions.

## Respect Unsupported Boundaries

- **Historical conversations/artifacts:** no import path is established by the repository sources. Preserve the old store read-only or export it separately; do not fabricate Cloud records.
- **Custom tool execution:** supported. For Chat Completions, preserve the application's standard assistant-tool/result loop. For Responses, declare `type: "function"` tools and use the current template's bounded loop from [Responses](agent/responses.md#app-owned-function-tools). Never execute or answer Cloud-owned `thesys_*` function calls in the Responses loop.
- **Custom artifact-producing tools:** managed `artifactTool()` covers the documented report and slide path inside Responses. Use [artifacts.md](artifacts.md) for standalone slide/report programs; do not infer support for arbitrary custom artifact types.
- **Attachments and media:** preserve an attachment-capable self-hosted path until the installed Cloud client, generation input, storage, and size-limit contracts are verified end to end.
- **Non-React clients:** the verified managed client surface is React. Require a first-party runtime/example before promising another framework.

## Verify

1. Run formatter, typecheck, tests, and production build.
2. Search for stale adapter/format pairs, browser-exposed keys, and obsolete provider routes. For Responses conversations reject duplicate full-history sends; for Chat Completions reject accidental latest-message-only sends.
3. Exercise both modes independently when dual mode is retained.
4. Verify existing self-hosted data remains accessible or deliberately archived; do not describe it as imported without evidence.
5. With an authorized test key, verify streaming, the selected persistence owner, and user isolation. Verify a managed artifact only when the selected surface supports that lifecycle.
6. Compare user-visible behavior—theme, starters, navigation, tools, and custom components—and explicitly list any capability intentionally left self-hosted.
