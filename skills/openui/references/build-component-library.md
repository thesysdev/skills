# Build an OpenUI Component Library

Read this reference completely when defining, extending, migrating, or validating an OpenUI component library. The model-facing library specification and the client-side renderer library are one contract: component names, prop order, descriptions, root, groups, tools, and interaction features must stay synchronized.

## Choose the Smallest Library Change

- Use `openuiLibrary` or `openuiChatLibrary` from `@openuidev/react-ui` for the built-in open-source React libraries.
- Add domain-specific components when the built-in set cannot express the application's objects or actions.
- Build a custom library when the application must use its own design system, needs a focused domain vocabulary, or targets another supported runtime.

Do not create a parallel component system just to rename existing components. Every added component makes the prompt longer and gives the model another choice to get wrong.

Before editing, inspect the existing library export, generated prompt and spec files, renderer wiring, tool provider, and a few representative generated programs.

## Define Components

Use the framework package that owns the renderer:

- React: `@openuidev/react-lang`
- Vue: `@openuidev/vue-lang`
- Svelte: `@openuidev/svelte-lang`
- Prompt/schema-only code: `@openuidev/lang-core` with an opaque renderer value

Use Zod v4 schemas from `zod/v4`. In React, use `.tsx` when the library file contains JSX.

```tsx
import { MetricView } from "@/components/metric-view";
import { createLibrary, defineComponent } from "@openuidev/react-lang";
import { z } from "zod/v4";

const Metric = defineComponent({
  name: "Metric",
  description: "A labeled KPI with an optional trend direction.",
  props: z.object({
    label: z.string(),
    value: z.string(),
    trend: z.enum(["up", "down"]).optional(),
  }),
  component: ({ props }) => <MetricView {...props} />,
});

const Panel = defineComponent({
  name: "Panel",
  description: "Top-level vertical container for dashboard content.",
  props: z.object({
    children: z.array(Metric.ref),
  }),
  component: ({ props, renderNode }) => (
    <section>{renderNode(props.children)}</section>
  ),
});

export const appLibrary = createLibrary({
  id: "acme-operations@1",
  root: "Panel",
  components: [Panel, Metric],
});
```

The `root` must name a component in the library. Give the library a stable optional `id` when the application needs to identify revisions across generated specs or deployments.

## Design for Model Reliability

- Use descriptive, distinct component names and descriptions that explain when to choose each component.
- Keep schemas flat. Prefer several composable components over deeply nested objects.
- Compose children through component `.ref` schemas. Use `z.union([...])` only when the container genuinely accepts several child types.
- Order Zod object keys deliberately: required and distinctive props first, optional props last. OpenUI Lang maps positional arguments using this order.
- Keep the library focused on components the model should generate. Do not expose an entire product design system by default.
- Choose a predictable root that can render before its referenced children arrive, then keep `root = ...` first in generated examples for progressive streaming.
- Use `componentGroups` and short group `notes` when they materially help the model choose related components or avoid an invalid combination.
- Add one or two valid `PromptOptions.examples` for unusual component shapes. Do not compensate for ambiguous schemas with a long universal ruleset.

Inspect the generated prompt after schema changes. A TypeScript-valid library can still produce ambiguous or unnecessarily large model instructions.

## Add State, Actions, and Tools Deliberately

Enable only interaction features the application can execute:

- Bindings and reactive state require the renderer/store path to handle `$variables`, `@Set`, and `@Reset`.
- `Query` and `Mutation` require a matching tool provider or query loader. Keep them as top-level statements in generated OpenUI Lang.
- Actions such as `@Run`, `@ToAssistant`, and `@OpenUrl` require the corresponding renderer/action contract.
- Tool descriptors in prompt options must match executable, authorized server tools. Never advertise a tool only in the prompt.

When tools are enabled, provide valid `toolExamples` that use the application's actual tool names and result shapes. Keep authorization, input validation, and side-effect policy outside the model.

## Generate the Handover Spec

Every backend needs the serialized spec of the library the client renders, including a built-in one. The CLI templates export the built-in library from a small module:

```ts
export { openuiLibrary as library } from "@openuidev/react-ui/genui-lib";
```

Generate the spec whenever component names, descriptions, prop schemas, root, groups, or prompt options change:

```bash
npx @openuidev/cli@latest generate --spec ./src/lib/app-library.tsx --out ./src/generated/library-spec.json
```

The CLI detects the library export automatically. Pass `--export <name>` or `--prompt-options <name>` when the module exports more than one candidate.

`--spec` and `--json-schema` are not interchangeable:

- `--spec` produces the serialized contract consumed by `generateSystemPrompt()` and by Autofix's `createAutofix`, which also reads its `schema` field.
- `--json-schema` produces a schema for external tooling and lacks the prompt contract.

When the repository already regenerates prompt and spec files during its build, treat them as build output and follow its convention for committing or ignoring them.

## Connect the Backend

Compile the prompt with `generateSystemPrompt()` from `@openuidev/lang-core`.

Self-hosted generation renders the full prompt locally:

```ts
import { generateSystemPrompt, type LibrarySpec } from "@openuidev/lang-core";
import librarySpec from "./generated/library-spec.json";

const systemPrompt = generateSystemPrompt({
  library: librarySpec as LibrarySpec,
  promptOptions,
});
```

Gateway generation sends the same spec as Gateway's prompt config:

```ts
const gatewayPrompt = generateSystemPrompt({
  cloud: true,
  library: librarySpec,
  instructions: "Optional trusted application instructions.",
  promptOptions, // only examples, preamble, and additionalRules apply
});
```

Put the result in the Responses `instructions` or in the Chat Completions `role: "system"` message ([Gateway prompt](gateway/overview.md#configure-the-prompt)). Keep user content out of `instructions`, `preamble`, rules, and examples.

## Connect the Renderer

Pass the runtime library that produced the spec to the client:

```tsx
<AgentInterface componentLibrary={appLibrary} llm={llm} />
```

For renderer-only surfaces:

```tsx
<Renderer library={appLibrary} response={response} isStreaming={isStreaming} />
```

Regenerate the spec every time the client library changes, including for a built-in library after a package upgrade. Unknown components, wrong positional arguments, and blank or partial renders usually mean the spec and the client library have drifted apart.

## Verify

1. Run `openui generate --spec` successfully against the actual library export.
2. Inspect the generated component signatures, root, groups, schema order, tools, rules, and examples.
3. Parse representative settled OpenUI Lang programs with the library schema and inspect `result.meta.errors`.
4. Stream representative programs and confirm the root renders early while forward references resolve.
5. Exercise bindings, actions, queries, mutations, and tool results that the library exposes.
6. Run representative user prompts repeatedly across the intended model choices; compare structural failures, latency, and prompt size rather than trusting one success.
7. Run the host application's formatter, typecheck, tests, and production build.

## First-Party References

- `https://www.openui.com/docs/openui-lang/defining-components`
- `https://www.openui.com/docs/openui-lang/system-prompts`
- `https://www.openui.com/docs/openui-lang/reliability`
- `https://www.openui.com/docs/gateway/generate-openui-lang`
- `https://www.openui.com/docs/api-reference/cli#generate`
- `https://github.com/thesysdev/openui/tree/main/examples/design-systems`
