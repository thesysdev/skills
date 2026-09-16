# Agent Interface

Use this guide when adding, configuring, customizing, or debugging `AgentInterface` from `@openuidev/react-ui`. It provides a React chat application with a sidebar, thread history, messages, composer, navigation, and an artifact workspace. The application supplies generation, tools, and any durable storage.

For a new app, start with the [Gateway quickstart](gateway/quickstart.md), then use this guide to configure the generated interface. For an existing app, preserve its backend, authentication, transport, and storage. Agent Interface works with Gateway or an application-owned backend.

## Contents

- [Choose the UI surface](#choose-the-ui-surface)
- [Connect the interface](#connect-the-interface)
- [Match the browser stream](#match-the-browser-stream)
- [Choose conversation storage](#choose-conversation-storage)
- [Render messages and generated UI](#render-messages-and-generated-ui)
- [Customize the shell](#customize-the-shell)
- [Connect navigation](#connect-navigation)
- [Add artifacts](#add-artifacts)
- [Verify](#verify)

## Choose the UI Surface

| Application need | Integration |
| --- | --- |
| A complete chat application with threads and an artifact workspace | `AgentInterface` from `@openuidev/react-ui` |
| An existing assistant-ui, CopilotKit, or custom chat surface that needs generated UI | Keep its shell and render OpenUI Lang in the assistant-message slot; follow [existing-chat integrations](examples.md#existing-chat-ui-integration-guides) |
| An application-owned React chat layout that needs chat state and adapters | Use `@openuidev/react-headless` directly |

Agent Interface measures its container and uses the mobile layout below 768px. A narrow container still has chat-shell navigation and controls. For a compact assistant rail, choose the surface based on which parts of the UI the host already owns; when using Agent Interface, customize its slots and test the actual container width.

## Connect the Interface

Configure generation and persistence independently:

- `llm` is required. Supply a `ChatLLM`, usually created by `fetchLLM()` for an application HTTP route.
- `storage` is optional. Supply a `ChatStorage` for durable threads; without it, conversations live in memory and are cleared on reload.
- `componentLibrary` enables inline OpenUI Lang rendering. Omit it for the default Markdown message rendering.
- `artifactRenderers` connects application tool results to artifact previews and full views.

This example uses Gateway Responses and Gateway conversation storage. Implement the server generation route from [Responses](gateway/chat/responses.md) and the frontend-token route from [Conversations](gateway/chat/conversations.md). Keep the generated framework's adapter when its browser stream differs.

```tsx
"use client";

import {
  AgentInterface,
  fetchLLM,
  openAIConversationMessageFormat,
  openAIResponsesAdapter,
  openuiChatLibrary,
  useOpenuiCloudStorage,
} from "@openuidev/react-ui";
import "@openuidev/react-ui/components.css";
import "@openuidev/react-ui/styles/index.css";

const llm = fetchLLM({
  url: "/api/chat",
  streamAdapter: openAIResponsesAdapter(),
  messageFormat: openAIConversationMessageFormat,
});

export function Agent() {
  const storage = useOpenuiCloudStorage({ token: "/api/frontend-token" });

  return (
    <div style={{ height: "100dvh" }}>
      <AgentInterface
        llm={llm}
        storage={storage}
        componentLibrary={openuiChatLibrary}
        agentName="Research assistant"
      >
        <AgentInterface.Welcome
          title="What would you like to explore?"
          description="Ask a question or create something to work with."
        />
      </AgentInterface>
    </div>
  );
}
```

The server must use the matching serialized library spec with `generateSystemPrompt({ cloud: true, library })`; follow [component-library handoff](build-component-library.md). `componentLibrary` configures rendering in the browser; it does not configure the model's prompt.

Import the styles once at the location allowed by the host framework. For Tailwind layers and provider ownership, follow [the theme guide](theme-provider.md). In Next.js, keep the interactive interface in a client module and retain server authentication in the surrounding page/layout. Give the interface a usable height within the host layout; the example's viewport height is appropriate for a full-page app.

## Match the Browser Stream

Select the adapter from the bytes returned to the browser. A framework may call a provider's Responses or Chat Completions API internally while sending its own event format to the client.

| Browser response | `streamAdapter` | Outgoing message format |
| --- | --- | --- |
| Responses SSE | `openAIResponsesAdapter()` | `openAIConversationMessageFormat` for the Gateway conversation route above; preserve the route's selected history model |
| Raw Chat Completions SSE | `openAIAdapter()` | `openAIMessageFormat` |
| OpenAI SDK Chat Completions `toReadableStream()` | `openAIReadableStreamAdapter()` | `openAIMessageFormat` |
| Framework stream, such as Vercel AI SDK, LangGraph, AG-UI, or Eve | The framework's adapter | Match its request contract; inspect [the scaffold contracts](gateway/quickstart.md#work-from-the-generated-app) or [framework examples](examples.md) |

`fetchLLM({ url, streamAdapter, messageFormat })` posts to the application's route, converts outgoing messages, and forwards cancellation. Its body includes `threadId`, `runId`, `messages`, `tools`, and `context`; inspect the host route before changing which fields it consumes. Provider keys belong on the server.

For custom transport, implement `ChatLLM` directly. Its property is `streamProtocol`, while the `fetchLLM` option is `streamAdapter`. A direct implementation owns message conversion and abort forwarding. For example, an app-owned raw Chat Completions route can use:

```ts
import {
  type ChatLLM,
  openAIAdapter,
  openAIMessageFormat,
} from "@openuidev/react-ui";

const llm: ChatLLM = {
  streamProtocol: openAIAdapter(),
  send: ({ threadId, messages, signal }) =>
    fetch("/api/chat", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        threadId,
        messages: openAIMessageFormat.toApi(messages),
      }),
      signal,
    }),
};
```

Call adapter factories with `()`. Verify text deltas, tool calls, tool results, errors, and completion events through the selected adapter. The application's backend or agent runtime executes its tools; configuring the UI does not implement a tool loop.

## Choose Conversation Storage

| Persistence owner | `storage` configuration |
| --- | --- |
| In-memory conversations | Omit `storage` |
| Gateway Conversations | `useOpenuiCloudStorage({ token: "/api/frontend-token" })` from `@openuidev/react-ui`; follow [Conversations](gateway/chat/conversations.md) |
| Application backend following the REST adapter's endpoints | `restStorage({ baseUrl: "/api/chat/storage" })` from `@openuidev/react-ui` |
| Existing database or framework with another API | Implement `ChatStorage` around that API |

`ChatStorage.thread` supplies `listThreads`, `createThread`, `getMessages`, `updateThread`, and `deleteThread`. Match the installed `restStorage` endpoint and message-format contract before adopting it; setting a base URL does not create those server routes.

Trace who persists completed user, assistant, and tool messages. The generation route or framework must write the history that `getMessages` later returns. Adding a storage prop alone does not make a generation endpoint persist replies. Preserve message ids and tool call/result pairing across reloads.

Agent Interface captures storage when its provider mounts. Remount the interface when the authenticated identity or storage configuration changes so a session cannot retain another user's store. Keep adapters stable during ordinary renders.

Artifact persistence uses the separate optional `ChatStorage.artifact` interface. `restStorage` supplies thread storage only. Configure artifact storage when users must browse or reopen saved artifacts; follow [the artifact storage workflow](artifacts.md#handle-streaming-and-storage).

## Render Messages and Generated UI

- Use `openuiChatLibrary` for OpenUI's chat components, or pass the application's custom library through `componentLibrary`.
- Keep the renderer's component definitions synchronized with the server's prompt/spec. Regenerate the serialized spec when definitions change; see [build-component-library.md](build-component-library.md).
- Use `components.AssistantMessage` or `components.UserMessage` for a complete message-renderer override. An explicit override takes precedence over the corresponding renderer derived from `componentLibrary`.
- When replacing message rendering, retain the required streaming state, tool activity, artifact previews, and UI action handling. Check the existing renderer before replacing it with a text-only component.

Inline generated UI and artifacts are separate configuration choices: `componentLibrary` renders assistant OpenUI Lang, while `artifactRenderers` renders application tool content. They can be used together.

## Customize the Shell

Start with props for branding and configuration, then replace the specific slots needed by the product:

| Need | Public surface |
| --- | --- |
| Agent name and logo | `agentName`, `logoUrl` |
| Suggested prompts | `starters`: entries with `displayText`, `prompt`, and optional `icon`; `starterVariant` selects `short` or `long` |
| Empty-thread greeting | `AgentInterface.Welcome` with `title`, `description`, and optional starters, or custom children |
| Sidebar content | `AgentInterface.Sidebar`, composed with `SidebarHeader`, `SidebarContent`, `NewChatButton`, `ThreadList`, `ArtifactNav`, and `SidebarItem` as needed |
| Thread controls and input | `AgentInterface.ThreadHeader`, `AgentInterface.Composer`, and `AgentInterface.MobileHeader` |
| Artifact workspace | `AgentInterface.Workspace` |
| Colors, typography, radii, light/dark mode | `theme`; use `disableThemeProvider` when the host owns the surrounding provider |

Place slot elements directly under `AgentInterface` so its slot extraction recognizes them. Unspecified sidebar, header, composer, and workspace slots use their defaults. When replacing the whole sidebar, include its needed controls inside that slot; a separate top-level `SidebarHeader` is ignored when `Sidebar` is supplied.

Use [the theme guide](theme-provider.md) for tokens, provider ownership, and CSS layers. Scope CSS overrides to a host wrapper around `.openui-agent-*`, and inspect both the desktop and mobile layouts. Avoid relying on viewport width alone when the interface is embedded in a smaller container.

## Connect Navigation

Use `AgentInterface.Route` for application pages inside the shell. `undefined` selects the thread view. Without `onNavigate`, navigation is internal and `defaultPath` can set the initial page.

For host-controlled navigation, provide both `path` and `onNavigate`, and update the host state or router in the callback. The presence of `onNavigate` selects controlled mode even when `path` is `undefined`. Pass navigation changes back into `path`; otherwise the interface cannot change pages.

Descendants can use `useNav()` from `@openuidev/react-ui` to read the current path and call `navigate(next)`, including `navigate(undefined)` to return to chat. Preserve the reserved `artifacts/` paths, which are matched before application routes. When synchronizing with a URL router, round-trip artifact paths as well as custom pages.

## Add Artifacts

Agent Interface has one artifact type. An application tool supplies the content, and an application-provided renderer can show HTML, Markdown, a presentation, or any other requested output.

Register stable `artifactRenderers` built with `defineArtifactRenderer`. Each renderer connects the tool payload to an inline preview and a full view; the interface provides the workspace and, when storage is configured, artifact browsing. `artifactCategories` optionally organizes saved content using application-chosen renderer identifiers.

Follow [Agent Interface Artifacts](artifacts.md) for tool-result delivery, parsing partial data, custom views, edits, and durable storage. Reuse the same lifecycle for HTML, Markdown, and other payloads.

## Verify

1. Send a message through the actual backend; confirm progressive text or UI rendering, completion, cancellation, and visible error handling.
2. Exercise a starter, UI action, and application tool when configured. Confirm tool results and artifact previews survive any custom message rendering.
3. With persistence enabled, create, select, rename, delete, and reload a thread. Verify full history and tool pairing, plus isolation across authenticated users.
4. Check welcome content, composer, sidebar, scrolling, and workspace in the real desktop and narrow containers.
5. For controlled routing, verify custom pages, return-to-thread navigation, and saved-artifact routes; test browser back/forward when synchronized with a URL router.
6. When artifacts are requested, run [artifact verification](artifacts.md#verify). When theming is changed, run [theme verification](theme-provider.md#verify-the-result).
7. Run the host typecheck and production build, including the framework's client boundary and CSS setup.

## First-Party References

- [Agent Interface props](https://www.openui.com/docs/agent/reference/agentinterface-props)
- [Components and slots](https://www.openui.com/docs/agent/reference/components)
- [Adapters and formats](https://www.openui.com/docs/agent/reference/adapters-and-formats)
- [Agent Interface source](https://github.com/thesysdev/openui/tree/main/packages/react-ui/src/components/AgentInterface)
- [Chat storage and generation interfaces](https://github.com/thesysdev/openui/blob/main/packages/react-headless/src/adapters/types.ts)
