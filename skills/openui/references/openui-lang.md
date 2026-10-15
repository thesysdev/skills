# OpenUI Lang 1.0

Use this when reading, writing, validating, or debugging OpenUI Lang by hand: prompt examples, test fixtures, stored responses, parser errors. Models learn the language from the generated system prompt, so app code rarely needs it. Component names below are examples; each library defines its own, and the [prompt options](component-library.md#4-prompt-options) decide which features the model is taught.

## Programs

A program is one statement per line: `name = expression`.

```text
root = Card([header, chart])
header = Header("Monthly Revenue", "Last 6 months")
chart = BarChart(labels, [series])
labels = ["Jan", "Feb", "Mar", "Apr", "May", "Jun"]
series = Series("Revenue", [12000, 15000, 14000, 18000, 21000, 25000])
```

- **Entry.** Rendering starts at `root`, and whatever it reaches renders. Write `root` first, so the page shape streams before its contents. With no `root`, the first value statement that calls the library's root component is the entry, with a warning.
- **Positional arguments** follow the order of the component's props. Optional arguments at the end can be left out; `null` skips one in the middle.
- **References.** A name can be used before its statement. Until the statement arrives, it is unresolved: it renders nothing and is left out of lists. That is "not yet", not an error.
- **Duplicates.** When two statements share a name, the later one wins.
- **Orphans.** Statements the entry cannot reach are not rendered.
- **Lists.** A list prop needs `[...]`, even for one item.
- **Literals.** Strings use double or single quotes. Commas between arguments are optional. `//` and `#` start comments.
- **Fences.** If the text has code fences, only their contents are read.
- **Multi-line statements.** A statement continues across lines inside unclosed brackets or strings, and across a multi-line ternary.

## Streaming

The client parses after every chunk and closes open strings and brackets, so every partial program renders. When the host ends the stream (in React, `isStreaming` turns false), the result equals parsing the full text at once. Queries and mutations wait until the stream ends.

## Expressions and built-ins

- **Operators:** arithmetic, comparison, `&&`, `||`, `!`, and `cond ? a : b`. `+` joins strings when either side is a string.
- **Member access:** `a.field`, `a[i]`, and `list.length`. On a list, `rows.amount` takes `amount` from every element.

Built-ins start with `@`; without the `@`, a call is read as a plain reference.

| Group | Built-ins |
| --- | --- |
| Aggregation | `@Count`, `@Sum`, `@Avg`, `@Min`, `@Max`, `@First`, `@Last` |
| Reshaping | `@Filter(list, field, op, value)` with `==`, `!=`, `>`, `<`, `>=`, `<=`, `contains`; `@Sort(list, field, "desc"?)` |
| Math | `@Round(n, decimals?)`, `@Abs`, `@Floor`, `@Ceil` |
| Iteration | `@Each(list, "item", template)` |

```text
urgent = @Filter(tickets.rows, "priority", "==", "high")
rows = @Each(@Sort(urgent, "createdAt", "desc"), "t", Row(t.title, Tag(t.priority)))
```

A library can add **custom functions**, called the same way (`@Percent(done, total)`); they are pure and run in the browser. An unknown `@Name` evaluates to null and reports `unknown-function`.

## State

`$name` holds state that the user or an action writes.

```text
$days = "7"
filter = Select("days", [SelectItem("7", "7 days"), SelectItem("30", "30 days")], null, null, $days)
label = TextContent("Showing " + $days + " days")
```

- Passing `$days` to a prop whose signature shows `$binding<type>` binds it both ways. Anything that reads it updates when it changes.
- An undeclared `$name` starts as `null`.
- Reparsing never overwrites a value the user entered.

## Queries and mutations

```text
tickets = Query("list_tickets", { status: $status }, { rows: [] })
create = Mutation("create_ticket", { title: $title })
saveBtn = Button("Create", Action([@Run(create), @Run(tickets), @Reset($title)]))
```

- `Query(tool, args, defaults, refreshSeconds?)` runs when the stream ends, and again when a `$variable` written directly in `args` changes. Until data arrives, the defaults are shown.
- `Mutation(tool, args)` runs only through `@Run`. Its value is `{ status, data, error }`, where `status` is `idle`, `loading`, `success`, or `error`.
- Both must be the whole right-hand side of a statement. Inside a component call they report `inline-reserved`.
- Name them without `$`.
- The browser runs them through the renderer's `toolProvider`. Their results never enter the conversation.

## Actions

```text
askBtn = Button("Explain", @ToAssistant("Explain the revenue chart", { chart: "revenue" }))
docsBtn = Button("Docs", @OpenUrl("https://example.com/docs"))
closeBtn = Button("Close", Action([@Run(save), @Set($open, false), @ToAssistant("Closed")]))
```

- **Built-in actions:** `@ToAssistant(message, context?)`, `@OpenUrl(url)`, `@Set($var, value)`, `@Reset($a, …)`, and `@Run(queryOrMutation)`.
- **Wrapping:** one action needs no wrapper; several go in `Action([...])` and run in order. A failed mutation stops the rest.
- **Custom actions** from the library (`@CopyToClipboard(text)`) are delivered to the host's `onAction`.

The host receives events of this shape:

```json
{ "type": "continue_conversation", "humanFriendlyMessage": "Closed",
  "params": { "context": { "chart": "revenue" } }, "formName": "ticket", "formState": { … } }
```

`open_url` events carry `params.url`.

## Forms

Inputs take validation rules as an object: `Input("email", "you@company.com", "email", { required: true, email: true })`.

- **Rules:** `required`, `email`, `url`, `numeric`, `minLength`, `maxLength`, `min`, `max`, and `pattern`.
- **Grouping:** inputs inside a form component group under the form's name. Their values travel with every event fired from inside the form.

## Incremental edits

In edit mode (`editMode` prompt option), the model answers with only the changed statements.

- The host merges them into the current program with `mergeStatements`, after the patch finishes streaming.
- Merging replaces statements by name, appends new ones, and removes statements `root` no longer reaches.
- `name = null` deletes a statement.

## Stored messages

The host stores a response with marker lines the model never sees:

```text
]]>openui:content?library=support%401.2.0
root = Card([title])
title = TextContent("Order")
]]>openui:context
{"formState": {"note": "rush"}}
]]>openui:end
```

- `content` holds the program, `context` holds host JSON such as form state, and `end` marks a completed stream.
- Strip every marker line before sending history to the model.
- `buildMessage` and `parseMessage` from `@openuidev/react-lang` write and read this format, and `Renderer` accepts it directly.

## Errors

A mistake removes the smallest broken piece, and the rest renders. Each error carries `code`, `statementId`, `component`, `path`, `message`, and `hint`, ready to send back to the model.

| Code | Meaning | What happens |
| --- | --- | --- |
| `unknown-component` | Component not in the library | Dropped |
| `missing-required`, `null-required` | Required prop missing or null | Schema default, else the component is dropped |
| `type-mismatch` | Wrong type or enum value | Default, else the prop is removed or the component dropped |
| `excess-args` | More arguments than props | Extras dropped |
| `inline-reserved` | `Query` or `Mutation` inside an expression | Evaluates to nothing |
| `unknown-function` | Unknown `@Name` | Evaluates to null |
| `no-root` | No usable entry when the stream ends | Fatal unless a fallback entry renders |
| `parse-failed`, `parse-exception` | No statements could be read | Fatal |
| `runtime-error`, `render-error` | An expression or component threw | The prop becomes null; the last good render is kept |
| `tool-not-found`, `tool-error`, `mcp-error` | A query or mutation failed | The query keeps its defaults; the mutation result holds the error |

## Validate a program

```ts
import { createParser } from "@openuidev/react-lang";
import { openuiChatLibrary } from "@openuidev/react-ui";

const result = createParser(openuiChatLibrary.toJSONSchema(), "Card").parse(program);
const ok =
  result.root !== null &&
  !result.meta.incomplete &&
  result.meta.errors.length === 0 &&
  result.meta.unresolved.length === 0 &&
  result.meta.orphaned.length === 0;
```

- Pass the library's root: `"Card"` for `openuiChatLibrary`, `"Stack"` for `openuiLibrary`, or a custom library's `root`. From a spec file, use `createParser(spec.schema, spec.root)`.
- Errors are in `result.meta.errors`; there is no top-level `result.errors`.
- `createStreamingParser` takes the same arguments and has `push(chunk)` for streamed text.
- Judge a program only after its stream ends; `incomplete` and `unresolved` are normal while it streams.

## Reserved names

Libraries must not use these for components, functions, or actions:

- `true`, `false`, `null`, `Query`, `Mutation`, `Action`;
- `@Count`, `@First`, `@Last`, `@Sum`, `@Avg`, `@Min`, `@Max`, `@Filter`, `@Sort`, `@Round`, `@Abs`, `@Floor`, `@Ceil`, `@Each`;
- `@Set`, `@Reset`, `@Run`, `@ToAssistant`, `@OpenUrl`.

## Sources

- [Overview](https://github.com/thesysdev/openui/blob/main/spec/overview.md)
- [Language](https://github.com/thesysdev/openui/blob/main/spec/language.md)
- [Prompt and LibrarySpec](https://github.com/thesysdev/openui/blob/main/spec/prompt.md)
