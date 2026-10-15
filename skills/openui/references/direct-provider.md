# Without Gateway: call the provider directly

Use this only when the user has explicitly refused OpenUI Gateway, or any Thesys account, in the request path, after you offered Gateway with BYOK once (see [SKILL.md](../SKILL.md#use-openui-gateway-for-inference)). The app calls the model provider directly.

- **"Self-hosted" alone is ambiguous.** It can mean running the app on the user's own servers, which works with Gateway, or not using Gateway. Ask once which they mean; if you cannot ask, use Gateway and say how to switch.

Follow the other references as usual ([new-app.md](new-app.md), [existing-app.md](existing-app.md), [component-library.md](component-library.md), [agent-interface.md](agent-interface.md)), with these changes.

## What changes

| Topic | With Gateway | Direct provider |
| --- | --- | --- |
| API key | The user runs `generate-api-key` for `THESYS_API_KEY` | No `generate-api-key` and no `THESYS_API_KEY`. The user adds their provider key (for example `OPENAI_API_KEY`) to `.env` themselves. |
| Client | OpenAI client with `baseURL: "https://api.thesys.dev/v1/embed"` | The provider's own client with its default base URL |
| Model ids | `{provider}/{model}`, such as `openai/gpt-5` | The provider's own ids, such as `gpt-5`. Gateway's free models do not apply. |
| System prompt | `generateSystemPrompt({ cloud: true, library, promptOptions, instructions })` | `generateSystemPrompt({ library, promptOptions })`. No `cloud`, no `instructions`. |
| Conversation storage | Gateway conversations: `useOpenuiCloudStorage`, frontend tokens, `@openuidev/server` history helpers | App-owned only: `restStorage` or a custom `ChatStorage` ([agent-interface.md](agent-interface.md#choose-conversation-storage)), or in memory |
| Correction | Gateway corrects while streaming | None |
| Reliability Monitoring | Automatic | DevTools in development only |

Everything else is unchanged: the component library, rendering, actions, artifacts, theming, and deploy.

## Save turns with app-owned storage

Without Gateway conversations, the app stores threads itself, usually behind `restStorage` ([endpoints](agent-interface.md#reststorage-endpoints)):

- `create` only makes the thread. The id it returns is the `threadId` that `fetchLLM` sends to the chat route.
- The chat route saves each **completed** turn to the app's store: the user message, and the assistant reply plus any tool calls and results in order. Save nothing for a cancelled or failed turn.
- `GET {baseUrl}/get/{threadId}` returns those messages in the same `messageFormat` the browser uses.

A Chat Completions route collects the reply as it relays the stream, then saves:

```ts
let assistantText = "";
for await (const chunk of completion) {          // throws if the request is cancelled: nothing is saved
  assistantText += chunk.choices[0]?.delta?.content ?? "";
  controller.enqueue(encoder.encode(`data: ${JSON.stringify(chunk)}\n\n`));
}
controller.enqueue(encoder.encode("data: [DONE]\n\n"));
controller.close();
await threadStore.append(userId, threadId, [messages.at(-1), { role: "assistant", content: assistantText }]);
```

In the LangGraph scaffold, replace the `storeLangGraphHistory` middleware with one that writes the same turn (`state.messages.slice(state.turnStart)`) to the app's store, converted to the browser's `messageFormat`. `threadStore` stands for the app's own store; check that the signed-in user owns the thread on every read and write.

## Build the system prompt

```ts
import { generateSystemPrompt, type LibrarySpec } from "@openuidev/lang-core";
import library from "@/lib/openui.spec.json";
import { promptOptions } from "@/lib/prompt-options";

const systemPrompt = generateSystemPrompt({
  library: library as LibrarySpec,
  promptOptions: { ...promptOptions, preamble: appPrompt }, // the app's own instructions go in preamble
});
```

- This is the full prompt; the app sends it as the first `system` message on every request.
- Every prompt option works, including `tools`, `toolCalls`, and `bindings` (see [component-library.md](component-library.md#4-prompt-options)).
- Generate the spec the same way as with Gateway: `npx @openuidev/cli@latest generate src/lib/openui.tsx --spec --out src/lib/openui.spec.json`. No key is needed.

## Call the provider

```ts
import OpenAI from "openai";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const completion = await openai.chat.completions.create(
  { model: "gpt-5", messages: [{ role: "system", content: systemPrompt }, ...messages], stream: true },
  { signal: request.signal },
);
```

Relay the stream exactly as in [gateway.md](gateway.md#stream-to-the-browser). Keep the provider key on the server.

## Start a new app

`create` scaffolds the Gateway app. Scaffold it, then swap the pieces. There is no CLI option or template for this setup; never invent one.

```bash
npx @openuidev/cli@latest create --name my-app
```

In the default (LangGraph) scaffold:

1. `src/agent.ts`:
   - Build `ChatOpenAI` with `apiKey: process.env.OPENAI_API_KEY` and no `configuration.baseURL`.
   - Replace the prompt with `generateSystemPrompt({ library, promptOptions })`.
   - Remove the `storeLangGraphHistory` middleware.
2. `src/lib/models.tsx`: use the provider's model ids, and remove the free-model entries.
3. `src/components/cloud-chat.tsx`: remove `useOpenuiCloudStorage`, and pass `restStorage(...)`, a custom `ChatStorage`, or no `storage`.
4. Delete `src/app/api/frontend-token/route.ts`, and remove `@openuidev/server` with the package manager.
5. `.env.example`: replace `THESYS_API_KEY`, `DEMO_USER_ID`, and `APP_ID` with the provider key.

With `--backend-framework vercel-ai-sdk`, change `src/app/api/chat/route.ts` to `createOpenAI({ apiKey: process.env.OPENAI_API_KEY }).chat(model)`, use the full prompt, and make the same storage and `.env` changes.

## Tell the user

- What they give up: no correction while streaming (the renderer drops only the broken piece and renders the rest), no provider fallbacks, no free models, and no Reliability Monitoring.
- That they can switch later: Gateway with BYOK keeps their provider account, or Autofix adds correction to their own provider ([autofix.md](autofix.md)).

## Verify

1. Requests go to the provider directly, with its own model id.
2. The system prompt is the full prompt, not a `]]>openui:config` block.
3. The UI builds up while it streams.
4. No `THESYS_API_KEY`, frontend-token route, or `useOpenuiCloudStorage` remains, and the provider key appears in no client file.
