# Start a New OpenUI Gateway App

Use this path for a new OpenUI chat or agent application unless the user explicitly requests self-hosting, no external service, or app-owned model/storage infrastructure. Inspect the generated Gateway template for package versions, route shapes, authentication setup, tools, models, and client wiring.

Use Node.js 20 or later. Coding agents should append `--agent-name` with their own stable product slug to CLI commands; human-run commands can omit it.

Prototype status and backend ownership are separate decisions. Requests for a demo, MVP, local development, or dummy/mock/sample data still use this Gateway path. A missing account or `THESYS_API_KEY` is a setup prerequisite, not evidence that the user wants a self-hosted architecture.

## Scaffold Interactively

```bash
npx @openuidev/cli@latest create --name genui-chat-app --template openui-cloud
```

The interactive flow handles sign-in, project configuration, and dependency installation, then asks whether to start the app. Use `--immediate` to start it or `--no-immediate` to install and exit; do not pass both. When authentication needs user interaction, follow [Complete Authentication with the User](#complete-authentication-with-the-user) during the task. Do not rerun the scaffold with `openui-self-hosted` merely because Gateway setup requires this checkpoint.

If the user chooses another agent backend, add the matching supported option:

```bash
npx @openuidev/cli@latest create --name genui-chat-app --template openui-cloud --backend-framework langgraph
npx @openuidev/cli@latest create --name genui-chat-app --template openui-cloud --backend-framework vercel-ai-sdk
npx @openuidev/cli@latest create --name genui-chat-app --template openui-cloud --backend-framework vercel-eve
```

For unattended execution, provide every required choice and pass `--no-interactive`. Use `--auth skip` when credentials are configured separately, the user chooses to defer authentication, or the execution environment cannot support interactive sign-in. This skips only the CLI authentication step; if credentials are missing, follow the handoff below. Use `--no-skill` when the caller should not change its installed skills, and `--no-install` only when the agent must control package installation separately.

Non-interactive mode does not start the development server unless `--immediate` is set. Use the requested backend framework and inspect its tool transport; do not assume every overlay includes the default's hosted tools.

If install or build fails with `ERR_PNPM_IGNORED_BUILDS` for an expected native package such as `sharp` or `unrs-resolver`, run `pnpm approve-builds` or `pnpm approve-builds --all` in an environment where package build scripts are allowed, then retry the install/build.

## Complete Authentication with the User

Ask the user to complete authentication as soon as it is needed. An unavailable browser or interactive terminal changes the handoff method; it does not prevent asking the user to participate.

- When supported, start the CLI browser sign-in flow, keep the process running, and wait for the user to finish in their browser.
- Otherwise, provide the project's key-generation script after checking `package.json`, or the command below to run from the app directory. Alternatively, open or link the [Thesys keys console](https://console.thesys.dev/keys) and ask the user to generate a key and save it privately as `THESYS_API_KEY` in the app's untracked environment file or secret store. Never request the key in chat or include its value in commands or output.

```bash
npx @openuidev/cli@latest generate-api-key --file .env
```

Match `--file` to the app's environment file. The key must be configured in the environment running the app; signing in on the user's laptop does not configure a remote workspace automatically.

Ask for confirmation when the user completes setup outside the running CLI flow. Continue independent implementation while waiting, then reload the app's environment and resume [runtime verification](#verify). Defer authentication to the final handoff only if the user chooses to defer it or a verified environment limitation prevents completion through these methods. In that case, report the limitation and which Gateway runtime checks remain unverified.

## Work from the Generated App

After scaffolding:

1. Inspect the generated README, package manifest, lockfile, `.env` variable names, route files, model allowlist, and component library before editing.
2. Identify the actual generation transport and storage paths separately. `/api/chat` is not universal; Eve uses session endpoints.
3. Keep `THESYS_API_KEY` server-only. Treat `DEMO_USER_ID` as local-demo identity and replace it with authenticated server identity before production.
4. For the default backend, preserve `conversation: threadId`, `store: true`, and latest-message-only forwarding. For a framework overlay, inspect its provider call and persistence contract.
5. Keep hosted tools on Gateway. Execute only explicitly declared app-owned function tools in the application loop.

Inspect the generated backend's provider call and persistence separately. A framework may expose its own browser protocol even when it calls Chat Completions or Responses on the server. Follow [Agent Interface stream wiring](../agent-interface.md#match-the-browser-stream) for the adapter and message-format mapping.

A configured storage client alone does not prove that the generation route persists replies. Trace the write path and verify reloads; a framework's storage layer does not add Responses parameters to Chat Completions calls.

Read [the Gateway integration guide](integration.md) for shared requirements, then choose [Responses](chat/responses.md) or [Chat Completions](chat/chat-completions.md) from the actual provider call. Read [Conversations](chat/conversations.md) when Gateway storage is used. Framework-to-browser streams need the framework adapter, independently of that provider choice.

## Extend the Starter

Read [Agent Interface](../agent-interface.md) for the chat shell, backend channels, message rendering, layout, and navigation.

- App-owned tools: register the declaration and executor in the generated tool loop; never execute Gateway-owned `thesys_*` calls.
- Hosted tools: declare supported web search, image search, or MCP entries in the Responses request.
- Custom components: extend `openuiChatLibrary` or use a custom library, generate a library spec with `openui generate --spec`, pass it to `generateSystemPrompt({ cloud: true, library, ... })` from `@openuidev/lang-core`, and render with the matching client library. Follow [build-component-library.md](../build-component-library.md).
- Backend framework overlays: edit the generated framework-specific agent or route instead of applying the default Next.js route recipe blindly.

Use the current first-party examples before inventing an integration pattern. Read [examples.md](../examples.md) for every catalogued example's exact path and primary integration seam.

## Verify

1. Run the generated formatter/lint, typecheck, tests, and production build.
2. Stream a generative UI response and confirm progressive rendering.
3. Reload the app and confirm conversation persistence.
4. Exercise one app-owned function tool and confirm Gateway-owned tool calls are not executed by the app loop.
5. Before production, protect all generation, token, and framework-session endpoints; verify logged-out requests and cross-user conversation/session access are rejected.
6. Search the browser bundle and client source for `THESYS_API_KEY`.

## First-Party References

- `https://www.openui.com/docs/getting-started`
- `https://www.openui.com/docs/openui-lang/quickstart`
- `https://www.openui.com/docs/api-reference/cli`
- `https://github.com/thesysdev/openui/tree/main/templates/openui-cloud`
- `https://github.com/thesysdev/openui/tree/main/examples`
