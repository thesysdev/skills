# Build live dashboards

Use this when the user wants an agent that builds dashboards from a chat request: a layout of charts, tables, metrics, and filters that loads fresh data every time it opens. The agent builds a dashboard once. The user then reopens, filters, and refreshes it without another model call.

Docs: https://www.openui.com/docs/dashboards. The server APIs are in `@openuidev/server` (`OpenUIClient`, `defineDashboard`). The browser APIs are in `@openuidev/react-ui/cloud` (`OpenUICloudProvider`, `LiveArtifactRenderer`, `useArtifactList`). Use `@openuidev/server` 0.4.0 or later and `@openuidev/react-ui` / `@openuidev/react-headless` 0.18.4 or later: earlier `react-ui` versions can't read the tool result, and the card never renders. Check the installed versions before writing code.

## How it fits together

1. **Set up:** the server defines dashboards once with a component library, a model, and data tools.
2. **Generate:** the agent's `create_dashboard` tool calls `dashboards.create()`. It returns a response right away, and the agent returns it as the tool result.
3. **Render:** the chat passes the tool result to `LiveArtifactRenderer`, which shows the dashboard as it's built.
4. **Load data:** each time the dashboard opens, the renderer sends its queries to an interpreter route on the app's server. The route runs the data tools, and generated scripts run in OpenUI's interpreter with secure execution.
5. **Edit:** the agent's `edit_dashboard` tool calls `dashboards.edit()` with the dashboard's id. Only the parts the prompt describes change.

The agent never calls the data tools itself. They belong to the dashboard definition.

## Start a new app

When the user has no app yet, scaffold the dashboards example:

```bash
npx @openuidev/cli@latest create --example assistant-ui-dashboards
```

