# Self-Hosted OpenUI

Use this guide when the application owns its model provider, generation route, and storage. Everything here works without a Thesys account. Each section ends with what Gateway adds, so the app can move to Autofix or Gateway later without a rewrite.

## Scaffold

```bash
npx @openuidev/cli@latest create --name my-app --template openui-self-hosted
```

Add `--backend-framework langgraph`, `vercel-ai-sdk`, or `vercel-eve` for an agent framework; the default backend is a Chat Completions route using the OpenAI SDK. Set `OPENAI_API_KEY`, and optionally `OPENAI_MODEL`, in `.env.local`. The [Gateway quickstart](gateway/quickstart.md#scaffold) covers the shared CLI flags.

## Generate on the Server

The model's system prompt comes from the serialized spec of the library the client renders. Generate the spec with `openui generate --spec` ([component-library handoff](build-component-library.md#generate-the-handover-spec)) and compile it with `generateSystemPrompt` from `@openuidev/lang-core`:

```ts
import librarySpec from "@/generated/spec.json";
import { promptOptions } from "@/lib/prompt-options";
import { generateSystemPrompt } from "@openuidev/lang-core";
import OpenAI from "openai";
import type { ChatCompletionMessageParam } from "openai/resources/chat/completions";

const client = new OpenAI();

export async function POST(req: Request) {
  // Authenticate, rate-limit, and validate the body before calling the provider.
  const { messages } = (await req.json()) as { messages: ChatCompletionMessageParam[] };

  return client.chat.completions
    .create(
      {
        model: process.env.OPENAI_MODEL ?? "gpt-5.2",
        messages: [
          { role: "system", content: generateSystemPrompt({ library: librarySpec, promptOptions }) },
          ...messages,
        ],
        stream: true,
      },
      { signal: req.signal },
    )
    .asResponse();
}
```

`.asResponse()` returns the SDK's raw Chat Completions SSE unchanged. For the built-in library, the template exports `openuiLibrary` from `@openuidev/react-ui/genui-lib` and its prompt options from `@openuidev/react-ui/genui-lib/prompt-options`, which keeps React components out of the server route.

For a provider without a matching stream adapter, such as Anthropic, translate its stream into AG-UI events on the server and use `agUIAdapter()` on the client, following the [self-hosting reference](https://www.openui.com/docs/agent/reference/self-hosting).

**Gateway adds:** changing the key, base URL, and model id routes the same request through Gateway, which corrects OpenUI Lang in the stream and falls back across providers. See [migrate.md](gateway/migrate.md#self-hosted-openui-app).

## Connect the Client

The template renders the same library in Agent Interface and parses raw Chat Completions SSE:

```tsx
"use client";

import { AgentInterface, fetchLLM, openAIAdapter, openAIMessageFormat } from "@openuidev/react-ui";
import { openuiLibrary } from "@openuidev/react-ui/genui-lib";

const llm = fetchLLM({
  url: "/api/chat",
  streamAdapter: openAIAdapter(),
  messageFormat: openAIMessageFormat,
});

export default function Chat() {
  return (
    <div style={{ height: "100dvh" }}>
      <AgentInterface llm={llm} componentLibrary={openuiLibrary} agentName="Assistant" />
    </div>
  );
}
```

Pick the adapter from the bytes the route returns; the full table is in [Agent Interface](agent-interface.md#match-the-browser-stream). For a chat UI the app already owns, render assistant text with `Renderer` instead (see [SKILL.md](../SKILL.md#set-up-react-ui)).

## Store Conversations

The self-hosted template keeps messages in memory, so a reload clears them. To persist threads, pass `storage` to Agent Interface: `restStorage({ baseUrl })` against routes the app implements, or a custom `ChatStorage` around its existing database. The route or framework must also write completed assistant and tool messages, because a storage adapter alone does not persist replies. Details are in [Agent Interface storage](agent-interface.md#choose-conversation-storage); the [Supabase example](examples.md#specialized-examples) shows a complete app-owned store.

**Gateway adds:** Conversations stores threads and items for the app. With Responses, Gateway writes each turn itself; with Chat Completions, `storeChatCompletionHistory` from `@openuidev/server/openai` writes them. See [conversations.md](gateway/conversations.md).

## Run Function Tools

Chat Completions declares function tools but never executes them. Run a bounded loop in the route:

1. Send the relevant `messages` with the function declarations.
2. Read `tool_calls` from the assistant message.
3. Check each name against the app's allowlist and validate its arguments.
4. Execute authorized tools on the server.
5. Append the assistant tool-call message and one `role: "tool"` result per call.
6. Repeat, up to a fixed iteration limit, until the model returns a final answer.

Forward tool calls and results through the browser stream when the UI should show them or open them as [artifacts](artifacts.md). The LangGraph, Vercel AI SDK, and Eve overlays run their own tool loops; extend those instead of adding one to the route.

**Gateway adds:** the Responses API runs web search, image search, and remote MCP inside Gateway; see [Responses hosted tools](gateway/responses.md#add-hosted-tools).

## Correct and Monitor Output

Model output is nondeterministic, so plan for invalid OpenUI Lang. [reliability.md](reliability.md) covers measurement, development tools, and correction. Self-hosted apps choose one of two ways to correct output:

- **Autofix:** wrap the existing stream with `createAutofix` from `@openuidev/server`. The provider stays the same, a fix is billed only when one runs, and failures appear in Reliability Monitoring. See [autofix.md](gateway/autofix.md).
- **Your own loop:** parse the settled response, send precise errors back to the model in one bounded retry, and fall back when it still fails. See [correct invalid output](reliability.md#correct-invalid-output).

For production monitoring without Gateway or Autofix, install `@openuidev/observability-cloud` ([monitor production](reliability.md#monitor-production)).

## Verify

1. `openui generate --spec` succeeds, and the route prompt and client renderer use the same library.
2. Responses stream progressively; cancel and provider errors close the stream and show an error.
3. Provider keys are absent from client source and the built bundle.
4. With storage, reload a thread and confirm full history, including tool calls and results.
5. With tools, run a multi-step tool call and confirm the loop stops at its limit.
