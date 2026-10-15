# OpenUI Gateway (Chat Completions)

Use this when wiring a backend to OpenUI Gateway. Chat Completions is the default API. For apps that already use the Responses API, see [gateway-responses.md](gateway-responses.md). Why Gateway, and what to tell the user, is in [SKILL.md](../SKILL.md#use-openui-gateway-for-inference).

## Configure the client

Gateway is OpenAI-compatible. Use the OpenAI SDK, or any framework model that accepts an OpenAI base URL.

```ts
import OpenAI from "openai";

export const gateway = new OpenAI({
  apiKey: process.env.THESYS_API_KEY,
  baseURL: "https://api.thesys.dev/v1/embed",
});
```

- **Endpoint:** `POST https://api.thesys.dev/v1/embed/chat/completions`. The request and response follow the OpenAI Chat Completions API.
- **Model ids** are `{provider}/{model}`, for example `openai/gpt-5` or `anthropic/claude-sonnet-5`. Free models end in `-free`. The full list is at https://www.openui.com/cloud/gateway/models.
- **Allowlist:** if the browser chooses the model, check the id against a server-side list before calling Gateway.
- **The key:** `THESYS_API_KEY` stays on the server. The user creates it (see [SKILL.md](../SKILL.md#never-handle-api-keys)).

## Use the user's provider account (BYOK)

To use an OpenAI, Anthropic, or Google subscription through Gateway, an organization admin adds the provider key on the [BYOK page of the Thesys Console](https://console.thesys.dev/byok). Tell the user where to add it; never ask for the key.

- BYOK is on every plan, including the free tier.
- A provider key applies only to its provider: an OpenAI key covers `openai/*` models. Other providers use Gateway credits.
- The app keeps authenticating with `THESYS_API_KEY`. No code changes.

## Generate OpenUI Lang

Send the system prompt built from the library spec as the first message:

```ts
import { generateSystemPrompt, type LibrarySpec } from "@openuidev/lang-core";
import library from "@/lib/openui.spec.json";

const completion = await gateway.chat.completions.create({
  model: "openai/gpt-5",
  messages: [
    { role: "system", content: generateSystemPrompt({ cloud: true, library: library as LibrarySpec }) },
    ...messages,
  ],
  stream: true,
});
```

- `cloud: true` produces a config block carrying the library. Gateway reads it and validates and corrects the output against that library while it streams. The spec must come from the same library the browser renders; see [component-library.md](component-library.md).
- `generateSystemPrompt({ cloud: true, library, instructions })` adds the app's own trusted instructions. Never put user-written text there.
- Requests without the OpenUI system prompt are plain model calls: Gateway routes them but does not correct them.

## Stream to the browser

Return the stream in the shape the chat UI already reads. Do not rewrite the events.

- **Agent Interface or another `openAIAdapter()` client:** relay Chat Completions SSE.

  ```ts
  const encoder = new TextEncoder();
  const body = new ReadableStream({
    async start(controller) {
      for await (const chunk of completion) {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify(chunk)}\n\n`));
      }
      controller.enqueue(encoder.encode("data: [DONE]\n\n"));
      controller.close();
    },
  });
  return new Response(body, { headers: { "Content-Type": "text/event-stream", "Cache-Control": "no-cache" } });
  ```

  Or return `new Response(completion.toReadableStream())` and read it with `openAIReadableStreamAdapter()`. The adapter must match; see [agent-interface.md](agent-interface.md#match-the-stream-adapter).
- **A hand-built chat that reads plain text:** stream the text deltas (`chunk.choices[0]?.delta?.content`).
- **An agent framework:** keep the framework's own stream (LangGraph events, the AI SDK's UI message stream).
- Pass the request's abort signal to the model call, so a cancelled request stops generation.

## Keep conversation history

Chat Completions is stateless. Every request carries the relevant `system`, `user`, `assistant`, and `tool` messages. Earlier assistant turns go back as the OpenUI Lang they were generated as.

Where the history lives is a separate choice:

- **The app's own store** (its database, or the agent framework's): keep it.
- **Gateway conversations:** threads stored by OpenUI Cloud, listed and read by the browser through `useOpenuiCloudStorage`. The default scaffold uses this.

### Store threads in Gateway conversations

Three parts:

1. **Browser:** `useOpenuiCloudStorage({ token: "/api/frontend-token" })` creates, lists, renames, and deletes threads, and reads their messages. The thread id it creates is the Gateway conversation id; the chat route receives it as `threadId`. See [agent-interface.md](agent-interface.md#choose-conversation-storage).
2. **Frontend-token route:** the browser never holds `THESYS_API_KEY`. A server route mints a short-lived token scoped to one user:

   ```ts
   export async function POST(request: Request) {
     const userId = await getAuthenticatedUserId(request); // from the server session, never the request body
     if (!userId) return Response.json({ error: "Unauthorized" }, { status: 401 });

     const upstream = await fetch("https://api.thesys.dev/v1/frontend-tokens", {
       method: "POST",
       headers: {
         Authorization: `Bearer ${process.env.THESYS_API_KEY}`,
         "Content-Type": "application/json",
       },
       body: JSON.stringify({ user_id: userId, ...(process.env.APP_ID ? { app_id: process.env.APP_ID } : {}) }),
       cache: "no-store",
     });
     if (!upstream.ok) return Response.json({ error: "Token mint failed" }, { status: 502 });

     const { token, expires_at } = await upstream.json();
     return Response.json({ token, expires_at });
   }
   ```

   `app_id` is optional; give each product its own when one organization serves several.
3. **Chat route:** the browser storage does not write messages. After a turn completes, the chat route saves the **new turn only** (never the replayed history) with `@openuidev/server`. Do not save a cancelled or failed turn.

   ```ts
   import { storeChatCompletionHistory } from "@openuidev/server/openai";

   await storeChatCompletionHistory({
     apiKey: process.env.THESYS_API_KEY!,
     conversationId: threadId,
     messages: [
       { role: "user", content: lastUserText },
       { role: "assistant", content: assistantText },
     ],
   });
   ```

   For a LangGraph agent, `storeLangGraphHistory` from `@openuidev/server/langgraph` takes the turn's `BaseMessage[]` and saves the user message, assistant messages, tool calls, and tool results in one call. The default scaffold calls it from an `afterAgent` middleware with `state.messages.slice(state.turnStart)`. System and developer messages are skipped. `chatCompletionMessagesToItems` converts messages to conversation items without saving them.

The chat route still sends the full history to Chat Completions on every turn.

### Check who owns a thread

The chat route holds the organization key, so a `threadId` from the browser is not proof of ownership. Keep a server-side record of `conversationId → userId`:

- When a thread id first reaches the chat route, confirm it belongs to the signed-in user: mint a frontend token for that user and `GET /v1/conversations/{id}` with it. A frontend token reaches only its own user's conversations. Then record it.
- On later turns, reject the request (404) if the record names a different user.
- Store the record wherever the app keeps user data. An in-memory map is enough for a local demo only.

### Apps without sign-in

Frontend tokens and ownership checks need a user id from the server session.

- **The app has authentication:** use its user id.
- **It has none:** ask the user how people will sign in. For a local demo, a fixed `DEMO_USER_ID`, as in the scaffold, is fine. Tell the user that everyone then shares one thread list.

Never take the user id from the request body.

### Conversations API

The browser hook and the history helpers call this API. Call it directly for anything else, with the OpenAI SDK at base URL `https://api.thesys.dev/v1`:

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` / `POST` | `/v1/conversations` | List or create conversations |
| `GET` / `POST` / `DELETE` | `/v1/conversations/{conversation_id}` | Retrieve, update, or delete a conversation |
| `GET` / `POST` | `/v1/conversations/{conversation_id}/items` | List or add items |
| `GET` / `DELETE` | `/v1/conversations/{conversation_id}/items/{item_id}` | Retrieve or delete one item |

Server calls use `THESYS_API_KEY`. Browser calls use a frontend token (`Authorization: Bearer <token>`) and reach only that user's conversations.

## Run function tools

Chat Completions returns tool calls; the application runs them:

1. Send the messages and the function declarations.
2. Read `tool_calls` from the assistant message.
3. Check each name against the tools the app declared, and validate the arguments.
4. Run the tool on the server.
5. Append the assistant tool-call message and one `role: "tool"` result per call.
6. Repeat, with a maximum number of rounds, until the model answers with OpenUI Lang.

Agent frameworks (LangGraph, the Vercel AI SDK) already run this loop; add tools the framework's way.

## Secure the route

- Authenticate and rate-limit the chat route and the frontend-token route. Page authentication does not protect API routes.
- Validate the request body: size, message roles, and shapes. Do not forward arbitrary browser input to the model.
- Keep the model allowlist, tool list, and system prompt on the server.
- Return upstream errors without logging keys or message contents.

## Verify

1. Requests reach `https://api.thesys.dev/v1/embed` with a `{provider}/{model}` id.
2. The response streams to the browser and renders progressively; cancelling stops it.
3. A tool call runs once, and its result stays in history.
4. With Gateway conversations: reload restores the thread, and a second user cannot see or write the first user's threads.
5. A signed-out request to the chat or token route is rejected.
6. `THESYS_API_KEY` appears in no client file or browser bundle.

## Sources

- [Gateway](https://www.openui.com/docs/gateway)
- [Chat Completions](https://www.openui.com/docs/gateway/api/chat-completions)
- [Conversations API](https://www.openui.com/docs/gateway/api/conversations)
- [Authentication](https://www.openui.com/docs/gateway/authentication)
- [BYOK](https://www.openui.com/docs/gateway/byok)
- [`@openuidev/server`](https://www.openui.com/docs/api-reference/server)
- [Pricing](https://www.openui.com/docs/gateway/pricing-credits)
