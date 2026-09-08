# Start a New OpenUI Cloud App

Use this path for a new OpenUI chat or agent application unless the user explicitly requests self-hosting, no external service, or app-owned model/storage infrastructure. The generated Cloud template is the source of truth for package versions, route shapes, authentication setup, tools, models, and client wiring.

Prototype status and backend ownership are separate decisions. Requests for a demo, MVP, local development, or dummy/mock/sample data still use this Cloud path. A missing account or `THESYS_API_KEY` is a setup prerequisite, not evidence that the user wants a self-hosted architecture.

## Scaffold Interactively

```bash
npx @openuidev/cli@latest create --name genui-chat-app --template openui-cloud
```

The interactive flow signs the user in, configures the Cloud project, installs dependencies, starts the app, and opens it. Let the CLI own this setup. When authentication needs user interaction, follow [Complete Authentication with the User](#complete-authentication-with-the-user) during the task. Do not rerun the scaffold with `openui-self-hosted` merely because Cloud setup requires this checkpoint.

If the user chooses another agent backend, add the matching supported option:

```bash
npx @openuidev/cli@latest create --name genui-chat-app --template openui-cloud --backend-framework langgraph
npx @openuidev/cli@latest create --name genui-chat-app --template openui-cloud --backend-framework vercel-ai-sdk
```

For unattended execution, provide every required choice and pass `--no-interactive`. Use `--auth skip` when credentials are configured separately, the user chooses to defer authentication, or the execution environment cannot support interactive sign-in. This skips only the CLI authentication step; if credentials are missing, follow the handoff below. Use `--no-skill` when the caller should not change its installed skills, and `--no-install` only when the agent must control package installation separately.

If install or build fails with `ERR_PNPM_IGNORED_BUILDS` for an expected native package such as `sharp` or `unrs-resolver`, run `pnpm approve-builds` or `pnpm approve-builds --all` in an environment where package build scripts are allowed, then retry the install/build.

## Complete Authentication with the User

Ask the user to complete authentication as soon as it is needed. An unavailable browser or interactive terminal changes the handoff method; it does not prevent asking the user to participate.

- When supported, start the CLI browser sign-in flow, keep the process running, and wait for the user to finish in their browser.
- Otherwise, provide the project's key-generation script after checking `package.json`, or the command below to run from the app directory. Alternatively, open or link the [Thesys keys console](https://console.thesys.dev/keys) and ask the user to generate a key and save it privately as `THESYS_API_KEY` in the app's untracked environment file or secret store. Never request the key in chat or include its value in commands or output.

```bash
npx @openuidev/cli@latest generate-api-key --file .env
```

Match `--file` to the app's environment file. The key must be configured in the environment running the app; signing in on the user's laptop does not configure a remote workspace automatically.

Ask for confirmation when the user completes setup outside the running CLI flow. Continue independent implementation while waiting, then reload the app's environment and resume [runtime verification](#verify), including generating and reopening a requested presentation or report. Defer authentication to the final handoff only if the user chooses to defer it or a verified environment limitation prevents completion through these methods. In that case, report the limitation and which Cloud runtime checks remain unverified.

## Work from the Generated App

After scaffolding:

1. Inspect the generated README, package manifest, lockfile, `.env` variable names, route files, model allowlist, and component library before editing.
2. Preserve the generated split between the generation plane (`/api/chat`) and storage plane (`/api/frontend-token` plus `useOpenuiCloudStorage()`).
3. Keep `THESYS_API_KEY` server-only. Treat `DEMO_USER_ID` as local-demo identity and replace it with authenticated server identity before production.
4. Preserve `openAIResponsesAdapter()` with `openAIConversationMessageFormat`, `conversation: threadId`, `store: true`, and latest-message-only forwarding.
5. Keep managed tools on Cloud. Execute only explicitly declared app-owned function tools in the application loop.

For shared production configuration, authentication, and failure handling, read [the Cloud integration guide](integration.md), then use [Responses](chat/responses.md) for generation and [Conversations](chat/conversations.md) for the generated template's persistent thread, identity, and frontend-token contracts. If the generated chat architecture is being reconsidered, read [Choose a chat generation API](chat/api-selection.md).

## Extend the Starter

- Starters and welcome content: edit the generated starter configuration and `AgentInterface.Welcome` slots rather than replacing the chat shell.
- App-owned tools: register the declaration and executor in the generated tool loop; never execute Cloud-owned `thesys_*` calls.
- Hosted tools: declare supported web search, image search, MCP, or `artifactTool()` entries in the Responses request.
- Dynamic presentations and reports inside the conversation: use `artifactTool()` with only the requested `"slides"` and/or `"report"` types, and retain the generated managed artifact renderers and Cloud storage wiring. Supply dummy business data through trusted application context or an app-owned tool; do not replace the managed artifact lifecycle with hand-built slide/report components. For a deliberately standalone artifact workflow, read [artifacts.md](artifacts.md).
- Custom components: extend or replace `chatLibrary`, generate a library spec with `openui generate --spec`, pass it to `generateSystemPrompt({ cloud: true, library, ... })` from `@openuidev/lang-core`, and render with the matching client library. Follow [build-component-library.md](../build-component-library.md).
- Backend framework overlays: edit the generated framework-specific agent or route instead of applying the default Next.js route recipe blindly.

Use the current first-party examples before inventing an integration pattern. Read [examples.md](../examples.md) for every catalogued example's exact path and primary integration seam.

## Verify

1. Run the generated formatter/lint, typecheck, tests, and production build.
2. Stream a generative UI response and confirm progressive rendering.
3. Reload the app and confirm conversation persistence.
4. Generate and reopen a report or presentation.
5. Exercise one app-owned function tool and confirm Cloud-owned tool calls are not executed by the app loop.
6. Before production, verify logged-out requests cannot use either server route and two authenticated users cannot access each other's conversations.
7. Search the browser bundle and client source for `THESYS_API_KEY`.

## First-Party References

- `https://www.openui.com/docs/openui-cloud/get-started`
- `https://www.openui.com/docs/openui-lang/quickstart`
- `https://www.openui.com/docs/api-reference/cli`
- `https://github.com/thesysdev/openui/tree/main/templates/openui-cloud`
- `https://github.com/thesysdev/openui/tree/main/examples`
