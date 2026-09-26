# Improve and Measure Reliability

Generated interfaces are nondeterministic. A response that renders once can fail on a later run: an invented component, an invalid value, an unresolved reference, or a stream that ends before the component graph is complete. Use this guide to measure failures, reduce them, correct what remains, and monitor production.

## Measure First

Build a baseline before changing anything:

1. Collect representative user prompts for the app's real component library.
2. Generate each prompt several times per model under consideration.
3. Record structural errors, partial or blank renders, latency, and cost.
4. Repeat the same run after every change. One successful generation is not evidence of an improvement.

## Inspect in Development

OpenUI DevTools shows each stream's text, parser issues, validation errors, and timing, including failures hidden behind a partly rendered interface. `@openuidev/react-lang` mounts it automatically in browser development builds, and CLI templates include it. To configure it in another app, install `@openuidev/devtools` as a development dependency and mount one `OpenUIDevtools` instance, which replaces the automatic one. Use Inspect to review settled streams and Debug to replay failing output against the same library without calling the model. Keep DevTools out of production builds.

## Reduce Failures

Use the measured failure types to choose a fix:

1. **Simplify the schema.** Distinct component names, clear descriptions, focused props, unambiguous enums, and no overlapping components. Group related components with `componentGroups`. See [build-component-library.md](build-component-library.md#design-for-model-reliability).
2. **Refine the prompt.** Add a narrow rule or one valid example for each recurring error. Test each addition against the baseline, since one incorrect example can cause broad regressions.
3. **Compare models.** Run the same prompt set across candidate models with the app's library and compare reliability, latency, and cost.

## Correct Invalid Output

Parse after the stream ends; while it streams, incomplete statements and unresolved references are expected. A settled response is valid when `root` is set and `meta.incomplete` is false, with no `meta.errors`, `meta.unresolved`, or `meta.orphaned` entries.

Choose one correction layer per generation path:

| Generation path | Correction |
| --- | --- |
| Gateway with `generateSystemPrompt({ cloud: true, library })` | Built in. Gateway validates and repairs OpenUI Lang in the stream; do not add Autofix or another rewrite layer. |
| Any other provider or framework | [Autofix](gateway/autofix.md): wrap the stream with `@openuidev/server`, or call the API when a settled response is invalid. |
| No hosted service | An application loop, described below. |

For an application loop, send the parser's errors back to the model with the failing response, allow one bounded retry, and fall back to text or an error state if the retry also fails. Feed back specific codes such as unknown components, missing required props, excess positional arguments, inline `Query`/`Mutation`, and unresolved references. `Renderer` `onError` reports runtime errors in the browser for the same purpose.

Correction repairs OpenUI Lang only. It does not validate business data or make tool execution safe.

## Monitor Production

Reliability Monitoring in the [Thesys Console](https://console.thesys.dev/reliability) groups generation errors by type and shows the output behind each one.

| Generation path | Setup |
| --- | --- |
| Gateway or Autofix | Automatic for every request; no SDK required |
| Any other path | Install `@openuidev/observability-cloud` and initialize it once before rendering, following the [setup guide](https://www.openui.com/docs/reliability/installation) |

The Observability SDK needs a separate [client API key](https://console.thesys.dev/client-api-keys), which is visible in the browser. It never replaces the server-only `THESYS_API_KEY`. The SDK can also run alongside Gateway or Autofix to capture rendering errors from the browser. Monitoring reports errors; it does not correct them.

## Verify

1. Run the baseline prompt set before and after each change, and compare structural errors and partial renders, not only blank screens.
2. Confirm exactly one correction layer runs on each generation path.
3. For production monitoring, trigger a known failure and confirm it appears in the dashboard.

## First-Party References

- `https://www.openui.com/docs/openui-lang/reliability`
- `https://www.openui.com/docs/openui-lang/developer-tools`
- `https://www.openui.com/docs/reliability`
- `https://www.openui.com/docs/reliability/optimizing`
- `https://www.openui.com/docs/autofix`
