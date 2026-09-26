# Choose a First-Party OpenUI Example

Read this guide when an example could supply the integration pattern, component-library approach, or verification steps for a task. Examples are standalone reference apps, and they use a mix of Gateway, self-hosted, and framework-owned backends. Read the chosen example's README, manifest, and source before copying from it.

The authoritative catalog is `https://github.com/thesysdev/openui/blob/main/examples/README.md`; when it differs from this list, follow the repository.

To start a new app from an example, pass its folder name to the CLI:

```bash
npx @openuidev/cli@latest create --name my-app --example shadcn
```

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
| Angular | `examples/app-frameworks/angular` | An NG-ZORRO X chat interface with the Angular renderer from `@openuidev/angular-lang` |
| FastAPI | `examples/app-frameworks/fastapi` | Python FastAPI streaming with a React OpenUI client in `frontend/` |
| React Native | `examples/app-frameworks/react-native` | An Expo client rendering native components, with separate `backend/` and `chat-app/` apps |
| Svelte | `examples/app-frameworks/svelte` | OpenUI Lang parsing and rendering in SvelteKit |
| Vue | `examples/app-frameworks/vue` | OpenUI Lang parsing and rendering in Nuxt and Vue |

## Cookbooks

| Example | Path | Use it for |
| --- | --- | --- |
| Conversational analytics | `examples/cookbooks/conversational-analytics` | A complete Gateway app with streamed charts and visible tool calls, with a [step-by-step tutorial](https://www.openui.com/docs/cookbooks/conversational-analytics) |

## Design Systems

| Example | Path | Use it for |
| --- | --- | --- |
| Material UI | `examples/design-systems/material-ui` | Mapping Material UI into an OpenUI component library |
| shadcn/ui | `examples/design-systems/shadcn` | Mapping shadcn/ui into an OpenUI component library |

## Coding Harnesses

| Example | Path | Use it for |
| --- | --- | --- |
| Grok Build | `examples/harnesses/grok-build` | Grok Build sessions and tool activity in Agent Interface |
| Pi | `examples/harnesses/pi` | A Pi coding-agent session streamed into Agent Interface |

## Specialized Examples

| Example | Path | Use it for |
| --- | --- | --- |
| Autofix | `examples/miscellaneous/autofix` | Direct OpenAI generation repaired by `@openuidev/server` Autofix |
| Handsontable | `examples/miscellaneous/handsontable` | Spreadsheet-style generated interfaces backed by Handsontable |
| HTML artifact | `examples/miscellaneous/html-artifact` | Sandboxed open-ended HTML artifacts |
| React Email | `examples/miscellaneous/react-email` | Generating and previewing emails with the React Email library |
| Supabase | `examples/miscellaneous/supabase` | App-owned persistence of conversations and threads in Supabase |

## Existing Chat UI Integration Guides

These docs pages keep the host's chat shell and runtime and add OpenUI rendering to one slot:

| Existing surface | Guide | Integration point |
| --- | --- | --- |
| Agent backend | [Backend setup](https://www.openui.com/docs/build-agents/backend-setup) | Generate instructions from the matching library spec; Gateway or [Autofix](gateway/autofix.md) adds correction |
| assistant-ui | [assistant-ui](https://www.openui.com/docs/build-agents/assistant-ui) | Replace the `MessagePrimitive.Parts` text renderer for assistant OpenUI Lang |
| assistant-ui tool output | [Package API](https://www.openui.com/docs/api-reference/assistant-ui) | `@openuidev/assistant-ui` renders OpenUI tool calls, a separate output mode |
| CopilotKit | [CopilotKit](https://www.openui.com/docs/build-agents/copilotkit) | The assistant message's `markdownRenderer` slot; match the installed CopilotKit API |
| Custom message list | [Custom Chat UI](https://www.openui.com/docs/build-agents/custom-chat-ui) | Render accumulated OpenUI Lang inside the existing assistant message |

Agent runtime guides cover [LangGraph Platform](https://www.openui.com/docs/agent/agent-runtimes/langgraph-platform), [Vercel AI SDK](https://www.openui.com/docs/agent/agent-runtimes/vercel-ai-sdk), [Vercel Eve](https://www.openui.com/docs/agent/agent-runtimes/vercel-eve), and [Pi](https://www.openui.com/docs/agent/agent-runtimes/pi). A runtime example and the matching CLI overlay can use different transports and storage, so inspect the one being copied.

## Use an Example Safely

1. Choose the example for its main integration point, not for a secondary dependency it happens to share.
2. Read its README and key files before editing the user's app.
3. Copy the smallest relevant pattern into the app instead of replacing working code with the example.
4. Install and verify inside the example's own directory. In the upstream monorepo, `pnpm install --ignore-workspace` keeps pnpm from using the repository workspace; `npm install` and `bun install` also work.
5. Run the example's credential-free `verify` script when it has one, then the host app's own checks.
