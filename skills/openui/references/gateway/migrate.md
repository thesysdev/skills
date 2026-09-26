# Move an Existing App to Gateway

Read [overview.md](overview.md) first. Pick the lowest [level](../../SKILL.md#choose-a-level) that delivers the request:

| Goal | Change |
| --- | --- |
| Fix invalid generated UI, keeping the provider | Add [Autofix](autofix.md); no generation change |
| Model routing, fallbacks, BYOK, or in-stream correction | Point the existing protocol at Gateway (below) |
| Hosted tools or Gateway-managed threads | Also adopt [Responses](responses.md) and [Conversations](conversations.md) |

Code migration and data migration are separate tasks. Decide whether Gateway replaces the old backend or runs beside it during rollout; ask only when the request leaves this open and the choice matters. Keep a working backend in place until the new path is verified.

## OpenAI-Compatible App

For an app that already calls `chat.completions.create()` or `responses.create()`:

1. Store a Gateway key as `THESYS_API_KEY` on the server ([authentication handoff](quickstart.md#complete-authentication-with-the-user)).
2. Set the client's base URL to `https://api.thesys.dev/v1/embed`.
3. Change model ids to `{provider}/{model}`, such as `openai/gpt-5`, and keep the app's allowlist.

```ts
import OpenAI from "openai";

export const client = new OpenAI({
  apiKey: process.env.THESYS_API_KEY,
  baseURL: "https://api.thesys.dev/v1/embed",
});
```

Messages, streaming handlers, function-tool loops, and persistence stay as they are. This alone routes model traffic through Gateway. To generate UI as well, add the `cloud: true` prompt from [overview.md](overview.md#configure-the-prompt) and render the matching library on the client. Use [BYOK](overview.md#configure-byok) to keep billing on the app's own provider account.

## Self-Hosted OpenUI App

A self-hosted OpenUI app (for example, one created from `--template openui-self-hosted`) already has a library spec, a route, and a client. Move it step by step:

| Piece | Self-hosted | Gateway |
| --- | --- | --- |
| Server key | `OPENAI_API_KEY` or another provider key | `THESYS_API_KEY` |
| Client base URL | Provider default | `https://api.thesys.dev/v1/embed` |
| Model id | Provider id, such as `gpt-5.2` | `{provider}/{model}` from a server allowlist |
| Prompt | `generateSystemPrompt({ library, promptOptions })` | `generateSystemPrompt({ cloud: true, library })`; only `examples`, `preamble`, and `additionalRules` carry over as `promptOptions` |
| Correction | App loop or Autofix | Built in; remove the app loop and Autofix from this path |

1. **Keep Chat Completions first.** Change only the table's rows. The route's `.asResponse()` stream and the client's `openAIAdapter()` stay the same.
2. **Keep storage.** In-memory, `restStorage`, or a custom `ChatStorage` all keep working. To move threads into Gateway while staying on Chat Completions, call `storeChatCompletionHistory` after each turn ([conversations.md](conversations.md#store-chat-completions-turns)).
3. **Move to Responses only when needed.** Hosted tools or Gateway-injected history need a Responses route, `openAIResponsesAdapter()` with `openAIConversationMessageFormat` on the client, a frontend-token route, and `useOpenuiCloudStorage`. The [Gateway template](quickstart.md#know-the-generated-app) is the reference implementation.

## Migrate Stored Data

Changing the generation endpoint does not move existing threads. When moving storage to Conversations:

1. Keep historical data readable in its current store; no first-party import API is documented.
2. Preserve old thread ids, or keep an explicit mapping to new conversation ids.
3. Test user and app isolation before removing the old storage routes.
4. Tell the user what was migrated and what stayed in the old store.

## Verify

1. Requests use `THESYS_API_KEY`, the Gateway base URL, and `{provider}/{model}` ids.
2. Streaming, cancellation, and error behavior match the app before migration.
3. A function-tool flow and history reload still work.
4. With generated UI, output uses the expected library and renders; only Gateway corrects it.
5. The previous provider configuration is removed only after these checks pass.

## First-Party References

- `https://www.openui.com/docs/gateway/migrate-to-openui-gateway`
- `https://www.openui.com/docs/gateway/generate-openui-lang`
