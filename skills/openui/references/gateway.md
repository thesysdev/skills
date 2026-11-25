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
- **Fallbacks** depend on which models are compatible and how the account is configured. If the user needs one exact model or data residency, check the fallback behavior before relying on it.
- **A model in the catalog** may still lack tools or a modality the app needs. Check before adding it to an allowlist.

## Use the user's provider account (BYOK)

To use an OpenAI, Anthropic, or Google subscription through Gateway, an organization admin adds the provider key on the [BYOK page of the Thesys Console](https://console.thesys.dev/byok). Tell the user where to add it; never ask for the key.

- BYOK is on every plan, including the free tier.
- A provider key applies only to its provider: an OpenAI key covers `openai/*` models. Other providers use Gateway credits, and fail when the organization has none.
- The app keeps authenticating with `THESYS_API_KEY`. No code changes.
- Only organization admins can add keys. Explain which credential the console asks for without prescribing fields (for example Google's), let the user enter it, and continue after they confirm.

## Generate OpenUI Lang

Send the system prompt built from the library spec as the first message:

```ts
import { generateSystemPrompt, type LibrarySpec } from "@openuidev/lang-core";
import library from "@/lib/openui.spec.json";
import { promptOptions } from "@/lib/prompt-options";

const completion = await gateway.chat.completions.create({
  model: "openai/gpt-5",
  messages: [
    {
      role: "system",
      content: generateSystemPrompt({ cloud: true, library: library as LibrarySpec, promptOptions }),
    },
    ...messages,
  ],
  stream: true,
});
```

- `cloud: true` produces a config block carrying the library. Gateway reads it and validates and corrects the output against that library while it streams. The spec must come from the same library the browser renders; see [component-library.md](component-library.md).
- **Pass `promptOptions`.** The spec file does not carry the library's prompt options. In cloud mode, only `examples`, `preamble`, and `additionalRules` are sent; see [component-library.md](component-library.md#4-prompt-options).
- `instructions` adds the app's own trusted instructions after the config block. Never put user-written text in `instructions`, `preamble`, rules, or examples.
- **One OpenUI prompt per request.** When migrating an app that already sends a full OpenUI prompt (`library.prompt()`, or `generateSystemPrompt` without `cloud`), replace it; do not send both. Do not copy earlier system messages from history into later turns.
- Requests without the OpenUI system prompt are plain model calls: Gateway routes them but does not correct them.

What correction does not do:
- make output deterministic: measure across repeated runs (see [reliability.md](reliability.md));
- validate business data, or make tool execution safe;
- repair arbitrary JSON or component code.

To cut prompt cost and latency, keep the system prompt and spec prefix stable across requests; see [Prompt caching](https://www.openui.com/docs/gateway/prompt-caching).

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
- **A hand-built chat that reads plain text:** stream only the text deltas, and read them in the browser:

  ```ts
  // server
  const body = new ReadableStream({
    async start(controller) {
      for await (const chunk of completion) {
        const text = chunk.choices[0]?.delta?.content;
        if (text) controller.enqueue(encoder.encode(text));
      }
      controller.close();
    },
  });
  return new Response(body, { headers: { "Content-Type": "text/plain; charset=utf-8" } });
  ```

  ```ts
  // browser
  const res = await fetch("/api/chat", { method: "POST", body: JSON.stringify({ messages }), signal });
  const reader = res.body!.getReader();
  const decoder = new TextDecoder();
  let content = "";
  setStreaming(true);
  for (;;) {
    const { done, value } = await reader.read();
    if (done) break;
    content += decoder.decode(value, { stream: true });
    setContent(content); // <Renderer response={content} isStreaming={streaming} … />
  }
  setStreaming(false);
  ```
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
   // lib/frontend-token.ts
   export async function mintFrontendToken(userId: string): Promise<{ token: string; expires_at: number }> {
     const res = await fetch("https://api.thesys.dev/v1/frontend-tokens", {
       method: "POST",
       headers: {
         Authorization: `Bearer ${process.env.THESYS_API_KEY}`,
         "Content-Type": "application/json",
       },
       body: JSON.stringify({ user_id: userId, ...(process.env.APP_ID ? { app_id: process.env.APP_ID } : {}) }),
       cache: "no-store",
     });
     if (!res.ok) throw new Error(`Frontend token mint failed: ${res.status}`);
     return res.json();
   }
   ```

   ```ts
   // app/api/frontend-token/route.ts
   export async function POST(request: Request) {
     const userId = await getAuthenticatedUserId(request); // from the server session, never the request body
     if (!userId) return Response.json({ error: "Unauthorized" }, { status: 401 });
     try {
       const { token, expires_at } = await mintFrontendToken(userId);
       return Response.json({ token, expires_at }, { headers: { "Cache-Control": "private, no-store" } });
     } catch {
       return Response.json({ error: "Token mint failed" }, { status: 502 });
     }
   }
   ```

   - `user_id` must be a stable account id, never a display name or an email that can change.
   - `app_id` is optional. Give each product its own fixed value when one organization serves several. Never derive it from the API key: rotating the key would change it.
   - Conversation `metadata` is for app data, never for deciding who owns a thread.
3. **Chat route:** the browser storage does not write messages. After a turn completes, the chat route saves the **new turn only** (never the replayed history) with `@openuidev/server` 0.6.0 or later, as the signed-in user. Do not save a cancelled or failed turn. A route that relays Chat Completions SSE collects the text as it streams and saves after the last chunk:

   ```ts
   // lib/openui.ts
   import { OpenUIClient } from "@openuidev/server";

   // Reads THESYS_API_KEY. Use the same app id the frontend-token route mints with.
   export const openui = new OpenUIClient({ appId: process.env.APP_ID || undefined });
   ```

   ```ts
   import { openui } from "@/lib/openui";

   export async function POST(request: Request) {
     const userId = await getAuthenticatedUserId(request);
     if (!userId) return Response.json({ error: "Unauthorized" }, { status: 401 });
     const { threadId, messages } = await readChatBody(request); // validated; see "Secure the route"
     await assertThreadOwner(userId, threadId);                  // see "Check who owns a thread"

     const completion = await gateway.chat.completions.create(
       { model, messages: [systemMessage, ...messages], stream: true },
       { signal: request.signal },
     );

     const encoder = new TextEncoder();
     let assistantText = "";
     const body = new ReadableStream({
       async start(controller) {
         try {
           for await (const chunk of completion) {                // throws if the request is cancelled
             assistantText += chunk.choices[0]?.delta?.content ?? "";
             controller.enqueue(encoder.encode(`data: ${JSON.stringify(chunk)}\n\n`));
           }
           controller.enqueue(encoder.encode("data: [DONE]\n\n"));
           controller.close();
         } catch (error) {
           controller.error(error);
           return;                                                // cancelled or failed: save nothing
         }
         await openui.chat.conversations.items
           .create(threadId, [messages.at(-1), { role: "assistant", content: assistantText }], { userId })
           .catch(() => console.error("[chat] failed to save turn")); // never log message contents
       },
     });
     return new Response(body, { headers: { "Content-Type": "text/event-stream", "Cache-Control": "no-cache" } });
   }
   ```

   - Pass the user message as the browser sent it (`openAIMessageFormat` already produces a Chat Completions message); content parts are fine.
   - If the app runs a tool loop, save the assistant tool-call message and every `role: "tool"` result of the turn too, in order. A tool call is saved only together with its result, so a turn stopped mid tool call leaves the thread valid.
   - To load the thread on the server instead of trusting the browser's replay, `openui.chat.conversations.items.list(threadId, { userId })` returns all of it, oldest first, as Chat Completions messages.

   For a LangGraph agent, `openui.langgraph.conversations.items.create(threadId, messages, { userId })` takes the turn's `BaseMessage[]` and saves the user message, assistant messages, tool calls, and tool results in one call. The default scaffold saves `state.messages.slice(state.turnStart)` from an `afterAgent` middleware. `items.list` returns `{ role, content }` messages that `graph.invoke({ messages })` takes. System and developer messages are skipped.

   `storeChatCompletionHistory` and `storeLangGraphHistory` are deprecated: they save as the organization, with no ownership check. Where the app or scaffold calls them, switch to the calls above with the user's id.

The chat route still sends the full history to Chat Completions on every turn.

### Check who owns a thread

The chat route holds the organization key, so a `threadId` from the browser is not proof of ownership. Pass the signed-in user's id as `userId` on every `openui.conversations` call: the call then acts as that user, like their frontend token, and a thread that isn't theirs fails with a 404.

```ts
import { OpenUIApiError } from "@openuidev/server";
import { openui } from "@/lib/openui";

export async function assertThreadOwner(userId: string, threadId: string) {
  try {
    await openui.conversations.get(threadId, { userId });
  } catch (error) {
    if (error instanceof OpenUIApiError && error.status === 404) throw new HttpError(404, "Thread not found");
    throw error;
  }
}
```

- Without `userId`, a call acts as the organization and skips this check.
- `OpenUIClient` must use the same `appId` as the frontend-token route, or the threads the browser made 404.
- Never take `userId` from the request body, and never decide ownership from conversation `metadata`.

`HttpError` stands for the app's own error handling.

### Apps without sign-in

Frontend tokens and ownership checks need a user id from the server session.

- **The app has authentication:** use its user id.
- **It has none:** ask the user how people will sign in. For a local demo, a fixed `DEMO_USER_ID`, as in the scaffold, is fine. Tell the user that everyone then shares one thread list.

Never take the user id from the request body.

### Conversations API

The browser hook and `@openuidev/server` call this API:

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` / `POST` | `/v1/conversations` | List or create conversations |
| `GET` / `POST` / `DELETE` | `/v1/conversations/{conversation_id}` | Retrieve, update, or delete a conversation |
| `GET` / `POST` | `/v1/conversations/{conversation_id}/items` | List or add items |
| `GET` / `POST` | `/v1/conversations/{conversation_id}/items/{item_id}` | Retrieve or update one item |

- **Server calls** authenticate with `Authorization: Bearer $THESYS_API_KEY`. Use `openui.conversations` rather than raw HTTP: `list`, `create`, `get`, `update`, `delete`, and `items.list` / `create` / `get` / `update`. Each takes `{ userId }` to act as that user; `create` requires it, so the user's browser can read the thread.
- **`openui.chat.conversations` / `openui.langgraph.conversations`** are the same calls, but `items.create` takes Chat Completions or LangChain messages, and `items.list` reads the whole thread back as them.
- **Frontend-token calls** send the token in the `x-thesys-frontend-token` header and reach only that user's conversations. `useOpenuiCloudStorage` does this itself.

Moving existing threads into Gateway conversations is a separate data migration. Keep old threads readable where they are, keep a mapping from old thread ids, and never claim history was imported unless a verified import ran.

## Run function tools

Chat Completions returns tool calls; the application runs them:

1. Send the messages and the function declarations.
2. Read `tool_calls` from the assistant message.
3. Check each name against the tools the app declared, and validate the arguments.
4. Run the tool on the server.
5. Append the assistant tool-call message and one `role: "tool"` result per call.
6. Repeat, with a maximum number of rounds, until the model answers with OpenUI Lang.

Agent frameworks (LangGraph, the Vercel AI SDK) already run this loop; add tools the framework's way.

Hosted tools (`web_search`, `image_search`, remote MCP) work only with the Responses API. Never declare them on Chat Completions; see [gateway-responses.md](gateway-responses.md#hosted-tools).

## Secure the route

- Authenticate and rate-limit the chat route and the frontend-token route. Page authentication does not protect API routes.
- Require JSON, enforce a byte limit before parsing, then validate the body: message roles, shapes, and content parts. Do not forward arbitrary browser input to the model.
- When the app stores history, load it on the server instead of trusting the history the browser sends. Accept only complete assistant tool-call / tool-result pairs.
- Keep attachments and other content parts only after checking that the model supports them, with size limits.
- Keep the model allowlist, tool list, and system prompt on the server.
- Return upstream errors without logging keys or message contents.

## Verify

1. Requests reach `https://api.thesys.dev/v1/embed` with a `{provider}/{model}` id.
2. The response streams to the browser and renders progressively; cancelling stops it.
3. A tool call runs once, and its result stays in history.
4. With Gateway conversations: reload restores the thread with its tool calls and results in order, and a second user cannot see or write the first user's threads. An expired frontend token is refreshed.
5. Each request carries exactly one OpenUI prompt.
6. A signed-out request to the chat or token route is rejected, and an oversized or malformed body is rejected before reaching Gateway.
7. `THESYS_API_KEY` appears in no client file or browser bundle.

## Sources

- [Gateway](https://www.openui.com/docs/gateway)
- [Chat Completions](https://www.openui.com/docs/gateway/api/chat-completions)
- [Conversations API](https://www.openui.com/docs/gateway/api/conversations)
- [Authentication](https://www.openui.com/docs/gateway/authentication)
- [BYOK](https://www.openui.com/docs/gateway/byok)
- [Manage conversations from your server](https://www.openui.com/docs/gateway/conversations-from-server)
- [`@openuidev/server`](https://www.openui.com/docs/api-reference/server)
- [Pricing](https://www.openui.com/docs/gateway/pricing-credits)
