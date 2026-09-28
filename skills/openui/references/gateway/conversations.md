# Gateway Conversations

Conversations stores threads and their items in Gateway. It is a storage and identity layer, not a generation protocol. Read [overview.md](overview.md) first. Use this guide for persistent Gateway threads, item access, browser thread storage, scoped frontend tokens, or per-user and per-app isolation.

| Generation API | How turns reach a conversation |
| --- | --- |
| Responses with `conversation` and `store: true` | Gateway reads earlier items and writes the new turn automatically |
| Chat Completions | The route writes each completed turn with [`storeChatCompletionHistory`](#store-chat-completions-turns) |

Responses with full `input` or `previous_response_id` does not use Conversations, frontend tokens, or conversation ownership checks; the app still authorizes its own stored history or response ids. See [responses.md](responses.md#choose-the-history-model).

## Create and Read Conversations

Server calls use the stock OpenAI SDK with base URL `https://api.thesys.dev/v1`:

```ts
import OpenAI from "openai";

const conversations = new OpenAI({
  apiKey: process.env.THESYS_API_KEY,
  baseURL: "https://api.thesys.dev/v1",
});

const conversation = await conversations.conversations.create({ metadata: { workspace: "acme" } });
const page = await conversations.conversations.items.list(conversation.id, { order: "asc", limit: 100 });
```

Create the conversation before the first stored turn, then pass its id as the Responses `conversation` value or the `conversationId` below.

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` / `POST` | `/v1/conversations` | List or create conversations |
| `GET` / `POST` / `DELETE` | `/v1/conversations/{conversation_id}` | Retrieve, update, or delete a conversation |
| `GET` / `POST` | `/v1/conversations/{conversation_id}/items` | List or add items |
| `GET` / `DELETE` | `/v1/conversations/{conversation_id}/items/{item_id}` | Retrieve or delete one item |

Prefer the SDK methods. Check pagination, update fields, and item types against the installed SDK before writing raw HTTP helpers.

## Store Chat Completions Turns

A Chat Completions route can keep its protocol and still store threads in Gateway. After each completed turn, write only that turn's messages:

```ts
import { storeChatCompletionHistory } from "@openuidev/server/openai";

await storeChatCompletionHistory({
  apiKey: process.env.THESYS_API_KEY!,
  conversationId: authorizedConversationId,
  messages: [
    { role: "user", content: lastUserText },
    { role: "assistant", content: assistantText },
  ],
});
```

Sending the full `messages` array duplicates items. System and developer messages are skipped, and assistant tool calls and tool results are stored as function items. `chatCompletionMessagesToItems` converts messages without posting them. The request to the model still carries the relevant history, as in [chat-completions.md](chat-completions.md#keep-history-in-the-app).

## Mint Frontend Tokens

The browser reads and edits conversations directly with a short-lived token scoped to one user and, optionally, one app. Agent Interface uses it through `useOpenuiCloudStorage` ([Agent Interface storage](../agent-interface.md#choose-conversation-storage)). The server mints it with `POST https://api.thesys.dev/v1/frontend-tokens`, taking `user_id` from the authenticated session and never from the request body:

```ts
export async function POST(req: Request) {
  const userId = await getAuthenticatedUserId(req);
  if (!userId) return Response.json({ error: { message: "Unauthorized" } }, { status: 401 });

  const upstream = await fetch("https://api.thesys.dev/v1/frontend-tokens", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${process.env.THESYS_API_KEY}`,
    },
    body: JSON.stringify({ user_id: userId, ...(process.env.APP_ID ? { app_id: process.env.APP_ID } : {}) }),
    cache: "no-store",
  });
  if (!upstream.ok) {
    return Response.json({ error: { message: "Unable to mint frontend token" } }, { status: 502 });
  }

  const { token, expires_at } = (await upstream.json()) as { token: string; expires_at: number };
  return Response.json({ token, expires_at }, { headers: { "Cache-Control": "private, no-store" } });
}
```

Rate-limit this route. Frontend tokens work only for Conversations endpoints; they never authorize the app's generation route.

## Scope Users and Apps

- Use an immutable account id as `user_id`, not a display name, a changeable email, or a browser-supplied id.
- Use one stable `app_id` per product surface when one organization key serves several apps. Do not derive it from the API key, because rotating the key would change storage identity.
- The Gateway template's `DEMO_USER_ID` is for local demos only; replace it before real users.
- Keep app data in conversation `metadata`, but never use metadata to decide ownership.
- Use the same identity for token minting, generation authorization, and server-side conversation calls.

## Authorize the Generation Route

The generation route calls Gateway with the server key, so holding a `threadId` does not prove the user owns that conversation. Before each call:

1. Authenticate and rate-limit the route on its own.
2. Treat `threadId` as untrusted, and check it against a server-side record of `{ conversationId, ownerUserId, appId? }` written when the app created or bound the conversation.
3. Rebuild one bounded latest user turn instead of forwarding provider items from the browser.
4. Call Responses with `conversation: threadId` and `store: true`, or store the Chat Completions turn as shown above.

Never record ownership based on what the browser claims. If the app cannot establish ownership, keep the production route disabled and tell the user what is missing. Moving existing threads into Gateway is covered in [migrate.md](migrate.md#migrate-stored-data).

## Verify

1. Responses calls with `conversation` send only the new turn; Chat Completions routes store only the new turn.
2. The token route derives identity from the session, is rate-limited, and never returns `THESYS_API_KEY`.
3. The generation route rejects a conversation id owned by another user.
4. Create, retrieve, update, and delete a test conversation, and list its items across pages in order.
5. Reload the client and confirm the same threads and items return.
6. Two users, and two `app_id` values where used, see disjoint thread lists.
7. Token expiry and refresh, missing configuration, Gateway errors, and logged-out calls are handled.

## First-Party References

- `https://www.openui.com/docs/gateway/api/conversations`
- `https://www.openui.com/docs/gateway/authentication`
- `https://www.openui.com/docs/api-reference/server`
- `https://github.com/thesysdev/openui/blob/main/templates/openui-cloud/src/app/api/frontend-token/route.ts`
