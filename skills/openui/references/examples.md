# Choose a First-Party OpenUI Example

Read this reference when an existing example could provide the integration seam, component-library pattern, or verification contract for the task. Examples are standalone reference implementations, not CLI starter templates. Inspect the selected example's README, manifest, lockfile, and current source before copying it because examples and published package versions can move independently.

The authoritative catalog lives at `https://github.com/thesysdev/openui/blob/main/examples/README.md`. If that catalog differs from this reference, use the current repository.

## Agent Frameworks

| Example | Path | Use it for |
| --- | --- | --- |
| Google ADK | `examples/agent-frameworks/google-adk` | Google ADK TypeScript agents streaming OpenUI Lang to a Next.js client |
| LangGraph Platform | `examples/agent-frameworks/langgraph-platform` | DeepAgents on LangGraph Platform through the OpenUI LangChain adapter |
| Mastra | `examples/agent-frameworks/mastra` | Mastra agents connected through AG-UI |
| Vercel AI SDK | `examples/agent-frameworks/vercel-ai-sdk` | `AgentInterface` over a Vercel AI SDK `streamText` backend |
| Vercel Eve | `examples/agent-frameworks/vercel-eve` | Eve agents rendered through Agent Interface |

## App Frameworks

| Example | Path | Use it for |
| --- | --- | --- |
| FastAPI | `examples/app-frameworks/fastapi` | Python FastAPI streaming with a React OpenUI client; the JavaScript app is under `frontend/` |
| React Native | `examples/app-frameworks/react-native` | Expo-native OpenUI rendering with separate `backend/` and `chat-app/` applications |
| Svelte | `examples/app-frameworks/svelte` | OpenUI Lang parsing and rendering in SvelteKit |
| Vue | `examples/app-frameworks/vue` | OpenUI Lang parsing and rendering in Nuxt and Vue |

## Design Systems

| Example | Path | Use it for |
| --- | --- | --- |
| Material UI | `examples/design-systems/material-ui` | Mapping a broad Material UI design system into an OpenUI component library |
| shadcn/ui | `examples/design-systems/shadcn` | Mapping shadcn/ui components into an OpenUI component library |

## Coding Harnesses

| Example | Path | Use it for |
| --- | --- | --- |
| Grok Build | `examples/harnesses/grok-build` | Presenting Grok Build sessions and tool activity in Agent Interface |
| Pi | `examples/harnesses/pi` | Streaming a Pi coding-agent session into Agent Interface |

## Specialized Examples

| Example | Path | Use it for |
| --- | --- | --- |
| Handsontable | `examples/miscellaneous/handsontable` | Spreadsheet-style generated interfaces backed by Handsontable |
| HTML artifact | `examples/miscellaneous/html-artifact` | Sandboxed open-ended HTML artifacts |
| React Email | `examples/miscellaneous/react-email` | Generating and previewing emails with the OpenUI React Email library |
| Supabase | `examples/miscellaneous/supabase` | Application-owned persisted OpenUI conversations and threads with Supabase |

## Use an Example Safely

1. Choose the example for its primary integration seam, not because it happens to share one secondary dependency.
2. Read its README and key files before editing the user's application.
3. Preserve the user's framework, provider protocol, storage owner, package manager, authentication, and design system unless the user requested a migration.
4. Copy the smallest relevant pattern rather than replacing a working application with the example.
5. Install and verify inside the example directory. Repository-root installation does not install every standalone example.
6. Run the example's credential-free `verify` script when available, then run the host application's own checks.

Do not assume every example is an OpenUI Cloud Responses application. The catalog includes self-hosted, framework-owned, design-system, storage, and harness examples. Inspect its actual transport and state model before reusing it.
