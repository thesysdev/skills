# Gateway Responses API

Use this guide for new Gateway chat or agent apps, existing Responses apps, and apps that need hosted tools. Read [overview.md](overview.md) first for endpoints, the prompt, and route security. An existing Chat Completions app moves here only when the user chooses to migrate.

```text
POST https://api.thesys.dev/v1/embed/responses
```

The endpoint accepts Responses-compatible requests, streams Responses events, corrects OpenUI Lang when the `cloud: true` prompt is set, and runs hosted tools.

## Choose the History Model

Use exactly one:

| Model | Use when | App responsibility |
| --- | --- | --- |
| `conversation` plus `store: true` | New apps with persistent, listable threads | Authorize the conversation id and send only the new turn; follow [conversations.md](conversations.md) |
| Full `input` history | The app already owns storage or needs explicit control over context | Load, authorize, bound, and resend the relevant input items each turn |
| `previous_response_id` | A chain of stored responses is enough; no named thread list | Store and authorize the latest response id; send only the new turn with `store: true` |

Never combine `conversation` with full history or with `previous_response_id`. Frontend tokens and Gateway browser storage apply only to the `conversation` model. An existing Responses app keeps its history model unless the user asks to change it.

## Build the Request

```ts
import OpenAI from "openai";

const gateway = new OpenAI({
  apiKey: process.env.THESYS_API_KEY,
  baseURL: "https://api.thesys.dev/v1/embed",
});

const common = { model, instructions, stream: true as const }; // instructions from generateSystemPrompt({ cloud: true, library })

// Choose one of the following:
const named = await gateway.responses.create(
  { ...common, input: latestTurn, conversation: authorizedConversationId, store: true },
  { signal: req.signal },
);
const full = await gateway.responses.create(
  { ...common, input: authorizedInputHistory },
  { signal: req.signal },
);
const chained = await gateway.responses.create(
  { ...common, input: latestTurn, previous_response_id: authorizedPreviousResponseId, store: true },
  { signal: req.signal },
);
```

The `authorized*` values come from the app's own authentication and persistence. For a named conversation, check ownership before the call, as described in [conversations.md](conversations.md#authorize-the-generation-route).

On top of the [shared route rules](overview.md#protect-the-server-boundary), rebuild only allowed Responses input items from the request body instead of forwarding provider items from the browser. Support attachments and content parts only after checking model support and size limits.

## Relay the Stream

A direct proxy returns the Responses SSE events unchanged, including errors and completion, and the client uses `openAIResponsesAdapter()`. When a framework owns the agent loop, keep its browser stream and adapter. See [Agent Interface stream wiring](../agent-interface.md#match-the-browser-stream).

## Add Hosted Tools

Gateway runs hosted tools inside the Responses request; the app runs its own function tools.

| Tool | Declaration | Runs in |
| --- | --- | --- |
| Web search | `{ type: "web_search" }` | Gateway |
| Image search | `{ type: "image_search" }` | Gateway |
| Remote MCP server | `{ type: "mcp", server_label, server_url, headers? }` | Gateway |
| App function | `{ type: "function", name, description, parameters }` | App server |

```ts
import type { Tool } from "openai/resources/responses/responses";

const tools = [
  { type: "web_search" },
  { type: "image_search" } as unknown as Tool,
  { type: "mcp", server_label: "deepwiki", server_url: "https://mcp.deepwiki.com/mcp" } as unknown as Tool,
];
```

The casts cover Gateway tool types missing from the OpenAI SDK's union; keep them limited to those entries. Hosted tools work with every history model.

Declare MCP servers on each request that uses them. Load authenticated MCP headers from server-side secrets and send them only to approved origins. If an MCP server's tools never appear, read the `error` field of its `mcp_list_tools` output item.

## Run App Function Tools

The model emits a `function_call`; the app runs the function and continues with a `function_call_output` until the model returns a final answer. Copy the loop from the Gateway template's `src/lib/tool-loop.ts` into the app and keep its safeguards:

1. Execute only tool names the app declared and registered. Never execute Gateway-owned calls, such as names starting with `thesys_`.
2. Skip a call whose `function_call_output` already appears in the same stream.
3. Validate every argument and cap the number of continuation rounds.

Responses uses `function_call` and `function_call_output` items; do not reuse a Chat Completions `role: "tool"` loop here. When the UI should show a tool result or open it as an [artifact](../artifacts.md), make sure the stream carries that result to the browser.

## Verify

1. Run the [shared checks](overview.md#verify).
2. Requests go to `/v1/embed/responses`, and the client receives the stream format its adapter expects.
3. Exactly one history model is active. Full history is loaded and bounded each turn; a response chain stores ids and sets `store: true`; named conversations pass every check in [conversations.md](conversations.md#verify).
4. Injected provider items, oversized input, Gateway 4xx/5xx, and cancellation are handled.
5. Every declared hosted and app tool runs through its owner, and the app never executes Gateway-owned calls.

## First-Party References

- `https://www.openui.com/docs/gateway/api/responses`
- `https://www.openui.com/docs/gateway/api/responses/hosted-tools`
- `https://github.com/thesysdev/openui/blob/main/templates/openui-cloud/src/app/api/chat/route.ts`
