# Customize Agent Interface

Use this for `AgentInterface` from `@openuidev/react-ui`: the complete chat UI with a sidebar, threads, composer, messages, and an artifact workspace. New scaffolds use it, and it is the choice for an existing app with no chat UI. The app supplies the backend (`llm`) and, optionally, the storage.

## Connect it

```tsx
"use client";

import {
  AgentInterface,
  fetchLLM,
  openAIAdapter,
  openAIMessageFormat,
  openuiChatLibrary,
  useOpenuiCloudStorage,
} from "@openuidev/react-ui";

const llm = fetchLLM({
  url: "/api/chat",
  streamAdapter: openAIAdapter(),
  messageFormat: openAIMessageFormat,
});

export function Chat() {
  const storage = useOpenuiCloudStorage({ token: "/api/frontend-token" });
  return (
    <div style={{ height: "100dvh" }}>
      <AgentInterface llm={llm} storage={storage} componentLibrary={openuiChatLibrary} agentName="Assistant" />
    </div>
  );
}
```

- `llm` (required): a `ChatLLM`, usually from `fetchLLM({ url, streamAdapter, messageFormat, headers?, body? })`. `body` adds fields to every request, such as `{ model }`. The request body is `{ threadId, runId, messages, tools, context }`.
- `storage` (optional): where threads live. Without it, threads are in memory and vanish on reload.
- `componentLibrary`: the library used to render assistant messages. It must match the server's spec. Without it, messages render as Markdown.
- Give the interface a height; it fills its container. Import `@openuidev/react-ui/styles/index.css` once.

## Match the stream adapter

The adapter reads what the app's route sends back, which can differ from the API the server calls.

| The route returns | `streamAdapter` | `messageFormat` |
| --- | --- | --- |
| Chat Completions SSE (`data:` lines) | `openAIAdapter()` | `openAIMessageFormat` |
| OpenAI SDK `stream.toReadableStream()` | `openAIReadableStreamAdapter()` | `openAIMessageFormat` |
| Responses SSE | `openAIResponsesAdapter()` | `openAIConversationMessageFormat` |
| Vercel AI SDK UI message stream | `vercelAIAdapter()` | `vercelAIMessageFormat` |
| LangGraph `streamMode: "messages"` SSE | `langGraphAdapter()` | `langGraphMessageFormat` |
| AG-UI events | `agUIAdapter()` | the relay's own format |
| Eve session stream | `eveAdapter()` | see the `vercel-eve` scaffold |

Call adapter factories with `()`. For a custom transport, implement `ChatLLM` directly: `{ streamProtocol: openAIAdapter(), send: ({ threadId, messages, signal }) => fetch(...) }`. Pass `signal` so cancelling works.

## Choose conversation storage

| Threads live in | `storage` |
| --- | --- |
| Memory only | omit it |
| Gateway conversations | `useOpenuiCloudStorage({ token: "/api/frontend-token" })` |
| The app's REST API | `restStorage({ baseUrl: "/api/chat/storage" })`; the app implements the endpoints it calls |
| Anything else | an object implementing `ChatStorage` |

`useOpenuiCloudStorage` options:

- `token`: the URL of the app's frontend-token route, which the hook calls with `POST` and which returns `{ token, expires_at }`; or a function that returns a token.
- `features.artifact` (default `true`): stored-artifact reads and edits.
- `apiBaseUrl` (default `https://api.thesys.dev`).

The hook memoizes the storage and refreshes the token before it expires. It reads and manages threads but does not save messages. The chat route saves each turn; see [gateway.md](gateway.md#store-threads-in-gateway-conversations).

`ChatStorage` is `{ thread, artifact? }`:

- `thread` implements `listThreads`, `createThread`, `getMessages`, `updateThread`, and `deleteThread`.
- `artifact` is covered in [artifacts.md](artifacts.md).

Storage is captured when the interface mounts. Remount it (for example with a `key`) when the signed-in user changes.

## Customize the shell

| Need | Use |
| --- | --- |
| Name and logo | `agentName`, `logoUrl` |
| Suggested prompts | `starters: [{ displayText, prompt, icon? }]`, `starterVariant` |
| Empty-thread screen | `<AgentInterface.Welcome title description />` |
| Sidebar | `<AgentInterface.Sidebar>`, composed from `SidebarHeader`, `SidebarContent`, `NewChatButton`, `ThreadList`, `ArtifactNav`, `SidebarItem`, `SidebarFooter`, `ThemeModeToggle` |
| Header and composer | `AgentInterface.ThreadHeader`, `AgentInterface.MobileHeader`, `AgentInterface.Composer` |
| A model picker | `<ModelSwitcher models value onValueChange />` inside `ThreadHeader` |
| Artifact panel | `AgentInterface.Workspace` |
| Message rendering | `components={{ AssistantMessage, UserMessage, ToolCallTimeline }}`; an override replaces what `componentLibrary` would render |
| Custom actions from the library | `onAction` |
| Scrolling | `scrollVariant` (`"user-message-anchor"` or `"always"`), `scrollOnLoad` |
| Colors and dark mode | `theme`; see [theming.md](theming.md) |

- Place slot components directly under `<AgentInterface>`; unspecified slots keep their defaults.
- When supplying `Sidebar`, put every control the sidebar needs inside it.
- The layout switches to mobile below 768px of container width. Test the real container, not only the window.

## Add pages

- `<AgentInterface.Route path="settings">…</AgentInterface.Route>` adds a page inside the shell. `useNav()` returns the current path and `navigate(next)`; `navigate(undefined)` returns to the chat.
- Without `onNavigate`, navigation is internal, and `defaultPath` sets the first page.
- With `onNavigate`, the host controls navigation and must pass the new value back through `path`.
- Paths under `artifacts/` are reserved for the artifact browser.

## Verify

1. Send a message; it streams, can be cancelled, and shows errors.
2. With storage: create, rename, delete, and reload a thread. A second user sees none of the first user's threads.
3. A starter, a button in the generated UI, and a tool call each work.
4. Check desktop and narrow widths.
5. Run the project's typecheck and build.

## Sources

- [AgentInterface props](https://www.openui.com/docs/agent/reference/agentinterface-props)
- [Adapters and formats](https://www.openui.com/docs/agent/reference/adapters-and-formats)
- [Source](https://github.com/thesysdev/openui/tree/main/packages/react-ui/src/components/AgentInterface)
- [Storage and LLM interfaces](https://github.com/thesysdev/openui/blob/main/packages/react-headless/src/adapters/types.ts)
