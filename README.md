# Custom Skills

This repository contains custom skills for AI coding assistants.

## Available Skills

### [Thesys C1 Generative UI](/thesys-c1-skill/)

A comprehensive skill for developing applications with the **Thesys C1 API** and **GenUI SDK**—enabling AI-powered Generative UI applications that dynamically create interactive interfaces from natural language prompts.

**What's Included:**

| File | Description |
|------|-------------|
| [`SKILL.md`](/skills/thesys-c1-skill/SKILL.md) | Main skill file with quickstart and core concepts |
| [`conversational-ui-and-persistence.md`](/skills/thesys-c1-skill/conversational-ui-and-persistence.md) | State management & database persistence |
| [`custom-actions-components-thinking-states.md`](/skills/thesys-c1-skill/custom-actions-components-thinking-states.md) | Custom actions, components & loading states |
| [`artifacts.md`](/skills/thesys-c1-skill/artifacts.md) | Generating reports, presentations & documents |
| [`customizations-and-styling.md`](/skills/thesys-c1-skill/customizations-and-styling.md) | Theming, charts & CSS overrides |
| [`migration.md`](/skills/thesys-c1-skill/migration.md) | Migrating from text-based LLM apps |

**Key Capabilities:**
- Interactive UI generation (forms, charts, tables, buttons)
- Real-time streaming with progressive rendering
- OpenAI-compatible API (drop-in replacement)
- State persistence across sessions
- Custom actions and components
- Artifact generation (slides, reports)
- Full theming and styling control

## Installation

```bash
npx skills add thesysdev/skills
```

### Compatible Tools

- [Cursor](https://cursor.com)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Codex](https://chatgpt.com/features/codex)
- [Antigravity](https://antigravity.dev)
- [Gemini CLI](https://geminicli.com/)
- [OpenCode](https://opencode.ai)
