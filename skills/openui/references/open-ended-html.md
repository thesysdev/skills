# Open-ended HTML

Use this pattern when the user wants full generative UI, generated mini-apps, raw HTML, or a sandboxed iframe rather than a fixed component catalog.

For an application tool that returns HTML and opens it through a custom artifact renderer, use [Agent Interface Artifacts](artifacts.md). The pattern below embeds HTML in the assistant's OpenUI Lang response; both approaches can use the same sandboxed HTML view.

The canonical implementation is [`examples/miscellaneous/html-artifact`](https://github.com/thesysdev/openui/tree/main/examples/miscellaneous/html-artifact).

## Pattern

1. Define one OpenUI component with `title` and `document` string props.
2. Add it to a library with an ordinary container root, next to a text or Markdown component, so the model chooses per reply whether to emit HTML. Making the HTML component the root would force every reply to be a page.
3. Generate the system prompt with `openui generate`.
4. Pass that library to `AgentInterface.componentLibrary`.
5. Use `useIsStreaming()` to distinguish incoming source from the completed document.
6. Keep only a compact status preview inline.
7. Open a `DetailedViewPanel` containing Raw and Rendered tabs.
8. Show source in Raw, a loading state in Rendered while streaming, and a sandboxed `iframe srcDoc` when complete.

Use the existing assistant response stream. Do not add another streaming endpoint or a tool unless the user's application independently requires one.

## Prompt contract

Tell the model to:

- use the markdown component for normal conversation, and the artifact only when asked to build something interactive;
- emit a self-contained HTML document or fragment;
- use inline CSS and JavaScript;
- avoid external resources and network requests;
- omit Markdown fences;
- escape newlines, quotes, and backslashes for the OpenUI Lang string.

Keep one valid example in `promptOptions.examples`, then regenerate the prompt and spec following the host's build convention. First-party examples generate these files locally and ignore them in Git.

## Security

Treat generated HTML as untrusted. Before production, normalize and validate it, enforce a Content Security Policy, restrict network access, keep iframe sandbox permissions narrow, and validate iframe messages.
