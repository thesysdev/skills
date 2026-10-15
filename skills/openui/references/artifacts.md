# Show tool output as artifacts

Use this when an agent produces something the user should preview, open, revisit, or edit in Agent Interface: a document, an HTML app, a presentation, a dashboard. An artifact is a tool result plus a renderer the app supplies. Agent Interface provides the inline preview slot, the side panel, and, with storage, an artifact browser. Agent Interface itself is covered in [agent-interface.md](agent-interface.md).

## How it fits together

1. The app declares a tool (for example `create_artifact`) and runs it in its tool loop. The tool name and payload shape are the app's choice.
2. The tool call and its result reach the browser through the stream. Check that the route's stream carries tool results, not just tool calls; a result fed only to the next model request never reaches the UI. Raw Chat Completions deltas carry only the call's arguments, so the app's tool executor must also send the result to the browser. The same applies to a Responses `function_call_output`.
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
- Artifact content need not be OpenUI Lang, and needs no `root`. If it is OpenUI Lang, render it with `Renderer` and its library.

## Handle every parser input

The parser runs on every stream update and when a stored artifact opens:

| When | Input | The parser should |
| --- | --- | --- |
| The tool call is streaming | `args` may be partial JSON; `response` is `null` | Return `null` until there is enough to show |
| The tool result arrived | `response` holds the result | Validate it and treat it as the real content |
| A stored artifact opens | `args` is `undefined`; `response` is the stored `content` | Parse the same shape, without tool arguments |

Return `meta: null` while streaming. Return `{ id, version, heading }` once complete, to register the artifact in the thread. Keep `id` stable across edits and increase `version`. Registering it in the thread does not save it anywhere.

## Store artifacts

`ChatStorage.artifact` has three methods:

- `list(params)`
- `get(id)`
- `update({ id, content })`, which renderers call through `useArtifactStorage()` to save edits.

There is no `create`: the tool that makes the artifact writes it. Store the complete tool result as `content`, with the renderer's `type` and the owning thread id.

- **Inline after reload:** an artifact reappears in its message only if the tool call and its result are kept in the thread's stored history.
- **Edits:** load the stored artifact, check that the user may edit it, apply the change, save the new version, and emit an updated tool result with the same id.

- With Gateway conversations, `useOpenuiCloudStorage` includes artifact storage by default (`features.artifact`); the scaffold turns it off with `features: { artifact: false }`.
- For artifacts that the app's own tools create, implement `ChatStorage.artifact` against the app's backend, and combine it with the thread storage:

```ts
const storage = useMemo(
  () => ({ thread: cloudStorage.thread, artifact: appArtifactStorage }),
  [cloudStorage],
);
```

## Generated HTML

Two ways to show HTML the model writes:

- **As a tool artifact:** the pattern above, with an `actual` view that renders the HTML.
- **Inside the response itself,** with no tool. The [html-artifact example](https://github.com/thesysdev/openui/tree/main/examples/miscellaneous/html-artifact) does this:
  1. Add one component with `title` and `document` string props to the library. Do not make it the root, or every reply becomes an HTML page.
  2. In its renderer, use `useIsStreaming()` to tell incoming source from a finished document.
  3. Keep only a compact status inline. Open a `DetailedViewPanel` with Raw and Rendered tabs: the source in Raw, a loading state in Rendered while streaming, then a sandboxed iframe.
  4. Use the existing response stream; add no extra endpoint or tool.

Prompt rules for either way (in `additionalRules` and one valid example):
- use the component only when asked to build something interactive;
- write a self-contained document with inline CSS and JavaScript;
- no external resources or network requests;
- no Markdown fences;
- escape newlines, quotes, and backslashes inside the OpenUI Lang string.

Security: treat the HTML as untrusted.
- Normalize and validate it before rendering.
- Render it in `<iframe srcDoc>` with the narrowest `sandbox` permissions, a Content Security Policy, and no network access.
- Validate every `postMessage` from the iframe.

## Verify

1. Run the tool through the real loop; the preview and the full view both render.
2. Partial arguments, a malformed result, a cancelled stream, and a failed tool all render without errors.
3. With storage: reopen the artifact from the browser and get the same content; reload the thread and see the inline preview again; an edit keeps the id and bumps the version; another user cannot read or edit it.
4. Run the project's typecheck and build.

## Sources

- [Artifacts](https://www.openui.com/docs/agent/core-concepts/artifacts)
- [Custom artifacts](https://www.openui.com/docs/agent/guides/custom-artifacts)
- [`defineArtifactRenderer`](https://www.openui.com/docs/agent/reference/define-artifact-renderer)
- [Renderer types](https://github.com/thesysdev/openui/blob/main/packages/react-headless/src/store/artifactRendererTypes.ts)
