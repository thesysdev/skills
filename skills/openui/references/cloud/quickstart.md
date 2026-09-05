# Start a New OpenUI Cloud App

Use this path for a new production-oriented OpenUI agent application when the user has not explicitly chosen an app-owned model or storage layer. The generated Cloud template is the source of truth for package versions, route shapes, authentication setup, tools, models, and client wiring.

## Scaffold Interactively

```bash
npx @openuidev/cli@latest create --name genui-chat-app --template openui-cloud
```

The interactive flow signs the user in, configures the Cloud project, installs dependencies, starts the app, and opens it. Let the CLI own this setup. Never ask the user to paste an API key into chat or add `--api-key` with a literal value to generated commands.

If the user chooses another agent backend, add the matching supported option:

```bash
npx @openuidev/cli@latest create --name genui-chat-app --template openui-cloud --backend-framework langgraph
npx @openuidev/cli@latest create --name genui-chat-app --template openui-cloud --backend-framework vercel-ai-sdk
```

For unattended execution, provide every required choice and pass `--no-interactive`. Use `--auth skip` only when authentication is intentionally handled outside the CLI and the Cloud credential is already configured in an approved secret store. Use `--no-skill` when the caller should not change its installed skills, and `--no-install` only when the agent must control package installation separately.

If install or build fails with `ERR_PNPM_IGNORED_BUILDS` for an expected native package such as `sharp` or `unrs-resolver`, run `pnpm approve-builds` or `pnpm approve-builds --all` in an environment where package build scripts are allowed, then retry the install/build.

## Work from the Generated App

After scaffolding:

1. Inspect the generated README, package manifest, lockfile, `.env` variable names, route files, model allowlist, and component library before editing.
2. Preserve the generated split between the generation plane (`/api/chat`) and storage plane (`/api/frontend-token` plus `useOpenuiCloudStorage()`).
3. Keep `THESYS_API_KEY` server-only. Treat `DEMO_USER_ID` as local-demo identity and replace it with authenticated server identity before production.
4. Preserve `openAIResponsesAdapter()` with `openAIConversationMessageFormat`, `conversation: threadId`, `store: true`, and latest-message-only forwarding.
5. Keep managed tools on Cloud. Execute only explicitly declared app-owned function tools in the application loop.

For shared production configuration, authentication, and failure handling, read [existing-project.md](existing-project.md), then use [responses.md](responses.md) for generation and [conversations.md](conversations.md) for the generated template's persistent thread, identity, and frontend-token contracts. For endpoint and state-model selection, read [api-selection.md](api-selection.md).

## Extend the Starter

- Starters and welcome content: edit the generated starter configuration and `AgentInterface.Welcome` slots rather than replacing the chat shell.
- App-owned tools: register the declaration and executor in the generated tool loop; never execute Cloud-owned `thesys_*` calls.
- Hosted tools: declare supported web search, image search, MCP, or `artifactTool()` entries in the Responses request.
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
