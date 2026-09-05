# Integrate OpenUI Cloud with the Responses API

Read [existing-project.md](existing-project.md) first. Use this runbook for applications that already consume Responses events, new agents that need hosted tools, or workflows that need artifacts inside the agent stream. Do not apply it to an existing Chat Completions application unless the user has chosen a protocol migration.

Conversations is optional. Read [conversations.md](conversations.md) only when the application needs persistent named Cloud threads, item APIs, browser storage, frontend tokens, or Cloud user/app isolation.

## Contents

1. [Preserve the Responses Contract](#preserve-the-responses-contract)
2. [Choose the History Model](#choose-the-history-model)
3. [Configure the Generation Client](#configure-the-generation-client)
4. [Choose the Model-Facing Prompt](#choose-the-model-facing-prompt)
5. [Match the Client Stream](#match-the-client-stream)
6. [Adapt the Generation Route](#adapt-the-generation-route)
7. [Add Hosted Tools and Artifacts](#add-hosted-tools-and-artifacts)
8. [Run Application Function Tools](#app-owned-function-tools)
9. [Reliability and Observability](#reliability-and-observability)
10. [Verify](#verify)

## Preserve the Responses Contract

The Responses endpoint is:

```text
POST https://api.thesys.dev/v1/embed/responses
```

It provides Responses-compatible requests and events, managed generative UI, hosted tools, and artifacts inside agent turns. Preserve the host framework, route shape, authentication, history owner, renderer, component library, and working tool behavior unless the user requested a migration.

Do not conflate these choices:

- Responses is the generation protocol.
- Full `input`, `previous_response_id`, and `conversation` are alternative history models.
- Conversations is a named-thread storage API used by only one of those models.
- `AgentInterface` and `Renderer` are client presentation choices.

## Choose the History Model

Use exactly one Responses history pattern:

| Pattern | Use when | Application responsibility |
| --- | --- | --- |
| Full `input` history | The application already owns storage or needs explicit context control | Load, authorize, bound, and resend the relevant Responses input items |
| `previous_response_id` | Turns should form a stored response chain without a named/listable conversation | Persist and authorize the latest response id; send only the new turn with `store: true` |
| `conversation` plus `store: true` | The product needs persistent named Cloud threads, item CRUD, or `useOpenuiCloudStorage()` | Authorize the conversation id and send only the new turn; follow [conversations.md](conversations.md) |

Do not combine full history with `conversation`, or combine `previous_response_id` with `conversation`. Do not add frontend tokens or Cloud browser storage to the first two patterns.

When adapting an existing Responses application, preserve its current pattern unless the user explicitly requests a storage migration.

## Configure the Generation Client

Keep the key on the application server and use the stock OpenAI SDK:

```ts
import OpenAI from "openai";

const embedClient = new OpenAI({
  apiKey: process.env.THESYS_API_KEY,
  baseURL: "https://api.thesys.dev/v1/embed",
});
```

Use a current `{provider}/{model}` id selected through trusted server configuration. Preserve a host model allowlist and reject arbitrary browser-supplied model ids.

Install only the packages required by the selected runtime and features. Typical managed React integrations use `@openuidev/lang-core`, `@openuidev/react-ui`, `@openuidev/thesys`, `openai`, and the installed peer dependencies. Add `@openuidev/thesys-server` only for server helpers such as `artifactTool()`.

## Choose the Model-Facing Prompt

Use current managed prompt compilation from `@openuidev/lang-core`:

```ts
import { generateSystemPrompt } from "@openuidev/lang-core";

const instructions = generateSystemPrompt({
  cloud: true,
  instructions: trustedApplicationInstructions,
});
```

Pass the result as the Responses `instructions` value. Keep user-authored content out of trusted instructions, preambles, rules, and examples.

For a custom component library, follow [build-component-library.md](../build-component-library.md): generate a serialized spec, pass it as `library` with `cloud: true`, and render with the matching runtime library. Do not combine a custom client library with the built-in model-facing prompt.

## Match the Client Stream

Pair the Responses event stream with the Responses adapter:

```tsx
import {
  AgentInterface,
  fetchLLM,
  openAIConversationMessageFormat,
  openAIResponsesAdapter,
} from "@openuidev/react-ui";
import { chatLibrary } from "@openuidev/thesys";
import "@openuidev/thesys/styles.css";

const llm = fetchLLM({
  url: "/api/chat",
  streamAdapter: openAIResponsesAdapter(),
  messageFormat: openAIConversationMessageFormat,
});

export function CloudChat() {
  return <AgentInterface llm={llm} componentLibrary={chatLibrary} />;
}
```

Preserve the raw Responses SSE event shape through the application proxy. Do not use the Chat Completions adapters, parse the stream into a different protocol, or add another blind repair layer.

The exact browser request produced by a message format is version-sensitive. Inspect the installed formatter and route before validating or reconstructing input. If the application uses a custom UI, it may consume the Responses stream directly and render settled or streaming OpenUI Lang with `Renderer` instead of adopting `AgentInterface`.

When using Cloud browser storage, add the `storage` prop separately by following [conversations.md](conversations.md#connect-agent-interface-storage).

## Adapt the Generation Route

All history patterns share these route requirements:

- Authenticate and rate-limit the API route independently from the page or layout.
- Require JSON, enforce a byte limit before parsing, validate the body, and reconstruct only allowed Responses input items.
- Keep trusted instructions, credentials, model/tool configuration, and authorization decisions outside browser input.
- Preserve the host attachment and content-part behavior only after checking Cloud/model compatibility and explicit size bounds.
- Forward the request abort signal and stream Responses SSE unchanged.
- Return failures through the host's error contract without logging keys or sensitive content.

Build the request according to the chosen history owner. The alternatives below are mutually exclusive; implement only the selected block:

```ts
const common = {
  model,
  instructions: generateSystemPrompt({ cloud: true }),
  stream: true as const,
};

// Application-owned history:
const fullHistory = await embedClient.responses.create(
  { ...common, input: authorizedInputHistory },
  { signal: req.signal },
);

// Stored response chain:
const chained = await embedClient.responses.create(
  {
    ...common,
    input: latestTurn,
    previous_response_id: authorizedPreviousResponseId,
    store: true,
  },
  { signal: req.signal },
);

// Persistent named Cloud conversation:
const persistent = await embedClient.responses.create(
  {
    ...common,
    input: latestTurn,
    conversation: authorizedConversationId,
    store: true,
  },
  { signal: req.signal },
);
```

These snippets show the state distinction, not a drop-in route. The host must supply `authorizedInputHistory`, `authorizedPreviousResponseId`, or `authorizedConversationId` from its actual authentication and persistence boundaries. For named conversations, implement the separate ownership and frontend-token contract in [conversations.md](conversations.md#authorize-the-generation-route-separately).

## Add Hosted Tools and Artifacts

OpenUI Cloud executes hosted tools server-side inside the Responses request. Application function tools still execute on the application server.

| Capability | Declaration | Execution owner |
| --- | --- | --- |
| Slides and reports | `artifactTool({ artifacts: ["slides", "report"] })` | Cloud |
| Web search | `{ type: "web_search" }` | Cloud |
| Image search | `{ type: "image_search" }` | Cloud |
| Remote MCP server | `{ type: "mcp", server_label, server_url, headers? }` | Cloud |
| Application function | `{ type: "function", name, description, parameters }` | Application server |

Add persistent reports or presentations inside a named agent conversation with the server helper:

```ts
import { artifactTool } from "@openuidev/thesys-server";
import type { Tool } from "openai/resources/responses/responses";

const response = await embedClient.responses.create({
  model,
  conversation: authorizedConversationId,
  input: latestTurn,
  instructions: generateSystemPrompt({ cloud: true }),
  tools: [
    artifactTool({ artifacts: ["slides", "report"] }) as unknown as Tool,
    { type: "web_search" },
    { type: "image_search" } as unknown as Tool,
    {
      type: "mcp",
      server_label: "deepwiki",
      server_url: "https://mcp.deepwiki.com/mcp",
    } as unknown as Tool,
  ],
  stream: true,
  store: true,
});
```

Keep compatibility casts scoped to Cloud extensions missing from the installed OpenAI SDK tool union. Do not weaken unrelated types.

This artifact example uses the named-conversation state model so follow-up turns and browser storage can reopen the artifact. Apply the identity and authorization contract in [conversations.md](conversations.md); hosted search, MCP, and application function tools can also be used with the other Responses history patterns.

Remote MCP servers must be declared on each relevant request. Load authenticated MCP headers only from approved server-side secret storage and send them only to an explicitly approved origin. Inspect `mcp_list_tools.error` before concluding the model chose not to use a server.

Managed artifacts are separate stored objects. Register the installed `presentationArtifactRenderer` and `reportArtifactRenderer` client exports, and add Cloud storage when the product must persist and reopen them. Follow-up turns in the same stored conversation can edit them. For standalone generation or explicit program-based editing outside an agent stream, use Artifact Chat Completions instead.

## App-Owned Function Tools

The model emits a `function_call`; the application executes the authorized function and continues with a `function_call_output` until the model returns a final response.

Use the current Cloud template's `src/lib/tool-loop.ts` as the reference implementation. `runFunctionToolLoop` is not a published package export. When porting it, preserve these safeguards:

1. Execute only tool names explicitly declared and registered by the application. Do not execute Cloud-owned calls such as names beginning with `thesys_`.
2. Skip a call when the same stream already contains its `function_call_output`; that call is already settled.
3. Bound the number of continuation iterations and validate every tool argument before execution.

Do not reuse a Chat Completions assistant/tool-message loop; Responses uses `function_call` and `function_call_output` items.

## Reliability and Observability

Managed UI generation validates and repairs output against the selected component contract. Preserve the Cloud stream and matching adapter; do not insert a second blind stream-rewriting layer.

Run representative prompts repeatedly against the actual component library and model choices. Compare parser/renderer failures, partial renders, latency, and cost rather than trusting one successful generation. Use OpenUI DevTools during development.

When production monitoring is required, follow [existing-project.md](existing-project.md#reliability-and-observability) for the current `@openuidev/observability-cloud` boundary.

## Verify

1. Run the shared checks in [existing-project.md](existing-project.md#shared-verification).
2. Confirm the route calls `/v1/embed/responses` and the client uses `openAIResponsesAdapter()`.
3. Confirm exactly one history pattern is active: full `input`, `previous_response_id`, or `conversation`.
4. For full history, verify authorized storage is loaded and bounded on every turn without `conversation` or `previous_response_id`.
5. For a response-id chain, verify ids are stored and authorized and each continuation sets `store: true`.
6. For named conversations, run every identity, token, ownership, CRUD, isolation, and reload check in [conversations.md](conversations.md#verify).
7. Test invalid input-item injection, request limits, missing configuration, Cloud 4xx/5xx, cancellation, and stream closure.
8. Exercise every declared hosted and application-owned tool; confirm the application never executes Cloud-owned calls.
9. If artifacts are enabled, generate, reopen, and edit one supported artifact through the selected state model.
10. Run representative UI prompts repeatedly and inspect settled parser/renderer errors.
11. Run the host formatter, typecheck, tests, and production build.

## First-Party References

- `https://www.openui.com/docs/openui-cloud/api/responses`
- `https://www.openui.com/docs/openui-cloud/api/overview`
- `https://www.openui.com/docs/openui-cloud/api/conversations`
- `https://www.openui.com/docs/openui-cloud/api/artifacts`
- `https://www.openui.com/docs/openui-cloud/build/component-library`
- `https://www.openui.com/docs/agent/reference/adapters-and-formats`