It builds an assistant-ui + Vercel AI SDK chat app whose agent creates dashboards about npm packages and GitHub repositories, with a saved-dashboards sidebar (open, pin, delete) already in place. It creates a new folder (`--name`, default `openui-agent`) and fails on an existing one. Have the user add the API key as in [new-app.md](new-app.md#3-have-the-user-add-the-api-key), then replace the example's data tools, `preamble`, and agent system prompt with the user's domain.

## Add dashboards to an existing app

Install the packages with the project's package manager:

```bash
npm install @openuidev/server @openuidev/react-ui @openuidev/react-headless @openuidev/react-lang @openuidev/dashboard-library
npm install --save-dev @openuidev/cli
```

Add the agent SDK's packages too: `ai @ai-sdk/openai zod` for the Vercel AI SDK, or `openai`.

### 1. Define dashboards

```ts title="lib/openui.ts"
import { promptOptions } from "@openuidev/dashboard-library";
import { OpenUIClient } from "@openuidev/server";
import spec from "../generated/spec.json";
import { dataTools } from "./data";

export const openui = new OpenUIClient(); // reads THESYS_API_KEY

export const dashboards = openui.defineDashboard({
  library: spec,
  model: "anthropic/claude-sonnet-4.6",
  tools: dataTools,
  promptOptions: {
    preamble: "You build dashboards about the store's orders, customers, and refunds.",
    examples: promptOptions.examples,
    additionalRules: promptOptions.additionalRules,
  },
});
```

- `library`: the library's JSON spec. `tools`, `mcp`, or both are required.
- `model`: `provider/model`. See https://www.openui.com/docs/gateway/models.
- `preamble`: describe the domain. Keep the library's `examples` and `additionalRules` unless the user wants a different style.
- `OpenUIClient` also takes `apiKey`, `baseUrl`, and `appId` (separates apps that share one key).

Generate the spec from a file that re-exports the library alone (re-exporting `promptOptions` puts it in the spec, and the server rejects it), and run it before `dev` and `build`:

```ts title="lib/dashboard-library.ts"
export { library } from "@openuidev/dashboard-library";
```

```json title="package.json"
{
  "scripts": {
    "generate": "openui generate lib/dashboard-library.ts --spec --out generated/spec.json",
    "predev": "npm run generate",
    "prebuild": "npm run generate"
  }
}
```

`predev` runs only through `npm run dev`; run `npm run generate` first when starting `next dev` directly.

Start with `@openuidev/dashboard-library`. To use the app's own design system, build an OpenUI library ([component-library.md](component-library.md)), generate its spec the same way, and pass the same library to the renderer.

### 2. Define the data tools

A data tool is a server function that returns JSON:

```ts title="lib/data.ts"
import type { DataTool } from "@openuidev/server";
import { z } from "zod";

const ordersByDay = z.object({
  days: z.number().int().min(1).max(365).describe("How many days back to include."),
  status: z.enum(["paid", "refunded", "all"]).optional(),
});

// Lenient: "" and null mean unset, and out-of-range days are clamped.
function readOrdersByDay(args: Record<string, unknown>) {
  const days = Math.min(365, Math.max(1, Math.trunc(Number(args.days)) || 30));
  const status = args.status === "paid" || args.status === "refunded" ? args.status : "all";
  return { days, status };
}

type Context = { userId: string; accountId: string };

export const dataTools: Record<string, DataTool<Context>> = {
  get_orders_by_day: {
    description: "Order count and revenue per day, newest last. Revenue is in USD cents.",
    parameters: z.toJSONSchema(ordersByDay),
    output: {
      days: [
        { date: "2026-10-30", orders: 412, revenue: 3_840_500 },
        { date: "2026-10-31", orders: 398, revenue: 3_712_000 },
      ],
    },
    execute: async (args, context, { signal }) => {
      const { days, status } = readOrdersByDay(args);
      return { days: await db.ordersByDay({ days, status, accountId: context.accountId, signal }) };
    },
  },
};
```

- `output` is a static example the model reads to bind components to fields. Match the real shape exactly. Defining dashboards never calls a tool.
- Read arguments leniently: an unpicked filter arrives as `""` or `null`. Treat it as unset, and clamp limits instead of rejecting them. A strict `parse` throws on these and the dashboard shows an error. Keep the lenient reading out of the `parameters` schema: `z.toJSONSchema` can't convert `preprocess` or `transform`.
- Scope data with `context`, which `execute` receives unchanged from the interpreter route.
- Return only what the dashboard shows: results go to the browser as is.
- Honor `signal`, and prefer read-only tools: queries run on every view, and scripts can call any listed tool.
- Type the context with `defineDashboard<Context>()` and `DataTool<Context>`.

To offer an MCP server's tools, pass a connected client as `mcp: { client, outputs }`. Only tools marked `readOnlyHint` are offered unless `include` lists others. `connect: (context) => client` creates a client with the user's credentials. Pass an array for several servers.

### 3. Add the interpreter route

```ts title="app/api/interpreter/route.ts"
import { type InterpreterRequest, OpenUIApiError, ToolExecutionError } from "@openuidev/server";
import { dashboards } from "@/lib/openui";
import { getSession } from "@/lib/auth"; // the app's session lookup

export async function POST(request: Request) {
  const session = await getSession(request);
  if (!session) return new Response("Unauthorized", { status: 401 });

  let body: InterpreterRequest;
  try {
    body = await request.json();
  } catch {
    return Response.json(
      { error: { code: "invalid_input", message: "The request body must be JSON." } },
      { status: 400 },
    );
  }

  try {
    const result = await dashboards.interpreter.execute(
      body,
      { userId: session.user.id },
      { signal: request.signal },
    );
    return Response.json({ result });
  } catch (error) {
    const known = error instanceof ToolExecutionError || error instanceof OpenUIApiError;
    const code = known ? error.code : "internal_error";
    return Response.json(
      { error: { code, message: error instanceof Error ? error.message : String(error) } },
      { status: known ? (error.status ?? (code === "invalid_input" ? 400 : 500)) : 500 },
    );
  }
}
```

Without this route, dashboards render but load no data. `timeoutMs` (default 60000) sets the total time budget.

- The renderer sends `{ name, arguments }`, plus the dashboard's `id` and `version` for a generated script.
- `invalid_input` (no `name`, non-object `arguments`) carries no status, so the route maps it to 400. An unknown tool is `tool_not_found` with 404. An error thrown by a data tool is `internal_error` with 500.
- Scripts run only for the dashboard's owner, but data tools are not tied to a dashboard: any signed-in user can call one by name with any arguments. Scope every data tool by `context`.

### 4. Add the frontend token route and provider

```ts title="app/api/frontend-token/route.ts"
import { OpenUIApiError } from "@openuidev/server";
import { openui } from "@/lib/openui";
import { getSession } from "@/lib/auth";

export async function POST(request: Request) {
  const session = await getSession(request);
  if (!session) return new Response("Unauthorized", { status: 401 });

  try {
    const token = await openui.createFrontendToken({ userId: session.user.id }, { signal: request.signal });
    return Response.json(token, { headers: { "Cache-Control": "private, no-store" } });
  } catch (error) {
    const status = error instanceof OpenUIApiError ? (error.status ?? 500) : 500;
    return Response.json({ error: "Could not create a frontend token." }, { status });
  }
}
```

Return the token object unchanged. Wrap the part of the app that shows dashboards:

```tsx title="app/providers.tsx"
"use client";

import { OpenUICloudProvider } from "@openuidev/react-ui/cloud";

export function Providers({ children }: { children: React.ReactNode }) {
  return <OpenUICloudProvider token="/api/frontend-token">{children}</OpenUICloudProvider>;
}
```

When the token route needs custom headers, such as an `Authorization` header, pass `token` as an async function that fetches the route and returns the token string.

### 5. Give the agent the tools

With the Vercel AI SDK:

```ts title="lib/dashboard-tools.ts"
import { tool } from "ai";
import { z } from "zod";
import { dashboards } from "./openui";

export function dashboardTools(context: { userId: string }) {
  return {
    create_dashboard: tool({
      description:
        "Create a live dashboard. It loads its data every time it opens, so describe which data to show and how, without numbers. Returns as soon as the dashboard has an id.",
      inputSchema: z.object({
        title: z.string().min(1).describe('A short title, such as "Orders, last 30 days".'),
        prompt: z
          .string()
          .min(1)
          .describe("What to build: the sections, metrics, charts, tables, and filters, and which data each shows."),
      }),
      execute: ({ title, prompt }, { abortSignal }) =>
        dashboards.create({ title, prompt }, context, { signal: abortSignal }),
    }),
    edit_dashboard: tool({
      description:
        "Change a dashboard in place, such as adding a chart or a section. Returns as soon as the change has started.",
      inputSchema: z.object({
        id: z.string().min(1).describe("The dashboard's id, art_…, from its tool result."),
        prompt: z.string().min(1).describe("The change to make."),
      }),
      execute: ({ id, prompt }, { abortSignal }) =>
        dashboards.edit({ id, prompt }, context, { signal: abortSignal }),
    }),
  };
}
```

Build the tools per request with the signed-in user: `dashboardTools({ userId: session.user.id })`. With the OpenAI SDK, declare the same two function tools and, for each call, push `{ role: "tool", tool_call_id, content: await dashboards.create(...) }` (or `edit`). Catch a tool's error and push `Error: <message>` as the content instead of failing the turn. Keep the agent's existing model and tools.

A dashboard takes one generation at a time: `edit` rejects with "This dashboard is still generating; edit it once it finishes" while a create or earlier edit is running. The Vercel AI SDK turns a throwing `execute` into an error result for the model on its own.

The tool result must reach the browser. Check the stream format in [artifacts.md](artifacts.md#use-a-stream-that-carries-tool-results): Chat Completions SSE does not carry tool results, so stream with the Vercel AI SDK (`result.toUIMessageStreamResponse()`) for any chat UI below.

Add guidance to the agent's system prompt:

```text
Dashboards load their own data from these tools every time they open:
- get_orders_by_day: order count and revenue per day

- When the user asks for a dashboard, call create_dashboard with a short title and a prompt describing the sections, metrics, charts, tables, and filters, and which data each shows. Don't put numbers in the prompt.
- To change a dashboard, call edit_dashboard with the id from the tool result that created it. "It" means the latest dashboard. Create a new one only when asked.
- The user watches the dashboard being built. Don't repeat the tool result, and don't say it's finished: the tool returns when it starts.
- If a tool result starts with "Error:", or edit_dashboard says the dashboard is still generating, tell the user in one sentence and offer to try again.
- After a tool call, reply in one or two sentences: what the dashboard shows and one change they could ask for.
- You can't see the data, so don't state numbers.
```

List the data tools in the prompt so the agent knows what dashboards can show.

### 6. Render the tool result

With AI SDK `useChat` (`@ai-sdk/react`, `new DefaultChatTransport({ api: "/api/chat" })`), render the message parts:

```tsx
import { library } from "@openuidev/dashboard-library";
import { LiveArtifactRenderer } from "@openuidev/react-ui/cloud";

if (part.type === "tool-create_dashboard" || part.type === "tool-edit_dashboard") {
  if (part.state === "output-available") {
    return (
      <LiveArtifactRenderer
        key={part.toolCallId}
        response={part.output as string}
        library={library}
        interpreter={{ url: "/api/interpreter" }}
      />
    );
  }
}
```

- Default: a card in the chat that opens the dashboard under it.
- Side panel: pass the panel's element as `renderInHtmlElement`.
- In place: `preview={false}`, for a full page or an app-managed panel.
- `id` replaces `response` to open a saved dashboard.
- Slots (`Preview`, `Chrome`, `ChromeTitle`, `Content`, `Loading`, `Error`, …) are passed as children to change or replace each part. `useArtifactRenderer()` inside a slot returns `isStreaming`, `isFetching`, `error`, and `retry`.
- `ThemeProvider` from `@openuidev/dashboard-library` sets `"light"` or `"dark"` mode. See [theming.md](theming.md) for the app's own theme.

Save the tool result with the message. The card renders again when the chat reloads, with the latest version.

### Chat frontends

Each chat UI renders the same tool result with `LiveArtifactRenderer`, inside `OpenUICloudProvider`. Docs: https://www.openui.com/docs/dashboards/add-to-your-agent#chat-frontends.

- **AgentInterface** (`@openuidev/react-ui`): read the Vercel AI SDK route with `fetchLLM({ url: "/api/chat", streamAdapter: vercelAIAdapter(), messageFormat: vercelAIMessageFormat })`. Register a renderer and pass cloud storage:

  ```tsx
  const dashboardRenderer = defineArtifactRenderer({
    type: "dashboard",
    toolName: ["create_dashboard", "edit_dashboard"],
    parser: ({ response }) =>
      typeof response === "string" && response ? { props: { response }, meta: null } : null,
    preview: ({ response }) => <LiveArtifactRenderer response={response} library={library} interpreter={interpreter} />,
    actual: ({ response }) => (
      <LiveArtifactRenderer response={response} library={library} interpreter={interpreter} preview={false} />
    ),
  });

  const storage = useOpenuiCloudStorage({ token: "/api/frontend-token" });
  <AgentInterface llm={llm} storage={storage} artifactRenderers={[dashboardRenderer]} />;
  ```

  Without `storage`, AgentInterface's own storage context hides the provider's, and `useArtifactList` inside it throws "No artifact storage is configured". `openAIAdapter()` never emits tool results, so the renderer never gets a `response`.
- **assistant-ui**: `useChatRuntime({ transport: new AssistantChatTransport({ api: "/api/chat" }) })` on the Vercel AI SDK route. Register the tool UI with `<MessagePrimitive.Parts components={{ tools: { by_name: { create_dashboard: DashboardToolUI, edit_dashboard: DashboardToolUI } } }} />`, where `DashboardToolUI` (`ToolCallMessagePartProps`) renders `LiveArtifactRenderer` with `response={result}` once `result` is a string. The AI SDK `part.type` checks above don't apply to assistant-ui.
- **CopilotKit**: in a catch-all route `app/api/copilotkit/[[...slug]]/route.ts`, build a `BuiltInAgent` per request, with `defineTool` tools from `@copilotkit/runtime/v2` that close over the session user, then `createCopilotRuntimeHandler({ runtime, basePath: "/api/copilotkit" })(request)`. A module-level agent can't see the user. Render with `useRenderTool({ name: "create_dashboard", parameters, render: ({ status, result }) => … })` for each tool, showing `LiveArtifactRenderer` once `status === "complete"`. The runtime needs `ai@6` and `@ai-sdk/openai@3`.

## List saved dashboards

Import from `@openuidev/react-ui/cloud`: the `useArtifactList` in `@openuidev/react-ui` is a different hook, for one chat thread's artifacts. Inside AgentInterface, pass `storage` (see above).

```tsx
import { useArtifactList } from "@openuidev/react-ui/cloud";

const { artifacts, isLoading, error, hasMore, loadMore, refresh } = useArtifactList({
  filter: { type: "dashboard" }, // also pinned, search, limit
});
```

Each item has `id`, `title`, `type`, `updatedAt`, and `pinned`. Open one with `<LiveArtifactRenderer id={id} … />`. A dashboard joins the list once generation finishes; call `refresh` when a renderer's `isStreaming` turns false to show it right away.

Actions have their own hooks, each returning `isPending` and `error` for its last call:

| Hook | Returns |
| --- | --- |
| `useArtifactPin()` | `pin(id)` |
| `useArtifactUnpin()` | `unpin(id)` |
| `useArtifactUpdate()` | `update(id, { title })`, which renames the dashboard |
| `useArtifactDelete()` | `deleteArtifact(id)` |

The actions update every list and renderer right away and revert if the request fails. After pinning or unpinning across separate pinned and unpinned lists, `refresh` both. A deleted dashboard can't be restored: add a confirmation step before `deleteArtifact`.

## Auto-refresh and filters

The model writes both from the prompt. Ask for them in the `create_dashboard` prompt or an edit, for example "refresh the open tickets every minute" or "add a status filter". Filters and refreshes reload data through the interpreter route without a model call.

## Rules

- **Take `userId` from the session**, never from the request body. Use the same id in `create`, `edit`, `interpreter.execute`, and `createFrontendToken`. Only the owner can open, edit, or list a dashboard or run its scripts; data tools run for any signed-in user, so scope them by `context`.
- **Treat the `create` and `edit` response as opaque.** Return it unchanged as the tool result, and pass it unchanged to `LiveArtifactRenderer`. Don't parse it or build it by hand.
- **Use one library on both sides.** The spec passed to `defineDashboard` and the `library` passed to the renderer must come from the same library and version. Regenerate the spec after changing the library.
- **Keep `THESYS_API_KEY` on the server.** The browser uses only the frontend token. The user creates the key (see [SKILL.md](../SKILL.md#never-handle-api-keys)).
- **Keep numbers out of prompts.** Dashboards load data through tools; numbers in a prompt go stale.

## Verify

1. The spec generates, and the app builds.
2. Asking for a dashboard calls `create_dashboard`, and the card appears and fills in.
3. The dashboard shows data, and the interpreter route returns `{ result }` with no errors in the server log. A body without `name` gets 400, not 500.
4. Once the dashboard finishes generating, a follow-up ("add a table of the largest orders") calls `edit_dashboard` with the same id, and the open dashboard updates.
5. After a reload, the chat shows the dashboard again, and it appears in `useArtifactList`.
6. A second user can't open the first user's dashboard.

Without a browser, check the saved dashboards with a frontend token from the token route: `GET https://api.thesys.dev/v1/artifacts?kind=dashboard` with the token in the `x-thesys-frontend-token` header. `Authorization: Bearer` fails with 401, and `type=` is ignored.
