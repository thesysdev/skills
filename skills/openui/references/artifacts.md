# Agent Interface Artifacts

Use this guide when an agent produces content the user should preview, open, revisit, or edit. An artifact is the result of an application tool, shown in Agent Interface's workspace by a renderer the app supplies. The app chooses the data and the view, so an artifact can be HTML, Markdown, a presentation, a dashboard, or any other output.

For the chat shell, backend connections, and navigation, read [Agent Interface](agent-interface.md). Artifacts work the same on any backend: they use the app's tool loop ([self-hosted](self-hosted.md#run-function-tools), [Gateway Responses](gateway/responses.md#run-app-function-tools), or [Gateway Chat Completions](gateway/chat-completions.md#run-function-tools)), its stream adapter, and its storage.

## Connect a Tool to a Renderer

1. Declare an app tool that creates or updates the content. Validate its arguments and result, and run it through the app's tool loop.
2. Deliver the tool call and its result to Agent Interface through the stream. A result sent only to the next model request never reaches the UI, so the executor must also emit a tool-result event through the browser stream. This applies to Chat Completions tool results and to Responses `function_call_output` items alike.
3. Use `defineArtifactRenderer` from `@openuidev/react-ui` to parse the tool result, show an inline `preview`, and supply the full `actual` view.
4. Pass a stable renderer array to `AgentInterface` as `artifactRenderers`.

The renderer's `toolName` matches incoming tool calls. Its `type` is an app-chosen identifier used to look up stored artifacts; it does not limit which content formats are supported.

## Supply Your Own View

This example uses one renderer for every artifact. The tool names (`create_artifact`, `update_artifact`), the `type` value, and the payload fields are examples; choose names and a payload that fit the app. The host's `renderContent` function dispatches to its own HTML, Markdown, presentation, or other view.

```tsx
import type { ReactNode } from "react";
import { defineArtifactRenderer } from "@openuidev/react-ui";

export type ArtifactData = {
  id: string;
  title: string;
  version: number;
  mediaType: string;
  content: unknown;
};

function parseArtifact(raw: unknown): ArtifactData | null {
  try {
    const value: unknown = typeof raw === "string" ? JSON.parse(raw) : raw;
    if (!value || typeof value !== "object") return null;
    const data = value as Record<string, unknown>;
    if (
      typeof data.id !== "string" ||
      typeof data.title !== "string" ||
      typeof data.version !== "number" ||
      !Number.isInteger(data.version) || data.version < 1 ||
      typeof data.mediaType !== "string" ||
      !("content" in data)
    ) return null;
    return data as ArtifactData;
  } catch {
    return null;
  }
}

export function createAppArtifactRenderer(
  renderContent: (artifact: ArtifactData) => ReactNode,
) {
  return defineArtifactRenderer({
    type: "artifact",
    toolName: ["create_artifact", "update_artifact"],
    parser: ({ args, response }, { isStreaming }) => {
      const data = parseArtifact(response ?? (isStreaming ? args : null));
      if (!data) return null;
      return {
        props: data,
        meta: isStreaming
          ? null
          : { id: data.id, version: data.version, heading: data.title },
      };
    },
    preview: (data, { open, isStreaming }) => (
      <button type="button" onClick={open}>
        {data.title}{isStreaming ? " · building…" : ""}
      </button>
    ),
    actual: (data) => renderContent(data),
  });
}
```

Create the renderer and array outside the React render cycle, then pass them to the interface:

```tsx
import { AgentInterface } from "@openuidev/react-ui";

const artifactRenderers = [createAppArtifactRenderer(renderArtifactContent)];

<AgentInterface llm={llm} artifactRenderers={artifactRenderers} />;
```

Here `renderArtifactContent` is the host's React rendering function. Validate each content shape before using it. For generated HTML, use a sandboxed iframe and the restrictions in [open-ended-html.md](open-ended-html.md); for Markdown or presentations, use the application's chosen renderer. No OpenUI Lang root is required for arbitrary tool-result content. If the content itself is OpenUI Lang, use `Renderer` with its matching library.

## Handle Streaming and Storage

The parser is called during streaming and when a stored artifact opens:

| Path | Inputs | Parser behavior |
| --- | --- | --- |
| Tool call | `args` may be partial JSON; `response` may be absent | Tolerate incomplete input and return `null` until the view has enough data |
| Completed tool result | `response` contains the result | Validate it and use it as the authoritative content |
| Stored artifact | `args` is `undefined`; `response` is `artifact.content` | Parse the same result shape without relying on tool arguments |

Keep a stable artifact id and increment its version after edits. Return `meta: null` for an early preview and `{ id, version, heading }` when registering a completed artifact in the thread. Registry metadata alone does not make the artifact durable.

For persistence, implement the optional `ChatStorage.artifact` interface (`list`, `get`, `update`) against the application's backend, or reuse its existing compatible adapter. The creation tool owns the initial durable write; the storage interface has no `create` method. Store the complete tool result as `artifact.content`, with the same application-chosen `type` as the renderer and the owning `threadId`. Keep tool results in persisted message history when they must reappear inline after a thread reload.

Combine thread storage and the app's artifact adapter in one stable `ChatStorage` object: `{ thread: threadStorage.thread, artifact: appArtifactStorage }`. With Gateway threads, create the thread storage with `useOpenuiCloudStorage({ token, features: { artifact: false } })`, as the Gateway template does, then add the app's artifact adapter. Configure and verify thread and artifact persistence separately. For an edit, load and authorize the stored artifact, apply the change, persist the new version, and emit the updated tool result with the same id.

## Verify

- Exercise creation through the actual tool loop and confirm both the inline preview and full custom view render.
- Test partial arguments, a missing result, malformed data, cancellation, and a failed tool without parser exceptions or false completion.
- If persistence is configured, reopen the artifact from storage with no tool arguments and confirm the same content renders.
- Exercise an edit and confirm the stable id, new version, and saved content agree; reject cross-user reads and edits.
- Run the host typecheck/build and inspect the rendered output in its real container.

## First-Party References

- [Artifacts](https://www.openui.com/docs/agent/core-concepts/artifacts)
- [Custom artifacts](https://www.openui.com/docs/agent/guides/custom-artifacts)
- [defineArtifactRenderer](https://www.openui.com/docs/agent/reference/define-artifact-renderer)
- [Storage and stream adapters](https://www.openui.com/docs/agent/reference/adapters-and-formats)
