# OpenUI Skill

Agent-ready guidance for building generative interfaces with [OpenUI](https://www.openui.com/). This repository contains one skill that helps AI coding assistants work with OpenUI Lang, the OpenUI runtimes, Agent Interface, and OpenUI Gateway.

## What the Skill Covers

- Scaffold new apps on OpenUI Gateway by default with `@openuidev/cli`, and keep a complete self-hosted path for apps that own their provider and storage.
- Choose the right level for each app (self-hosted, Autofix, Gateway Chat Completions, or Gateway Responses), with thread storage as a separate choice.
- Stream and render OpenUI Lang in React, Vue, Svelte, Angular, and no-build browser apps.
- Build custom component libraries with typed schemas, state, actions, queries, and mutations.
- Set up and customize Agent Interface on any backend: streaming adapters, storage, message rendering, layout, theming, navigation, and artifacts.
- Keep an existing assistant-ui, CopilotKit, or custom chat surface when only the OpenUI renderer is needed.
- Repair invalid output with Gateway or Autofix, and track it in Reliability Monitoring.
- Deploy preview and production URLs with `openui deploy`.

## Installation

```bash
npx skills add thesysdev/skills
```

Once installed, try prompts such as:

- “Create a new OpenUI Gateway agent with LangGraph.”
- “Create a self-hosted OpenUI chat that uses my OpenAI key.”
- “Add Agent Interface to my existing Next.js app.”
- “Add Autofix to my OpenUI route without changing providers.”
- “Move this self-hosted OpenUI chat to OpenUI Gateway.”
- “Build a custom OpenUI component library for these domain objects.”

## Skill Contents

| Resource | Purpose |
| --- | --- |
| [`SKILL.md`](skills/openui/SKILL.md) | Names, levels, guide router, packages, OpenUI Lang, rendering, and verification |
| [`self-hosted.md`](skills/openui/references/self-hosted.md) | Self-hosted generation, client wiring, storage, tools, and correction, with what Gateway adds at each step |
| [`agent-interface.md`](skills/openui/references/agent-interface.md) | Agent Interface on Gateway or self-hosted backends: adapters, storage, rendering, shell, and navigation |
| [`artifacts.md`](skills/openui/references/artifacts.md) | Tool results shown through custom renderers, with editing and storage |
| [`build-component-library.md`](skills/openui/references/build-component-library.md) | Component definitions, schema design, spec generation, and backend and renderer wiring |
| [`theme-provider.md`](skills/openui/references/theme-provider.md) | Theme ownership, tokens, light and dark mode, nested scopes, and portals |
| [`reliability.md`](skills/openui/references/reliability.md) | Evaluation, DevTools, correction options, and production monitoring |
| [`open-ended-html.md`](skills/openui/references/open-ended-html.md) | Generated HTML in sandboxed iframes |
| [`examples.md`](skills/openui/references/examples.md) | First-party examples and existing-chat integration guides |
| [`deploy.md`](skills/openui/references/deploy.md) | `openui deploy` for preview and production URLs |
| [`gateway/overview.md`](skills/openui/references/gateway/overview.md) | Gateway capabilities, endpoints, API choice, prompt config, security, and BYOK |
| [`gateway/quickstart.md`](skills/openui/references/gateway/quickstart.md) | Gateway template scaffolding, authentication with the user, and the generated app |
| [`gateway/responses.md`](skills/openui/references/gateway/responses.md) | Responses history models, streaming, hosted tools, and app function tools |
| [`gateway/chat-completions.md`](skills/openui/references/gateway/chat-completions.md) | Chat Completions with app-owned history and function tools |
| [`gateway/conversations.md`](skills/openui/references/gateway/conversations.md) | Gateway threads and items, Chat Completions history storage, frontend tokens, and authorization |
| [`gateway/autofix.md`](skills/openui/references/gateway/autofix.md) | Repairing output from any provider with `@openuidev/server` or the Autofix API |
| [`gateway/migrate.md`](skills/openui/references/gateway/migrate.md) | Moving OpenAI-compatible and self-hosted OpenUI apps to Gateway, and migrating stored data |

## OpenUI Building Blocks

- **OpenUI Lang**: a language for AI-generated interfaces.
- **Runtime packages**: render those interfaces in React, Vue, Svelte, Angular, or the browser.
- **Component libraries**: the components the model can use.
- **Agent Interface**: an open-source chat app with conversation history, generated UI, and an artifact workspace.
- **OpenUI Gateway**: hosted model access with correction, fallbacks, managed conversations, hosted tools, and Autofix.
- **Reliability Monitoring**: find and inspect generation errors in production.

## Learn More

- [OpenUI documentation](https://www.openui.com/docs)
- [Gateway documentation](https://www.openui.com/docs/gateway)
- [Autofix](https://www.openui.com/docs/autofix)
- [Reliability Monitoring](https://www.openui.com/docs/reliability)
- [OpenUI source and examples](https://github.com/thesysdev/openui)
- [OpenUI Lang specification](https://www.openui.com/docs/openui-lang/specification-v05)
