# OpenUI Skill

Agent-ready guidance for building generative interfaces with [OpenUI](https://www.openui.com/). This repository contains one focused skill that helps AI coding assistants work with OpenUI Lang, the OpenUI runtimes, Agent Interface, and OpenUI Cloud.

## What the Skill Covers

- Scaffold new OpenUI Cloud applications by default with `@openuidev/cli`, while preserving an explicit self-hosted path.
- Stream and render OpenUI Lang in React, Vue, Svelte, and browser-based apps.
- Build custom component libraries with typed schemas, state, actions, queries, and mutations.
- Measure and improve generation reliability with repeated evaluations, DevTools, production observability, and runtime correction.
- Add `AgentInterface` to new or existing chat applications.
- Choose among the OpenUI Cloud Responses, Embed Chat Completions, Artifact Chat Completions, and Conversations APIs.
- Use built-in or custom component libraries with managed Cloud generation, including Cloud BYOK.
- Migrate legacy JSON UI or self-hosted OpenUI implementations.
- Debug prompts, parsers, renderers, adapters, storage, theming, tools, and artifacts.

## Installation

```bash
npx skills add thesysdev/skills
```

Once installed, try prompts such as:

- “Create a streaming OpenUI dashboard from this API.”
- “Create a new OpenUI Cloud agent with LangGraph.”
- “Add Agent Interface to my existing Next.js app.”
- “Build a custom OpenUI component library for these domain objects.”
- “Move this OpenAI Chat Completions app to OpenUI Cloud without changing its history model.”
- “Migrate this self-hosted OpenUI chat to OpenUI Cloud.”

## Skill Contents

| Resource | Purpose |
| --- | --- |
| [`skills/openui/SKILL.md`](skills/openui/SKILL.md) | Core workflows, package guidance, OpenUI Lang rules, and verification steps |
| [`cloud/quickstart.md`](skills/openui/references/cloud/quickstart.md) | Cloud-first scaffolding, generated-template workflow, and launch verification |
| [`examples.md`](skills/openui/references/examples.md) | Complete current first-party example catalog with exact paths and integration seams |
| [`build-component-library.md`](skills/openui/references/build-component-library.md) | Component definition, schema design, prompt/spec handoff, runtime wiring, and verification |
| [`cloud/api-selection.md`](skills/openui/references/cloud/api-selection.md) | Cloud endpoint, state model, adapter, and prompt-mode selection |
| [`cloud/existing-project.md`](skills/openui/references/cloud/existing-project.md) | Shared brownfield Cloud discovery, configuration, security, BYOK, and protocol routing |
| [`cloud/responses.md`](skills/openui/references/cloud/responses.md) | Responses generation, history modes, streaming, hosted tools, and artifacts |
| [`cloud/chat-completions.md`](skills/openui/references/cloud/chat-completions.md) | Existing Chat Completions, app-owned history/storage, adapters, and function-tool runbook |
| [`cloud/artifacts.md`](skills/openui/references/cloud/artifacts.md) | Standalone slide/report generation, explicit edits, rendering, and application-owned persistence |
| [`cloud/conversations.md`](skills/openui/references/cloud/conversations.md) | Persistent Responses threads, items, frontend tokens, identity, authorization, and browser storage |
| [`open-ended-html.md`](skills/openui/references/open-ended-html.md) | Guidance for generated HTML, sandboxed apps, and open-ended UI |
| [`cloud/oss-migration.md`](skills/openui/references/cloud/oss-migration.md) | Migration runbook from self-hosted OpenUI to OpenUI Cloud |

## OpenUI Building Blocks

- **OpenUI Lang** — a compact, streaming-first language for model-generated interfaces.
- **Runtime packages** — framework-agnostic core plus React, Vue, Svelte, and browser renderers.
- **Component libraries** — built-in or custom components exposed to the model through typed schemas.
- **Agent Interface** — a complete chat application shell with pluggable model and storage backends.
- **OpenUI Cloud** — managed generation, persistence, tools, artifacts, and production infrastructure.

## Learn More

- [OpenUI documentation](https://www.openui.com/docs)
- [OpenUI source and examples](https://github.com/thesysdev/openui)
- [OpenUI Lang specification](https://www.openui.com/docs/openui-lang/specification-v05)
