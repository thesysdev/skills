# Generic Agent Interface Artifacts

Use this guide when an agent produces content the user should preview, open, revisit, or edit. Agent Interface has one generic artifact abstraction: the application chooses the data and supplies its renderer. It can represent HTML, Markdown, a presentation, a dashboard, or any other user-requested output. Reports and presentations are no longer built-in managed products.

Artifacts are independent of Gateway. Preserve the application's model provider, tool loop, stream adapter, and storage choices. With Gateway, use ordinary application function tools through [Responses](gateway/chat/responses.md#app-owned-function-tools) or [Chat Completions](gateway/chat/chat-completions.md#keep-function-tools-in-the-application).

## Connect a Tool to a Renderer

1. Declare an application tool that creates or updates the requested content. Define and validate its arguments and result; execute it through the application's existing tool loop.
2. Deliver the tool call and its paired result to Agent Interface through the selected stream adapter. A result supplied only to the next model request is not automatically visible to the UI; verify the transport emits the corresponding tool-result event.
3. Use `defineArtifactRenderer` from `@openuidev/react-ui` to parse the tool envelope, show an inline `preview`, and supply the full `actual` view.
4. Register a stable renderer array with `AgentInterface.artifactRenderers`.

The renderer's `toolName` matches incoming tool calls. Its `type` is an application-chosen identifier for stored-artifact lookup; it is not a built-in list of supported content formats. Choose the payload and renderer around the user's content instead of imposing report/slide schemas.

## Supply Your Own View

This example uses one generic renderer with an application-defined payload. `create_artifact`, `update_artifact`, `artifact`, and the payload fields are application conventions, not SDK tool names or a required wire schema. The host provides `renderContent`, which can dispatch to its own HTML, Markdown, presentation, or other view.

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

For persistence, implement the optional `ChatStorage.artifact` interface (`list`, `get`, `update`) alongside thread storage. The creation tool owns the initial durable write; the storage interface has no `create` method. Store the complete tool result as `artifact.content`, with the same application-chosen `type` as the renderer and the owning `threadId`. Keep tool results in persisted message history when they must reappear inline after a thread reload.

Thread persistence, including Gateway Conversations, does not automatically implement artifact storage. Configure and verify each requested lifecycle separately. For an edit, load and authorize the stored artifact, apply the requested change, persist the new version, and emit the updated tool result for the same id.

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
