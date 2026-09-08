# OpenUI Skill

Agent-ready guidance for building generative interfaces with [OpenUI](https://www.openui.com/). This repository contains one focused skill that helps AI coding assistants work with OpenUI Lang, the OpenUI runtimes, Agent Interface, and OpenUI Gateway.

The current docs distinguish Gateway (model access and correction) from Observability (runtime monitoring). Hosted-service guidance lives under `references/gateway/`. The CLI template remains `openui-cloud`, and existing Thesys SDK identifiers are unchanged.

## What the Skill Covers

- Scaffold new OpenUI Gateway applications by default with `@openuidev/cli`, while preserving an explicit self-hosted path.
- Stream and render OpenUI Lang in React, Vue, Svelte, and browser-based apps.
- Build custom component libraries with typed schemas, state, actions, queries, and mutations.
- Measure and improve generation reliability with repeated evaluations, DevTools, production observability, and runtime correction.
- Add `AgentInterface`, or keep an existing assistant-ui, CopilotKit, or custom chat surface and integrate only the OpenUI renderer.
- Choose between Responses and Chat Completions, optionally add Conversations persistence to Responses, and distinguish each framework's browser transport from its provider API.
- Keep the default starter's managed slides/reports; separately verify the historical standalone Artifact Chat Completions contract before new integrations.
- Use built-in or custom component libraries with managed Gateway generation, including Gateway BYOK.
- Migrate legacy JSON UI or self-hosted OpenUI implementations.
- Debug prompts, parsers, renderers, adapters, storage, theming, tools, and artifacts.

## Installation

```bash
npx skills add thesysdev/skills
```

Once installed, try prompts such as:

- “Create a streaming OpenUI dashboard from this API.”
- “Create a new OpenUI Gateway agent with LangGraph.”
- “Add Agent Interface to my existing Next.js app.”
- “Build a custom OpenUI component library for these domain objects.”
- “Move this OpenAI Chat Completions app to OpenUI Gateway without changing its history model.”
- “Migrate this self-hosted OpenUI chat to OpenUI Gateway.”

## Skill Contents

| Resource | Purpose |
| --- | --- |
| [`skills/openui/SKILL.md`](skills/openui/SKILL.md) | Core workflows, package guidance, OpenUI Lang rules, and verification steps |
| [`gateway/integration.md`](skills/openui/references/gateway/integration.md) | Shared Gateway routing, configuration, security, BYOK, compatibility, reliability, and verification |
| [`gateway/quickstart.md`](skills/openui/references/gateway/quickstart.md) | Gateway-first scaffolding, generated-template workflow, and launch verification |
| [`gateway/chat/api-selection.md`](skills/openui/references/gateway/chat/api-selection.md) | Selection between Responses and Chat Completions for conversational generation |
| [`gateway/chat/responses.md`](skills/openui/references/gateway/chat/responses.md) | Responses generation, history modes, streaming, hosted tools, and in-conversation artifacts |
| [`gateway/chat/chat-completions.md`](skills/openui/references/gateway/chat/chat-completions.md) | Chat Completions, app-owned history/storage, adapters, and function-tool runbook |
| [`gateway/chat/conversations.md`](skills/openui/references/gateway/chat/conversations.md) | Optional Responses persistence: threads, items, frontend tokens, identity, authorization, and browser storage |
| [`gateway/artifacts.md`](skills/openui/references/gateway/artifacts.md) | Version-sensitive standalone slide/report contract, availability checkpoint, viewers, and application-owned persistence |
| [`gateway/oss-migration.md`](skills/openui/references/gateway/oss-migration.md) | Migration runbook from self-hosted OpenUI to OpenUI Gateway |
| [`examples.md`](skills/openui/references/examples.md) | Complete first-party example catalog plus current existing-chat and runtime integration guides |
| [`build-component-library.md`](skills/openui/references/build-component-library.md) | Component definition, schema design, prompt/spec handoff, runtime wiring, and verification |
| [`open-ended-html.md`](skills/openui/references/open-ended-html.md) | Guidance for generated HTML, sandboxed apps, and open-ended UI |
| [`theme-provider.md`](skills/openui/references/theme-provider.md) | Theme ownership, tokens, light/dark mode, nested scopes, and portals |

## OpenUI Building Blocks

- **OpenUI Lang** — a compact, streaming-first language for model-generated interfaces.
- **Runtime packages** — framework-agnostic core plus React, Vue, Svelte, and browser renderers.
- **Component libraries** — built-in or custom components exposed to the model through typed schemas.
- **Agent Interface** — a complete chat application shell with pluggable model and storage backends.
- **OpenUI Gateway** — model access, eligible OpenUI Lang correction, provider fallbacks, and optional Responses persistence/hosted tools.
- **OpenUI Observability** — independent production runtime monitoring for Gateway or direct-provider generation.

## Learn More

- [OpenUI documentation](https://www.openui.com/docs)
- [Gateway documentation](https://www.openui.com/docs/gateway)
- [Observability installation](https://www.openui.com/docs/observability/installation)
- [OpenUI source and examples](https://github.com/thesysdev/openui)
- [OpenUI Lang specification](https://www.openui.com/docs/openui-lang/specification-v05)
