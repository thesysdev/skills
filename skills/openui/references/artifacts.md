# Show tool output as artifacts

Use this when an agent produces something the user should preview, open, revisit, or edit in Agent Interface: a document, an HTML app, a presentation, a dashboard. An artifact is a tool result plus a renderer the app supplies. Agent Interface provides the inline preview slot, the side panel, and, with storage, an artifact browser. Agent Interface itself is covered in [agent-interface.md](agent-interface.md).

## How it fits together

1. The app declares a tool (for example `create_artifact`) and runs it in its tool loop. The tool name and payload shape are the app's choice.
2. The tool call and its result reach the browser through the stream. Check that the route's stream carries tool results, not just tool calls; a result fed only to the next model request never reaches the UI.
3. A renderer from `defineArtifactRenderer` turns the tool payload into an inline `preview` and a full `actual` view.
4. The renderers go to `<AgentInterface artifactRenderers={...} />`.

## Write the renderer

```tsx
import type { ReactNode } from "react";
import { defineArtifactRenderer } from "@openuidev/react-ui";

export type ArtifactData = { id: string; title: string; version: number; content: unknown };

function parseArtifact(raw: unknown): ArtifactData | null {
  try {
    const value = typeof raw === "string" ? JSON.parse(raw) : raw;
    if (!value || typeof value !== "object") return null;
    const data = value as Record<string, unknown>;
    if (typeof data.id !== "string" || typeof data.title !== "string") return null;
    if (typeof data.version !== "number" || !("content" in data)) return null;
    return data as ArtifactData;
  } catch {
    return null; // partial JSON while the tool call streams
  }
}

export function createArtifactRenderer(renderContent: (artifact: ArtifactData) => ReactNode) {
  return defineArtifactRenderer({
    type: "document",
    toolName: ["create_artifact", "update_artifact"],
    parser: ({ args, response }, { isStreaming }) => {
      const data = parseArtifact(response ?? (isStreaming ? args : null));
      if (!data) return null;
      return {
        props: data,
        meta: isStreaming ? null : { id: data.id, version: data.version, heading: data.title },
      };
    },
    preview: (data, { open, isStreaming }) => (
      <button type="button" onClick={open}>
        {data.title}
        {isStreaming ? " · building…" : ""}
      </button>
    ),
    actual: (data) => renderContent(data),
  });
}
```

Create the renderer array once, outside React render:

```tsx
const artifactRenderers = [createArtifactRenderer(renderDocument)];

<AgentInterface llm={llm} artifactRenderers={artifactRenderers} />;
```

- `toolName` matches tool calls. `type` is the app's own label, used for stored artifacts and categories.
- Renderers are captured when the interface mounts; changing the array later has no effect.
- `artifactCategories` groups artifacts in the sidebar and browser.

## Handle every parser input

The parser runs on every stream update and when a stored artifact opens:

| When | Input | The parser should |
| --- | --- | --- |
| The tool call is streaming | `args` may be partial JSON; `response` is `null` | Return `null` until there is enough to show |
| The tool result arrived | `response` holds the result | Validate it and treat it as the real content |
| A stored artifact opens | `args` is `undefined`; `response` is the stored `content` | Parse the same shape, without tool arguments |

Return `meta: null` while streaming. Return `{ id, version, heading }` once complete, to register the artifact in the thread. Keep `id` stable across edits and increase `version`.

## Store artifacts

`ChatStorage.artifact` has three methods:

- `list(params)`
- `get(id)`
- `update({ id, content })`, which renderers call through `useArtifactStorage()` to save edits.

There is no `create`: the tool that makes the artifact writes it. Store the complete tool result as `content`, with the renderer's `type` and the owning thread id.

- With Gateway conversations, `useOpenuiCloudStorage` includes artifact storage by default (`features.artifact`); the scaffold turns it off with `features: { artifact: false }`.
- For artifacts that the app's own tools create, implement `ChatStorage.artifact` against the app's backend, and combine it with the thread storage:

```ts
const storage = useMemo(
  () => ({ thread: cloudStorage.thread, artifact: appArtifactStorage }),
  [cloudStorage],
);
```

## Generated HTML

Render generated HTML in a sandboxed `<iframe srcDoc>` with the narrowest `sandbox` permissions, a Content Security Policy, and no network access. Treat it as untrusted. The [html-artifact example](https://github.com/thesysdev/openui/tree/main/examples/miscellaneous/html-artifact) shows the pattern.

## Verify

1. Run the tool through the real loop; the preview and the full view both render.
2. Partial arguments, a malformed result, a cancelled stream, and a failed tool all render without errors.
3. With storage: reopen the artifact from the browser and get the same content; an edit keeps the id and bumps the version; another user cannot read or edit it.
4. Run the project's typecheck and build.

## Sources

- [Artifacts](https://www.openui.com/docs/agent/core-concepts/artifacts)
- [Custom artifacts](https://www.openui.com/docs/agent/guides/custom-artifacts)
- [`defineArtifactRenderer`](https://www.openui.com/docs/agent/reference/define-artifact-renderer)
- [Renderer types](https://github.com/thesysdev/openui/blob/main/packages/react-headless/src/store/artifactRendererTypes.ts)
