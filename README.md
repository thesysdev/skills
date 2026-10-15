# OpenUI Skill

Agent-ready guidance for building generative UI with [OpenUI](https://www.openui.com/). This repository contains one skill that helps AI coding assistants build OpenUI apps, add generative UI to existing apps, and run them on OpenUI Gateway.

## Installation

```bash
npx skills add thesysdev/skills
```

Once installed, try prompts such as:

- "Build a generative UI agent that answers questions about our sales data."
- "Start a new OpenUI app with the shadcn design system."
- "Add generative UI to my assistant-ui chat."
- "Move my OpenAI Chat Completions backend to OpenUI Gateway."
- "Build a custom OpenUI component library for these domain objects."
- "Deploy this app and give me a preview URL."

## Skill contents

| File | Covers |
| --- | --- |
| [`SKILL.md`](skills/openui/SKILL.md) | What OpenUI is, which reference to read for each job, the rules every task follows, and the package map |
| [`new-app.md`](skills/openui/references/new-app.md) | Scaffolding with `openui create`, the API key handoff, the default scaffold, customizing, and verifying |
| [`existing-app.md`](skills/openui/references/existing-app.md) | Adding a component library, pointing the backend at Gateway, and rendering in assistant-ui, CopilotKit, or a custom chat |
| [`gateway.md`](skills/openui/references/gateway.md) | Gateway with Chat Completions: client setup, BYOK, the system prompt, conversation storage, tools, and security |
| [`gateway-responses.md`](skills/openui/references/gateway-responses.md) | Apps that already use the Responses API: history models and hosted tools |
| [`autofix.md`](skills/openui/references/autofix.md) | Correcting OpenUI Lang when inference stays with the user's provider |
| [`component-library.md`](skills/openui/references/component-library.md) | Defining and extending libraries, generating the spec, and keeping stored responses compatible |
| [`agent-interface.md`](skills/openui/references/agent-interface.md) | Agent Interface: stream adapters, storage, shell slots, and navigation |
| [`artifacts.md`](skills/openui/references/artifacts.md) | Tool output as artifacts: renderers, parser inputs, and storage |
| [`theming.md`](skills/openui/references/theming.md) | `ThemeProvider`, tokens, and light/dark/system mode |
| [`reliability.md`](skills/openui/references/reliability.md) | DevTools, Reliability Monitoring, and fixing generation errors |
| [`deploy.md`](skills/openui/references/deploy.md) | `openui deploy` and its options |
| [`openui-lang.md`](skills/openui/references/openui-lang.md) | OpenUI Lang 1.0: syntax, state, queries, actions, errors, and validation |
| [`examples.md`](skills/openui/references/examples.md) | Where the example catalog lives and how to use it |

## Learn more

- [OpenUI documentation](https://www.openui.com/docs)
- [OpenUI Gateway](https://www.openui.com/docs/gateway)
- [Reliability Monitoring](https://www.openui.com/docs/reliability)
- [OpenUI source and examples](https://github.com/thesysdev/openui)
- [OpenUI Lang specification](https://github.com/thesysdev/openui/tree/main/spec)
