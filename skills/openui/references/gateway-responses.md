# OpenUI Gateway (Responses API)

Use this only when the app already calls the Responses API. For new work, use Chat Completions; see [gateway.md](gateway.md). Configuration, BYOK, security, and verification are the same as there.

## Make a request

Same client and base URL as Chat Completions. The OpenUI prompt goes in `instructions`:

```ts
import { generateSystemPrompt, type LibrarySpec } from "@openuidev/lang-core";
import library from "@/lib/openui.spec.json";

const response = await gateway.responses.create({
  model: "openai/gpt-5",
  input,
  instructions: generateSystemPrompt({ cloud: true, library: library as LibrarySpec }),
  stream: true,
});
```

**Endpoint:** `POST https://api.thesys.dev/v1/embed/responses`. Relay the event stream unchanged. In Agent Interface, read it with `openAIResponsesAdapter()`.

## Keep the app's history model

Responses has three ways to carry history. Keep the one the app uses, and use only one per flow:

| Pattern | The app sends | Use when |
| --- | --- | --- |
| Full history in `input` | Every relevant input item, every turn | The app stores all messages |
| `previous_response_id` with `store: true` | Only the new turn | Chaining stored responses is enough; no named threads |
| `conversation` with `store: true` | Only the new turn | Threads live in a Gateway conversation ([Conversations API](gateway.md#conversations-api)) |

With `conversation`, Gateway reads and appends the thread's items itself, so the route must not also resend history. In Agent Interface, pair `openAIResponsesAdapter()` with `openAIConversationMessageFormat`. The route holds the organization key, so check that the signed-in user owns the conversation id before calling Gateway (see [gateway.md](gateway.md#check-who-owns-a-thread)).

### Chain with `previous_response_id`

- **Keep the id on the server**, keyed by signed-in user and thread. Never accept a response id from the browser: the route uses the organization key, so a browser-supplied id could continue another user's response.
- **Capture it when the response completes**, from the `response.completed` event (`event.response.id`). A cancelled or failed response must not become the base of the next turn.
- **The thread id** comes from the request body: Agent Interface's `fetchLLM` sends `threadId`.
- **Send `store: true` on every request**, and only the new user input with `previous_response_id`. Send `instructions` on every request too; they are not carried over from the previous response.
- **Fall back to full history in `input`** when there is no stored id, or it no longer matches what the user sees: the first turn, after a regenerate or edit, after a cancelled turn, after a server restart. Then start a new chain from that response.
- **Agent Interface still sends the full `messages`.** The route sends only the trailing user items while chaining.
- No frontend tokens or Conversations API calls are involved.

## Hosted tools

Gateway can run some tools itself, inside the request:

| Capability | Declaration | Runs in |
| --- | --- | --- |
| Web search | `{ type: "web_search" }` | Gateway |
| Image search | `{ type: "image_search" }` | Gateway |
| Remote MCP server | `{ type: "mcp", server_label, server_url }` | Gateway |
| Application function | `{ type: "function", name, parameters }` | The app's server |

```ts
import type { Tool } from "openai/resources/responses/responses";

tools: [
  { type: "web_search" },
  { type: "image_search" } as unknown as Tool,
  { type: "mcp", server_label: "deepwiki", server_url: "https://mcp.deepwiki.com/mcp" } as unknown as Tool,
],
```

The casts are needed while image search and MCP are missing from the OpenAI SDK's tool types. Declare MCP servers on every request that needs them. Load any MCP auth headers from server-side secrets.

For a function tool, run each returned `function_call` on the server and continue with a matching `function_call_output`. Run only tools the app declared; never run calls Gateway owns.

## Sources

- [Responses API](https://www.openui.com/docs/gateway/api/responses)
- [Hosted tools](https://www.openui.com/docs/gateway/api/responses/hosted-tools)
- [Conversations API](https://www.openui.com/docs/gateway/api/conversations)
