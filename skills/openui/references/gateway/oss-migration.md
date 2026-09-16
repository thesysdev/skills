# Migrate an OpenAI-Compatible App to Gateway

Use this runbook to route an existing OpenAI-compatible application through Gateway. Follow the [first-party migration guide](https://www.openui.com/docs/gateway/migrate-to-openui-gateway) and [shared configuration](integration.md). Preserve the existing messages, streaming handlers, function-tool loop, and persistence.

## Change the Client Configuration

1. Configure a server-side inference key as `THESYS_API_KEY`; use the [authentication handoff](quickstart.md#complete-authentication-with-the-user) if setup is needed.
2. Set the client's base URL to `https://api.thesys.dev/v1/embed`.
3. Choose a supported model using its full `{provider}/{model}` identifier, such as `openai/gpt-5`. Retain the application's model allowlist and verify availability in [Models](https://www.openui.com/docs/gateway/models).

```ts
import OpenAI from "openai";

export const client = new OpenAI({
  apiKey: process.env.THESYS_API_KEY,
  baseURL: "https://api.thesys.dev/v1/embed",
});
```

Use [BYOK](integration.md#configure-byok) when the application should use its own provider credentials through Gateway.

## Preserve the Request Contract

Continue using `chat.completions.create()` with the application's existing system prompt, relevant `messages`, function declarations, and streaming options. Keep the tool executor and append assistant tool calls and their results as before. Gateway provides model routing and provider fallbacks for this traffic.

Read [Chat Completions](chat/chat-completions.md) for endpoint details. An application already using Responses should preserve that protocol and its history model; follow [Responses](chat/responses.md). A change to the model provider does not require adopting Gateway Conversations or migrating stored data.

## Add Managed Generative UI When Requested

Managed OpenUI Lang generation is a separate opt-in. Follow [component-library handoff](../build-component-library.md) to export the existing library spec and configure `generateSystemPrompt({ cloud: true, library })` in the selected generation request. The application's renderer must use the matching library.

For ordinary model traffic, keep the existing prompt directly; no OpenUI prompt helper is needed.

## Verify

- Confirm requests use the Gateway key, base URL, and provider-qualified model id.
- Verify incremental streaming, cancellation, and the existing error behavior.
- Exercise a function-tool flow and check that message history and persistence still work.
- When managed generative UI is enabled, confirm output uses the expected library and renders correctly.
- Run the host's relevant checks before removing an unused provider configuration.
