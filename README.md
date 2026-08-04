# OpenUI Skill

Agent-ready guidance for building generative interfaces with [OpenUI](https://www.openui.com/). This repository contains one focused skill that helps AI coding assistants work with OpenUI Lang, the OpenUI runtimes, Agent Interface, and OpenUI Cloud.

## What the Skill Covers

- Scaffold new OpenUI applications with `@openuidev/cli`.
- Stream and render OpenUI Lang in React, Vue, Svelte, and browser-based apps.
- Build custom component libraries with typed schemas, state, actions, queries, and mutations.
- Add `AgentInterface` to new or existing chat applications.
- Choose between self-hosted OpenUI and OpenUI Cloud, then integrate the selected backend safely.
- Migrate legacy JSON UI or self-hosted OpenUI implementations.
- Debug prompts, parsers, renderers, adapters, storage, theming, tools, and artifacts.

## Installation

```bash
npx skills add thesysdev/skills
```

Once installed, try prompts such as:

- “Create a streaming OpenUI dashboard from this API.”
- “Add Agent Interface to my existing Next.js app.”
- “Build a custom OpenUI component library for these domain objects.”
- “Migrate this self-hosted OpenUI chat to OpenUI Cloud.”

## Skill Contents

| Resource | Purpose |
| --- | --- |
| [`skills/openui/SKILL.md`](skills/openui/SKILL.md) | Core workflows, package guidance, OpenUI Lang rules, and verification steps |
| [`cloud-integration.md`](skills/openui/references/cloud-integration.md) | Production-minded OpenUI Cloud integration runbook |
| [`open-ended-html.md`](skills/openui/references/open-ended-html.md) | Guidance for generated HTML, sandboxed apps, and open-ended UI |
| [`oss-to-cloud-migration.md`](skills/openui/references/oss-to-cloud-migration.md) | Migration runbook from self-hosted OpenUI to OpenUI Cloud |

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
