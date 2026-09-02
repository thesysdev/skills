# Choose an OpenUI Cloud API

Read this reference before selecting a Cloud endpoint, history model, stream adapter, or prompt helper. Prefer the current generated template when it differs from this guide.

## API Matrix

| Surface | Endpoint | Use when | History owner |
| --- | --- | --- | --- |
| Responses | `POST https://api.thesys.dev/v1/embed/responses` | Building a new agent app with managed UI, hosted tools, persistent conversations, or in-stream artifacts | Cloud when using `conversation` plus `store: true`; otherwise the application can send full input or use `previous_response_id` |
| Embed Chat Completions | `POST https://api.thesys.dev/v1/embed/chat/completions` | Preserving an existing OpenAI Chat Completions integration, using plain text passthrough, or running app-owned function tools with message history | Application |
| Artifact Chat Completions | `POST https://api.thesys.dev/v1/artifact/chat/completions` | Generating or explicitly editing a standalone slide deck or report outside an agent conversation | Application sends the current artifact program on edits |
| Conversations | `https://api.thesys.dev/v1/conversations` | Creating, listing, updating, or deleting persistent Responses threads and items | Cloud |

Use Responses for new agent applications unless the user's existing protocol or desired artifact lifecycle calls for another surface. Do not use Embed Chat Completions with the Conversations API; Chat Completions applications resend their own `messages` history.

## Match the Client Contract

| Server response | Agent Interface stream adapter | Message format |
| --- | --- | --- |
| Responses SSE | `openAIResponsesAdapter()` | `openAIConversationMessageFormat` when using Cloud conversations |
| Raw Chat Completions `data:` SSE | `openAIAdapter()` | `openAIMessageFormat` |
| OpenAI SDK `.toReadableStream()` | `openAIReadableStreamAdapter()` | `openAIMessageFormat` |

For Responses with `conversation: threadId` and `store: true`, send only the latest user turn. Do not also resend full history. Preserve the upstream event shape when proxying the stream.

The Conversations API is the storage plane used by `useOpenuiCloudStorage()`. Browser access requires a short-lived frontend token minted by the application server for an authenticated `user_id` and optional stable `app_id`; never expose `THESYS_API_KEY` to browser code.

## Choose the Prompt Helper

Use `generateSystemPrompt()` from `@openuidev/thesys-server` for managed Cloud generative UI:

- Responses: put the result in `instructions`.
- Embed Chat Completions: put the result in a `role: "system"` message.
- With the built-in `chatLibrary`, call `generateSystemPrompt()` with no library.
- With a custom library, pass the serialized `library` spec and optional `instructions`/`promptOptions`.

Use `generateSystemPrompt()` from `@openuidev/lang-core` only when the application owns prompt assembly and validation, including self-hosted generative UI sent through Embed Chat Completions.

## Use a Custom Component Library

Keep the renderer library and the backend library spec in sync:

1. Define or extend the client library with `defineComponent()` and `createLibrary()`.
2. Generate the serializable spec at build time:

   ```bash
   npx @openuidev/cli@latest generate --spec ./src/lib/chat-library.tsx --out ./src/generated/library-spec.json
   ```

3. Import the spec in the server route and pass it to the Cloud helper:

   ```ts
   import librarySpec from "@/generated/library-spec.json";
   import { generateSystemPrompt } from "@openuidev/thesys-server";

   const instructions = generateSystemPrompt({
     library: librarySpec,
     instructions: "Optional application instructions.",
     promptOptions: {
       preamble: "Optional domain context.",
     },
   });
   ```

4. Pass the matching runtime library to `AgentInterface.componentLibrary` or `Renderer.library`.
5. Regenerate the spec whenever the component names, descriptions, prop schemas, root, groups, or prompt options change.

`promptOptions` is valid only when a custom `library` spec is supplied to the managed Cloud helper. Do not pass a JSON Schema generated with `--json-schema`; Cloud needs the serialized library spec generated with `--spec`.

## Tools and Artifacts

- Responses supports hosted web search, image search, remote MCP, managed artifacts, and app-owned function tools.
- Embed Chat Completions returns app-owned function calls for the application to execute; it does not provide Responses-hosted search, MCP, or artifact tools.
- Artifact Chat Completions is for standalone slide/report programs and explicit program-based edits.
- Use Responses plus `artifactTool()` when artifacts should live inside a persistent agent conversation.

## First-Party References

- `https://www.openui.com/docs/openui-cloud/api/overview`
- `https://www.openui.com/docs/openui-cloud/api/responses`
- `https://www.openui.com/docs/openui-cloud/api/chat-completions`
- `https://www.openui.com/docs/openui-cloud/api/artifacts`
- `https://www.openui.com/docs/openui-cloud/api/conversations`
- `https://www.openui.com/docs/openui-cloud/build/component-library`
