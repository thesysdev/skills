# Build a new generative UI app

Use this when the user wants a new OpenUI app, agent, or chat. Scaffold it with the CLI, have the user add an API key, customize it, run it, then deploy.

## 1. Choose the starting point

Read the example catalog at https://github.com/thesysdev/openui/blob/main/examples/examples.json. Each entry has a `path` and a `description`.

- **An example matches the request** (a framework, design system, platform, or use case it names): scaffold that example. Its name is the last segment of its `path`, for example `shadcn` for `design-systems/shadcn` or `fastapi` for `app-frameworks/fastapi`.

  ```bash
  npx @openuidev/cli@latest create --name my-app --example shadcn
  ```

- **No example matches:** scaffold the default agent and build the rest yourself. Choose the backend framework the user asked for; otherwise keep the default, `langgraph`.

  ```bash
  npx @openuidev/cli@latest create --name my-app
  npx @openuidev/cli@latest create --name my-app --backend-framework vercel-ai-sdk
  npx @openuidev/cli@latest create --name my-app --backend-framework vercel-eve
  ```

`--example` and `--backend-framework` cannot be combined. An existing `--name` directory is an error.

## 2. Run `create`

When a coding agent runs `create`, it never prompts. It takes the defaults for anything not passed, installs dependencies without starting the dev server, skips sign-in, and prints the next commands. Other options:

- `--no-install`: scaffold without installing dependencies.
- `--immediate`: start the dev server after installing.
- `--no-skill`: do not install the OpenUI skill into the project.

## 3. Have the user add the API key

The app needs `THESYS_API_KEY` in its `.env`. Only the user creates it. Ask them to run this in their own terminal, from the app directory:

```bash
npx @openuidev/cli@latest generate-api-key
```

It opens a browser sign-in, mints the key, and writes it to `.env`. `--file .env.local` changes the file. Alternatively, they can create a key in the [Thesys Console](https://console.thesys.dev/keys) and add it to `.env` themselves.

Keep building while they do this. Wait for their confirmation before running anything that calls the model.

## 4. Know the default scaffold

The default scaffold is a Next.js app. The model calls go through OpenUI Gateway's Chat Completions API, and the default model is a free one.

| File | What it does |
| --- | --- |
| `src/agent.ts` | The LangGraph agent. `ChatOpenAI` points at `https://api.thesys.dev/v1/embed`. The system prompt is `generateSystemPrompt({ cloud: true, library })`. After each run, `storeLangGraphHistory` saves the turn to a Gateway conversation. |
| `src/app/api/chat/route.ts` | Validates the request and streams the agent's LangGraph events. |
| `src/app/api/frontend-token/route.ts` | Mints the short-lived browser token for conversation storage. |
| `src/components/cloud-chat.tsx` | `AgentInterface`, wired with `fetchLLM`, `langGraphAdapter()`, `langGraphMessageFormat`, and `useOpenuiCloudStorage`. |
| `src/generated/spec.json` | The library spec the server sends to the model. |
| `src/lib/models.tsx` | The model list for the model switcher. The route accepts only these ids. |
| `src/lib/tools/` | A sample `get_weather` tool. |

The overlays change the backend:

- **`vercel-ai-sdk`:** `src/app/api/chat/route.ts` calls `streamText` with `createOpenAI({ baseURL: "https://api.thesys.dev/v1/embed" })`. The browser uses `vercelAIAdapter()` and `vercelAIMessageFormat`. The route does not save turns to the Gateway conversation.
- **`vercel-eve`:** an Eve agent under `agent/`, rendered through Agent Interface.

An `--example` scaffold has its own layout. Read its README before editing.

## 5. Customize

- **Tools:** add them to the agent the way the framework defines tools (`src/agent.ts` for LangGraph, `src/lib/tools.ts` for the Vercel AI SDK). The application executes them.
- **Components:** the scaffold renders `openuiLibrary`. To change what the model can draw, follow [component-library.md](component-library.md), then regenerate `src/generated/spec.json` from the new library.
- **Chat UI:** layout, welcome screen, starters, and slots are in [agent-interface.md](agent-interface.md). Colors and dark mode are in [theming.md](theming.md).
- **Models:** edit `src/lib/models.tsx`. Ids use `{provider}/{model}`. See https://www.openui.com/cloud/gateway/models.
- **Users:** the frontend-token route uses `DEMO_USER_ID` as the user id. Before real users sign in, derive `user_id` from the server-side session instead.

## 6. Run and verify

1. Run the project's lint, typecheck, and build scripts.
2. Start the dev server, send a message, and confirm the UI builds up while it streams.
3. Reload and confirm the thread comes back.
4. Ask something that uses a tool and confirm the tool runs.
5. Confirm `THESYS_API_KEY` appears in no client file.

## 7. Deploy

Follow [deploy.md](deploy.md). A new app is done when it has a preview URL, unless the user asked to keep it local.

## Sources

- [CLI reference](https://www.openui.com/docs/api-reference/cli)
- [Default template](https://github.com/thesysdev/openui/tree/main/templates/openui-cloud)
- [Example catalog](https://github.com/thesysdev/openui/blob/main/examples/examples.json)
