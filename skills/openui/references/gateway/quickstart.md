# Start a New Gateway App

The Gateway template is the default for every new OpenUI chat or agent app, including prototypes, demos, and apps with sample data. Use the [self-hosted template](../self-hosted.md) only when the user asks to self-host, wants no third-party service, or has a provider or data requirement Gateway does not support.

## Scaffold

Use Node.js 20 or later.

```bash
npx @openuidev/cli@latest create --name my-app --template openui-cloud
```

The interactive flow signs the user in, writes `THESYS_API_KEY` to the app's env file, installs dependencies, and offers to start the dev server. When sign-in needs the user, follow [Complete Authentication with the User](#complete-authentication-with-the-user).

| Flag | Effect |
| --- | --- |
| `--backend-framework <name>` | `default` (a Responses route using the OpenAI SDK), `langgraph`, `vercel-ai-sdk`, or `vercel-eve` |
| `--example <folder>` | Scaffold an [example](../examples.md) by folder name, such as `shadcn` or `mastra`; cannot be combined with `--template` or `--backend-framework` |
| `--no-interactive` | Fail instead of prompting; supply every required choice |
| `--auth skip` | Skip CLI sign-in when credentials are configured separately or sign-in cannot run here |
| `-i, --immediate` / `--no-immediate` | Start the dev server after install, or install and exit; use one, not both |
| `--no-install` | Scaffold without installing dependencies |
| `--skill` / `--no-skill` | Install or skip the OpenUI agent skill; non-interactive runs skip it unless `--skill` is passed |
| `--agent-name <slug>` | Identify the calling coding agent in CLI telemetry |

The CLI also accepts `--api-key`, but a coding agent should never put a key value in a command. Use sign-in or `generate-api-key` instead.

If pnpm reports `ERR_PNPM_IGNORED_BUILDS` for a native package such as `sharp` or `unrs-resolver`, run `pnpm approve-builds` and install again.

## Complete Authentication with the User

Gateway sign-in is a step the user completes during the task. It is not a reason to switch to the self-hosted template or to hand-build a Gateway feature.

- **Browser available:** run the CLI sign-in, leave the process running, and wait for the user to finish in their browser.
- **No browser or TTY here:** ask the user to run the key command from the app directory on a machine where they can sign in, or to create a key in the [Thesys Console](https://console.thesys.dev/keys) and save it as `THESYS_API_KEY` in the app's untracked env file or secret store:

```bash
npx @openuidev/cli@latest generate-api-key --file .env.local
```

Match `--file` to the app's env file. The key must exist in the environment that runs the app; signing in on the user's laptop does not configure a remote workspace. Never ask for the key in chat or print it. Keep working on anything that does not need the key while waiting, then reload the environment and run the [checks below](#verify). If the user chooses to defer sign-in, report which checks still need a key.

## Know the Generated App

For the default backend:

| File | Role |
| --- | --- |
| `src/app/api/chat/route.ts` | Proxies Responses to `https://api.thesys.dev/v1/embed` with `conversation: threadId` and `store: true`, forwarding only the latest message. Declares hosted tools and runs app function tools through `src/lib/tool-loop.ts`. |
| `src/app/api/frontend-token/route.ts` | Mints the browser's scoped storage token. It uses `DEMO_USER_ID` as the user; replace that with the authenticated user's id before real users arrive. |
| `src/components/cloud-chat.tsx` | `AgentInterface` with `openAIResponsesAdapter()`, `openAIConversationMessageFormat`, `useOpenuiCloudStorage`, `openuiLibrary`, and a `ModelSwitcher` |
| `src/lib/models.tsx` | Server-side model allowlist; the browser sends a model choice that the route validates |
| `src/generated/spec.json` | Serialized library spec passed to `generateSystemPrompt({ cloud: true, library })` |

Framework overlays change the route, and Eve uses session endpoints instead of `/api/chat`. Read the generated README and route before editing, and identify the provider call, browser stream, and persistence path separately. A framework can send its own stream format to the browser even when it calls Responses on the server; choose the client adapter from [Agent Interface stream wiring](../agent-interface.md#match-the-browser-stream).

## Extend the Starter

- **App function tools:** register the declaration and executor in the tool loop ([Responses function tools](responses.md#run-app-function-tools)).
- **Hosted tools:** declare web search, image search, or MCP in the Responses request ([hosted tools](responses.md#add-hosted-tools)).
- **Custom components:** extend or replace `openuiLibrary`, regenerate the spec, and render the same library on the client ([build-component-library.md](../build-component-library.md)).
- **Chat shell, storage, and navigation:** see [agent-interface.md](../agent-interface.md).
- **Artifacts:** see [artifacts.md](../artifacts.md).

## Verify

1. Run the generated lint, typecheck, and production build.
2. Stream a generated-UI response and confirm progressive rendering.
3. Reload and confirm the thread persists.
4. Run one app function tool and confirm the app never executes Gateway-owned calls.
5. Search client source and the built bundle for `THESYS_API_KEY`.
6. Deploy a preview with [deploy.md](../deploy.md).

## First-Party References

- `https://www.openui.com/docs/agent/getting-started/quickstart`
- `https://www.openui.com/docs/api-reference/cli`
- `https://github.com/thesysdev/openui/tree/main/templates/openui-cloud`
