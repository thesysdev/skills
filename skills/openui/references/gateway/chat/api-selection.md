# Choose a Chat Generation API

Use this reference only to choose the generation protocol and state model for an interactive chat or agent application. Shared configuration, security, compatibility, and verification requirements live in [the Gateway integration guide](../integration.md).

## Choose the Generation Protocol

Responses and Chat Completions are the two conversational generation choices:

| Protocol | Endpoint | Use when | Continue with |
| --- | --- | --- | --- |
| Responses | `POST https://api.thesys.dev/v1/embed/responses` | Building a new chat or agent app, preserving an existing Responses integration, or needing hosted tools | [responses.md](responses.md) |
| Chat Completions | `POST https://api.thesys.dev/v1/embed/chat/completions` | Preserving an existing `chat.completions.create()` application, plain-text passthrough, app-owned messages, or app-run function tools | [chat-completions.md](chat-completions.md) |

Responses is the recommended starting point for new agent applications, not a mandatory migration target. Preserve an existing Chat Completions protocol unless the user requests migration or needs a Responses-only capability.

## Choose the State Model Separately

For a new Gateway chat that needs persistent threads, recommend `conversation` plus `store: true` with the Conversations API. Gateway then stores the history, and the application sends only the new turn.

Choose among the three Responses history patterns:

| Pattern | Use when |
| --- | --- |
| `conversation` plus `store: true` | Recommended for new persistent Gateway chats with named, reopenable threads |
| Full `input` history | The existing application owns history or needs explicit control over the context sent to the model |
| `previous_response_id` | Stored response chaining is enough; named threads and a conversation list are not needed |

Preserve an existing application's history model unless the user requests a storage migration. Read [conversations.md](conversations.md) for named threads or a separately configured framework storage integration needing conversation/item CRUD, frontend tokens, or `user_id`/`app_id` isolation. Conversations is a storage companion, not another generation protocol.

Chat Completions requires the application/framework to supply relevant `messages`; it does not apply Responses `conversation` or `previous_response_id` semantics. Preserve app-owned storage by default. If an existing framework intentionally uses Gateway storage separately, preserve and verify its write/reload path rather than forcing a protocol migration. See [generated-backend checks](../quickstart.md#work-from-the-generated-app).

## Preserve the Stream Contract

For Responses with `conversation: threadId` and `store: true`, send only the new turn. For Chat Completions, retain the complete relevant `messages` array. Preserve the event shape expected by the application's client.

Client adapters are covered in [Agent Interface](../../agent-interface.md#match-the-browser-stream), including frameworks that expose a different browser stream from their underlying model API.

## Choose the Prompt Mode

Use `generateSystemPrompt()` from `@openuidev/lang-core` when opting into OpenUI Lang generation through Gateway.

For OpenUI Gateway generative UI:

- Pass `{ cloud: true, library }` with the serialized spec of the client library.
- Responses: put the generated prompt in `instructions`.
- Chat Completions: put it in a `role: "system"` message.
- Add optional trusted `instructions`/`promptOptions` for that library.

For ordinary model traffic, preserve the application's existing trusted prompt without calling `generateSystemPrompt()`. Gateway provides model routing and provider fallbacks without OpenUI Lang correction.

Read [build-component-library.md](../../build-component-library.md) before defining or migrating a library. Keep the runtime library and serialized spec synchronized.

## First-Party References

- `https://www.openui.com/docs/gateway`
- `https://www.openui.com/docs/gateway/api/responses`
- `https://www.openui.com/docs/gateway/api/chat-completions`
- `https://www.openui.com/docs/gateway/api/conversations`
- `https://www.openui.com/docs/gateway/generate-openui-lang`
