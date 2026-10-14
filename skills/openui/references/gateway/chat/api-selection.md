# Choose a chat generation API

Use this reference only to choose the generation protocol and state model for an interactive chat or agent application. Shared configuration, security, compatibility, and verification requirements live in [the Gateway integration guide](../integration.md).

## Choose the generation protocol

Responses and Chat Completions are the two conversational generation choices:

| Protocol | Endpoint | Use when | Continue with |
| --- | --- | --- | --- |
| Responses | `POST https://api.thesys.dev/v1/embed/responses` | Building a new chat or agent app, preserving an existing Responses integration, or needing hosted tools | [responses.md](responses.md) |
| Chat Completions | `POST https://api.thesys.dev/v1/embed/chat/completions` | Preserving an existing `chat.completions.create()` application, plain-text passthrough, app-owned messages, or app-run function tools | [chat-completions.md](chat-completions.md) |

Responses is the recommended starting point for new agent applications, not a mandatory migration target. Preserve an existing Chat Completions protocol unless the user requests migration or needs a Responses-only capability.

## Choose the state model separately

For a new Gateway chat that needs persistent threads, recommend `conversation` plus `store: true` with the Conversations API. Gateway then stores the history, and the application sends only the new turn. Responses also supports full `input` history and `previous_response_id` chaining; [Choose the history model](responses.md#choose-the-history-model) compares all three.

Preserve an existing application's history model unless the user requests a storage migration. Read [conversations.md](conversations.md) for named threads or a separately configured framework storage integration needing conversation/item CRUD, frontend tokens, or `user_id`/`app_id` isolation. Conversations is a storage companion, not another generation protocol.

Chat Completions requires the application/framework to supply relevant `messages`; it does not apply Responses `conversation` or `previous_response_id` semantics. Preserve app-owned storage by default. If an existing framework intentionally uses Gateway storage separately, preserve and verify its write/reload path rather than forcing a protocol migration. See [generated-backend checks](../quickstart.md#work-from-the-generated-app).

## Preserve the stream contract

For Responses with `conversation: threadId` and `store: true`, send only the new turn. For Chat Completions, retain the complete relevant `messages` array. Preserve the event shape expected by the application's client.

Client adapters are covered in [Agent Interface](../../agent-interface.md#match-the-browser-stream), including frameworks that expose a different browser stream from their underlying model API.

## Choose the prompt mode

For OpenUI Lang generation through Gateway, build the prompt with `generateSystemPrompt({ cloud: true, library })` as described in [Connect the backend](../../build-component-library.md#connect-the-backend). For ordinary model traffic, keep the application's existing trusted prompt; Gateway then provides model routing and provider fallbacks without OpenUI Lang correction.

## First-party references

- [Gateway overview](https://www.openui.com/docs/gateway)
- [Responses API](https://www.openui.com/docs/gateway/api/responses)
- [Chat Completions API](https://www.openui.com/docs/gateway/api/chat-completions)
- [Conversations API](https://www.openui.com/docs/gateway/api/conversations)
- [Generate OpenUI Lang](https://www.openui.com/docs/gateway/generate-openui-lang)
