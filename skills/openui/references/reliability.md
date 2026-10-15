# Make generated UI reliable

Use this when renders are blank, partial, or wrong, or when the user wants to track generation errors in production. Models sometimes invent a component, pass an invalid value, leave a reference undefined, or stop mid-program. Measure first, then fix the cause, then measure again.

## Find the errors in development

OpenUI DevTools lists each stream's parse and render errors, with the statement, component, and error code.

- `@openuidev/react-lang` mounts it automatically in development builds, and scaffolded apps include it.
- **Inspect** shows each settled stream. **Debug** replays a failing response against the same library, without calling the model again.
- To configure it, install `@openuidev/devtools` as a dev dependency and mount one `<OpenUIDevtools />`; it replaces the automatic one. Props: `position`, `theme`, `errorsOnly`, `maxEvents`, `autoOpenOnError`, `enabled`.
- It loads from `cdn.jsdelivr.net`. A strict CSP must allow that origin, or the widget does not appear.
- It renders nothing in production unless `enabled` is set.

In code, `<Renderer onError={(errors) => …} />` receives the same errors, and `onParseResult` receives each parse result. The error codes are in [openui-lang.md](openui-lang.md#errors).

## Track errors in production (Reliability Monitoring)

- **Through Gateway or Autofix:** tracking is automatic. Open the [Reliability dashboard](https://console.thesys.dev/reliability); no SDK is needed.
- **Neither:** install `@openuidev/observability-cloud` in the app that renders OpenUI and initialize it once, before rendering:

  ```ts
  import * as Observability from "@openuidev/observability-cloud";

  Observability.init({ apiKey: process.env.NEXT_PUBLIC_OPENUI_CLIENT_KEY! });
  ```

  The SDK uses a **client key** from the [client keys page](https://console.thesys.dev/client-api-keys). Client keys are meant for the browser; never use `THESYS_API_KEY` here. The user creates the client key.
- The SDK also works alongside Gateway or Autofix to capture parsing and rendering signals from the browser.

## Fix the cause

Choose by the errors you see:

1. **Simplify the schema.** Use distinct component names, clear descriptions, focused props, and unambiguous enum values. Remove overlapping components. Group related ones with `componentGroups`. See [component-library.md](component-library.md#3-design-for-reliable-generation).
2. **Refine the prompt.** Add a narrow rule for each recurring error, or a valid example for a combination the model gets wrong. Test each change; one wrong example can cause broad regressions.
3. **Compare models** with the app's real library and prompts. Run each prompt several times and compare errors, latency, and cost.
4. **Correct the output.** Gateway corrects errors while streaming, so broken UI never reaches users. If the app does not use Gateway yet, move its inference to Gateway; see [gateway.md](gateway.md). If the user explicitly keeps calling their provider directly or declines Gateway, follow [autofix.md](autofix.md).

Also check the plumbing:

- The server's spec and the browser's library come from the same library file, and the spec was regenerated after the last change.
- `isStreaming` stays true until the stream closes.
- Earlier assistant turns go back to the model as OpenUI Lang.

## Measure

Generation is nondeterministic. One good run proves nothing.

- Pick representative prompts and run each several times, before and after a change.
- Count full failures, partial renders, and parse errors, not just blank screens.

## Sources

- [Reliability](https://www.openui.com/docs/openui-lang/reliability)
- [Developer tools](https://www.openui.com/docs/openui-lang/developer-tools)
- [Reliability Monitoring setup](https://www.openui.com/docs/reliability/installation)
- [Improve reliability](https://www.openui.com/docs/reliability/optimizing)
