# Agent Interface

`AgentInterface` from `@openuidev/react-ui` is an open-source React chat app with a sidebar, thread history, messages, a composer, navigation, and an artifact workspace. It works the same with Gateway or a self-hosted backend: the app supplies generation, tools, and durable storage through two independent props.

## Choose the UI Surface

| Need | Use |
| --- | --- |
| A complete chat app with threads and an artifact workspace | `AgentInterface` |
| Generated UI inside an existing assistant-ui, CopilotKit, or custom chat | Keep that shell and render OpenUI Lang in its assistant-message slot ([existing-chat guides](examples.md#existing-chat-ui-integration-guides)) |
| An app-owned React chat layout that needs chat state and adapters | `@openuidev/react-headless` |

Agent Interface measures its container and switches to the mobile layout below 768px. Give it a real height in the host layout, and test it at the container's actual width when embedding it in a side rail.

## Connect a Backend

| Prop | Required | Role |
| --- | --- | --- |
| `llm` | Yes | A `ChatLLM`, usually from `fetchLLM()` pointed at the app's route |
| `storage` | No | A `ChatStorage` for durable threads; without it, threads live in memory until reload |
| `componentLibrary` | No | Renders assistant OpenUI Lang inline; without it, messages render as Markdown |
| `artifactRenderers` | No | Renders tool results as [artifacts](artifacts.md) |

`componentLibrary` configures the browser only. The server's prompt must come from the same library ([build-component-library.md](build-component-library.md)). Styles and the Next.js client boundary are covered in [Set Up React UI](../SKILL.md#set-up-react-ui).

**Gateway**, as in the Gateway template (Responses with Gateway Conversations):

```tsx
"use client";

import {
  AgentInterface,
  fetchLLM,
  openAIConversationMessageFormat,
  openAIResponsesAdapter,
  openuiLibrary,
  useOpenuiCloudStorage,
} from "@openuidev/react-ui";

const llm = fetchLLM({
  url: "/api/chat",
  streamAdapter: openAIResponsesAdapter(),
  messageFormat: openAIConversationMessageFormat,
});

export function Agent() {
  const storage = useOpenuiCloudStorage({ token: "/api/frontend-token", features: { artifact: false } });

  return (
    <div style={{ height: "100dvh" }}>
      <AgentInterface llm={llm} storage={storage} componentLibrary={openuiLibrary} agentName="Assistant" />
    </div>
  );
}
```

The server side is [responses.md](gateway/responses.md) plus the token route in [conversations.md](gateway/conversations.md#mint-frontend-tokens).

**Self-hosted**, as in the self-hosted template (raw Chat Completions SSE), with optional app-owned storage:

```tsx
"use client";

import { AgentInterface, fetchLLM, openAIAdapter, openAIMessageFormat, restStorage } from "@openuidev/react-ui";
import { openuiLibrary } from "@openuidev/react-ui/genui-lib";

const llm = fetchLLM({ url: "/api/chat", streamAdapter: openAIAdapter(), messageFormat: openAIMessageFormat });
const storage = restStorage({ baseUrl: "/api/chat/storage" }); // optional; the app implements these routes

export function Agent() {
  return (
    <div style={{ height: "100dvh" }}>
      <AgentInterface llm={llm} storage={storage} componentLibrary={openuiLibrary} agentName="Assistant" />
    </div>
  );
}
```

The server side is [self-hosted.md](self-hosted.md#generate-on-the-server).

## Match the Browser Stream

Choose the adapter from the bytes the route returns to the browser. A framework can call Responses or Chat Completions on the server and still send its own event format to the client.

| Browser receives | `streamAdapter` | `messageFormat` |
| --- | --- | --- |
| Responses SSE | `openAIResponsesAdapter()` | `openAIConversationMessageFormat` with Gateway Conversations; otherwise match the route's history model |
| Raw Chat Completions SSE | `openAIAdapter()` | `openAIMessageFormat` |
| OpenAI SDK `toReadableStream()` | `openAIReadableStreamAdapter()` | `openAIMessageFormat` |
| Vercel AI SDK UIMessage stream | `vercelAIAdapter()` | `vercelAIMessageFormat` |
| Native LangGraph SSE | `langGraphAdapter()` | `langGraphMessageFormat` |
| AG-UI events (LangGraph Agent Server relay, or a translated provider stream) | `agUIAdapter()` | The route's request contract |
| Eve session NDJSON | `eveAdapter()` with the framework's session transport | Keep session ids, continuation tokens, and stream cursors; see the [Eve example](examples.md#agent-frameworks) |

Call adapter factories with `()`. `fetchLLM({ url, streamAdapter, messageFormat, body })` posts `threadId`, `runId`, formatted `messages`, `tools`, `context`, and any extra `body` fields, and forwards cancellation. Check which fields the route reads before changing them. Provider keys stay on the server.

For a custom transport, implement `ChatLLM` directly. Its property is `streamProtocol` (the `fetchLLM` option is `streamAdapter`), and it owns message conversion and abort forwarding:

```ts
import { type ChatLLM, openAIAdapter, openAIMessageFormat } from "@openuidev/react-ui";

const llm: ChatLLM = {
  streamProtocol: openAIAdapter(),
  send: ({ threadId, messages, signal }) =>
    fetch("/api/chat", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ threadId, messages: openAIMessageFormat.toApi(messages) }),
      signal,
    }),
};
```

The UI renders text deltas, tool calls, tool results, errors, and completion events from the adapter. Tools run in the backend or agent runtime; configuring the UI does not add a tool loop.

## Choose Conversation Storage

| Owner | `storage` |
| --- | --- |
| None (in memory) | Omit `storage` |
| Gateway Conversations | `useOpenuiCloudStorage({ token: "/api/frontend-token", features: { artifact: false } })` ([conversations.md](gateway/conversations.md)) |
| App backend implementing the REST adapter's endpoints | `restStorage({ baseUrl })` |
| Existing database or framework API | A custom `ChatStorage` |

`useOpenuiCloudStorage` takes a token-route URL (or a function that returns a token), mints tokens lazily, refreshes and retries them, and sends them on every storage request. It stores threads; artifacts use the app's own adapter ([artifact storage](artifacts.md#handle-streaming-and-storage)).

`ChatStorage.thread` provides `listThreads`, `createThread`, `getMessages`, `updateThread`, and `deleteThread`. `restStorage` calls fixed endpoints under `baseUrl`, and the app must implement those routes with the matching message format; check the installed adapter's contract before adopting it.

A storage adapter does not persist replies by itself. The generation route or framework must write the completed user, assistant, and tool messages that `getMessages` later returns, keeping message ids and tool call and result pairs intact.

Agent Interface captures `storage` when it mounts. Remount it (for example, with a `key` tied to the user) when the signed-in user or the storage configuration changes, and keep adapters stable across ordinary renders.

## Render Messages

- Pass `openuiLibrary`, `openuiChatLibrary`, or a custom library as `componentLibrary`. The same library must produce the server's prompt.
- `components.AssistantMessage` and `components.UserMessage` replace message rendering entirely and take precedence over `componentLibrary`. A replacement must keep streaming state, tool activity, artifact previews, and UI action handling.
- Inline generated UI (`componentLibrary`) and artifacts (`artifactRenderers`) are independent and can be used together.

## Customize the Shell

Start with props, then replace only the slots the product needs:

| Need | Surface |
| --- | --- |
| Name and logo | `agentName`, `logoUrl` |
| Suggested prompts | `starters` (`displayText`, `prompt`, optional `icon`) with `starterVariant` `short` or `long` |
| Empty-thread greeting | `AgentInterface.Welcome` with `title`, `description`, and optional starters, or custom children |
| Model picker | `ModelSwitcher` inside `ThreadHeader` and `MobileHeader`; send the choice in `fetchLLM({ body: { model } })` and validate it against the server's allowlist |
| Sidebar | `AgentInterface.Sidebar` composed from `SidebarHeader`, `SidebarContent`, `NewChatButton`, `ThreadList`, `ArtifactNav`, and `SidebarItem` |
| Thread controls and input | `AgentInterface.ThreadHeader`, `AgentInterface.Composer`, `AgentInterface.MobileHeader` |
| Artifact workspace | `AgentInterface.Workspace` |
| Colors, type, radii, light and dark | `theme`, or `disableThemeProvider` when the host owns the provider ([theme-provider.md](theme-provider.md)) |

Place slot elements as direct children of `AgentInterface`. Unspecified slots use their defaults. A custom `Sidebar` replaces the whole sidebar, so include every control it needs inside it; a separate top-level `SidebarHeader` is then ignored. Scope CSS overrides to a host wrapper around `.openui-agent-*` and check both desktop and mobile layouts.

## Connect Navigation

`AgentInterface.Route` adds app pages inside the shell; the path `undefined` is the thread view.

- **Internal:** omit `onNavigate`; `defaultPath` sets the first page.
- **Controlled:** pass both `path` and `onNavigate`, and feed each change back into `path`. Passing `onNavigate` alone makes navigation controlled, and the interface cannot change pages until `path` updates.

Descendants call `useNav()` to read the path and `navigate(next)`, including `navigate(undefined)` to return to chat. Paths starting with `artifacts/` are reserved and match before app routes; a URL router must round-trip them along with custom pages.

## Verify

1. Send a message through the real backend and confirm progressive rendering, completion, cancellation, and visible errors.
2. Exercise a starter, a UI action, and each configured tool; tool results and artifact previews survive any custom message renderer.
3. With storage, create, select, rename, delete, and reload a thread; history and tool pairs are intact, and another user cannot see it.
4. Check the welcome screen, composer, sidebar, scrolling, and workspace at desktop width and in the real container.
5. With controlled routing, test custom pages, the return to chat, artifact routes, and browser back and forward.
6. Run [artifact](artifacts.md#verify) and [theme](theme-provider.md#verify-the-result) checks when those features are used, then the host typecheck and production build.

## First-Party References

- `https://www.openui.com/docs/agent/reference/agentinterface-props`
- `https://www.openui.com/docs/agent/reference/components`
- `https://www.openui.com/docs/agent/reference/adapters-and-formats`
- `https://www.openui.com/docs/agent/reference/self-hosting`
- `https://github.com/thesysdev/openui/tree/main/packages/react-ui/src/components/AgentInterface`
