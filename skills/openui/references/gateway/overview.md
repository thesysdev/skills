# OpenUI Gateway

OpenUI Gateway is the hosted API for OpenUI apps. Read this guide before any Gateway integration, then continue with the guide for the chosen API.

## What Gateway Provides

| Capability | Available through |
| --- | --- |
| OpenUI Lang validation and correction in the stream | Responses and Chat Completions with `generateSystemPrompt({ cloud: true, library })` |
| Model routing and provider fallbacks | Responses and Chat Completions |
| Managed inference or your own provider keys (BYOK) | Responses and Chat Completions; see [BYOK](#configure-byok) |
| Hosted web search, image search, and remote MCP | [Responses](responses.md#add-hosted-tools) |
| Persistent threads with browser access | [Conversations](conversations.md) |
| Repair of output from any provider | [Autofix](autofix.md) |
| Reliability Monitoring | Automatic for all Gateway and Autofix requests; see [reliability.md](../reliability.md#monitor-production) |

Agent Interface, component rendering, theming, and application authorization belong to the app and work the same with or without Gateway.

## Endpoints

| Base URL | Paths | Use |
| --- | --- | --- |
| `https://api.thesys.dev/v1/embed` | `/responses`, `/chat/completions` | Generation; set as the OpenAI SDK `baseURL` |
| `https://api.thesys.dev/v1` | `/conversations`, `/frontend-tokens` | Conversation storage and browser tokens |
| `https://api.thesys.dev/v1/autofix` | `/chat/completions` (or the base path) | Autofix repair |

The stock `openai` SDK works for all three. Model ids use `{provider}/{model}`, for example `openai/gpt-5`; check availability in [Models](https://www.openui.com/docs/gateway/models) before changing an allowlist.

## Choose an API

| API | Choose it for | Guide |
| --- | --- | --- |
| Responses | New chat or agent apps, existing Responses apps, and hosted tools | [responses.md](responses.md) |
| Chat Completions | Existing `chat.completions.create()` apps, app-owned `messages`, and app-run function tools | [chat-completions.md](chat-completions.md) |

Responses is the default for new apps, not a required migration target. Keep an existing Chat Completions app on Chat Completions unless the user asks to migrate or needs hosted tools or Gateway-managed history.

History is a separate choice. Responses supports three history models (full `input`, `previous_response_id`, or a named `conversation`); see [responses.md](responses.md#choose-the-history-model). Chat Completions always sends the relevant `messages`. Conversations is a storage API, not a third generation protocol.

## Configure the Prompt

For generated UI, compile the prompt from the serialized spec of the library the client renders:

```ts
import { generateSystemPrompt } from "@openuidev/lang-core";
import librarySpec from "./generated/library-spec.json";

const prompt = generateSystemPrompt({
  cloud: true,
  library: librarySpec,
  instructions: trustedApplicationInstructions, // optional
});
```

- Responses: pass the result as `instructions`.
- Chat Completions: send it as the `role: "system"` message.
- `promptOptions` with `cloud: true` accepts only `examples`, `preamble`, and `additionalRules`.
- Regenerate the spec whenever the library changes; see [build-component-library.md](../build-component-library.md#generate-the-handover-spec).

For plain model traffic, keep the app's existing prompt and do not call `generateSystemPrompt()`. Gateway then routes models and falls back across providers without OpenUI Lang correction.

## Protect the Server Boundary

- Keep `THESYS_API_KEY` in server environment or a secret manager, never in browser code, logs, generated output, or chat. When the key is missing, follow the [authentication handoff](quickstart.md#complete-authentication-with-the-user).
- Authenticate and rate-limit every generation, token, and framework-session route on its own; page or layout auth does not protect API routes.
- Enforce content type and a byte limit, validate message or item shapes, and allow only supported roles before forwarding.
- Select models from a server-side allowlist; reject arbitrary model ids from the browser.
- Keep trusted instructions separate from user content; never concatenate user text into system or developer instructions.
- Forward the request's abort signal, and close the stream on success, failure, and cancellation without rewriting the upstream event format.

## Configure BYOK

BYOK is available on every plan. Provider credentials go in the [BYOK page of the Thesys Console](https://console.thesys.dev/byok), not in the app.

1. Tell the user which provider credential the console form asks for and link the page. Never ask for the secret in chat.
2. Ask the user to save it in the console and confirm when done.
3. Continue with a non-secret check, such as selecting a model from that provider and sending an authorized request.

Only organization admins can add provider credentials. BYOK is per provider: requests to other providers use managed inference and Gateway credits. The app still authenticates to Gateway with `THESYS_API_KEY`.

For prompt cost and latency, keep the prompt and spec prefix stable across requests and follow [Prompt Caching](https://www.openui.com/docs/gateway/prompt-caching).

## Verify

1. Run the host formatter, typecheck, tests, and production build.
2. Confirm `THESYS_API_KEY` is absent from client source and the built bundle.
3. Confirm logged-out requests, oversized bodies, disallowed roles, and unknown models are rejected.
4. Confirm the client adapter and message format match the route's stream.
5. Confirm the history owner receives exactly the intended history, with nothing duplicated or dropped.
6. Confirm the prompt spec and the client renderer come from the same library.
7. Test upstream failures, cancellation, and stream closure.

## First-Party References

- `https://www.openui.com/docs/gateway`
- `https://www.openui.com/docs/gateway/generate-openui-lang`
- `https://www.openui.com/docs/gateway/authentication`
- `https://www.openui.com/docs/gateway/models`
- `https://www.openui.com/docs/gateway/byok`
- `https://www.openui.com/docs/gateway/reliability`
- `https://www.openui.com/docs/gateway/pricing-credits`
