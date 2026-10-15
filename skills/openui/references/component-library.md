# Build or change the component library

Use this when defining, extending, or changing the components the model can use. The library is one contract with two consumers: the browser renders with it, and the server prompts the model with its spec. Both must come from the same library.

## 1. Start from the built-in library when it fits

`@openuidev/react-ui` ships two libraries:

| Library | Root | For |
| --- | --- | --- |
| `openuiLibrary` | `Stack` | General UI: charts, tables, forms, cards, images, tabs, modals |
| `openuiChatLibrary` | `Card` | Chat answers: follow-ups, steps, callouts, list and section blocks |

Each comes with prompt options (`openuiPromptOptions`, `openuiChatPromptOptions`). To add a few domain components, extend the built-in library instead of replacing it:

```tsx
import { openuiLibrary } from "@openuidev/react-ui";
import { ProductCard } from "./product-card";

export const library = openuiLibrary.extend({
  components: { add: [ProductCard] },
});
```

`extend({ components, functions, actions })` returns a new library:

- `components.add` appends components. An entry `{ component, slots }` also allows the component inside those slots. A slot is `"Parent.prop"`, or `"Parent"` when the parent has one content slot.
- `components.override` replaces a component by name.
- `components.remove` drops components everywhere.
- `functions.add` and `actions.add` add custom functions and actions.

Adding a name that already exists is an error.

Build a custom library when the app has its own design system or a focused domain vocabulary. Do not rebuild the built-in components only to rename them.

## 2. Define components

Use the runtime package of the app's framework: `@openuidev/react-lang` for React, or `@openuidev/vue-lang`, `@openuidev/svelte-lang`, or `@openuidev/angular-lang`. Schemas use Zod 4.

```tsx
import { createLibrary, defineComponent } from "@openuidev/react-lang";
import { z } from "zod";
import { MetricView } from "@/components/metric-view";

const Metric = defineComponent({
  name: "Metric",
  description: "A labeled KPI. value is preformatted text, e.g. \"$1.2M\".",
  props: z.object({
    label: z.string(),
    value: z.string(),
    trend: z.enum(["up", "down"]).optional(),
  }),
  component: ({ props }) => <MetricView {...props} />,
});

const Panel = defineComponent({
  name: "Panel",
  description: "Top-level vertical container.",
  props: z.object({ children: z.array(Metric.ref) }),
  component: ({ props, renderNode }) => <section>{renderNode(props.children)}</section>,
});

export const library = createLibrary({
  id: "acme-ops",
  root: "Panel",
  components: [Panel, Metric],
});
```

- **Arguments are positional, in the key order of `props`.** Required props come first, optional props last.
- **Child components** are typed with `Child.ref`. Use `z.union([A.ref, B.ref])` when a slot accepts several types.
- **`root`** names the component a page starts from. It must be in `components`.
- **Actions:** a prop typed with `action()` from `@openuidev/react-lang` takes an action. The component fires it with `useTriggerAction()`.
- **Custom functions:** `defineFunction({ name, description, params, returns, fn })` adds a pure, synchronous `@Name(...)` call. It runs in the browser.
- **Custom actions:** `defineAction({ name, description, params })` adds an `@Name(...)` action. The host receives it in `onAction`.
- Names must not collide with built-ins (`Query`, `Mutation`, `Action`, `@Count`, …). The full list is in [openui-lang.md](openui-lang.md#reserved-names).

## 3. Design for reliable generation

- Give components distinct names, and descriptions that say when to use each one. A description should state what the type does not: units, ranges, and when to pick each enum value.
- Keep props flat. Prefer several small components over deeply nested objects.
- Expose only components the model should generate, not the whole design system.
- Use `componentGroups: [{ name, components, notes? }]` when grouping helps the model choose.
- Add one or two valid `examples` in the prompt options for unusual shapes. Test every rule and example: a wrong example causes regressions.

## 4. Keep stored responses rendering

Stored responses have no prop names; position is the meaning. When changing a component that is already in use:

1. Add props only at the end. Never reorder or remove a prop; leave an unused one in place.
2. Give every new required prop a default.

## 5. Generate the spec for the server

```bash
npx @openuidev/cli@latest generate src/lib/openui.tsx --spec --out src/lib/openui.spec.json
```

- **The entry** is the positional argument. It must export the library: the CLI checks `library`, then `default`, then any library export. `--export <name>` and `--prompt-options <name>` choose specific exports.
- **`--spec`** writes the serialized library spec, which the server needs.
- **Without `--spec`,** `--out prompt.txt` writes the full system prompt and puts the spec next to it as `prompt.spec.json`.
- **`--json-schema`** writes only the JSON Schema, for other tooling.

Regenerate the spec whenever the library changes, for example from a `prebuild` script. The scaffold keeps it at `src/generated/spec.json`.

## 6. Connect the server and the browser

Server, through Gateway:

```ts
import { generateSystemPrompt, type LibrarySpec } from "@openuidev/lang-core";
import library from "@/lib/openui.spec.json";

const systemPrompt = generateSystemPrompt({ cloud: true, library: library as LibrarySpec });
```

Browser, with the same library:

```tsx
<Renderer response={text} library={library} isStreaming={isStreaming} />
// or
<AgentInterface llm={llm} componentLibrary={library} />
```

Blank or partial renders and `unknown-component` errors usually mean the spec and the browser library differ. Regenerate the spec.

## Verify

1. `openui generate --spec` succeeds on the library file.
2. Read the generated signatures: names, prop order, required props, root.
3. Parse a few hand-written programs and check `result.meta.errors` (see [openui-lang.md](openui-lang.md#validate-a-program)).
4. Run representative prompts several times and watch DevTools for errors (see [reliability.md](reliability.md)).
5. Run the project's typecheck and build.

## Sources

- [Defining components](https://www.openui.com/docs/openui-lang/defining-components)
- [System prompts](https://www.openui.com/docs/openui-lang/system-prompts)
- [LibrarySpec and prompt spec](https://github.com/thesysdev/openui/blob/main/spec/prompt.md)
- [Design-system examples](https://github.com/thesysdev/openui/tree/main/examples/design-systems)
