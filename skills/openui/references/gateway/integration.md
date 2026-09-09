# Integrate OpenUI Gateway

Read this reference first for shared OpenUI Gateway integration requirements. Gateway is the current name for hosted model access and OpenUI Lang correction; the CLI still uses `openui-cloud` and prompt compilation still uses `cloud: true`. This guide owns host discovery, configuration, compatibility, security, reliability, and verification. Then read only the guide for the requested workload:

| Workload | Continue with |
| --- | --- |
| Conversational generation | [Choose a chat generation API](chat/api-selection.md), then read either [Responses](chat/responses.md) or [Chat Completions](chat/chat-completions.md) |
| Gateway-managed persistent threads for a Responses chat | [Responses](chat/responses.md) plus [Conversations](chat/conversations.md) |
| Standalone slide or report generation and explicit program-based edits | [Standalone artifacts](artifacts.md) |
| New Gateway agent scaffold | [Gateway quickstart](quickstart.md), then the generated template |
| Existing self-hosted/OpenUI OSS application moving to Gateway | [OSS migration](oss-migration.md) |

Responses and Chat Completions are alternative conversational generation protocols. Conversations supplies named-thread persistence for Responses; it is not a third generation protocol. An explicitly configured framework can separately manage storage without changing its model protocol. Artifact Chat Completions is a separate workload for standalone slides and reports.

## Preserve Existing Architecture

In an existing application, do not migrate from Chat Completions to Responses merely because Responses is the default starter backend. Preserve the host protocol unless the user requests migration or needs Responses-specific conversation appends, hosted web/image search, remote MCP, or managed artifacts inside the agent stream.

Treat generation protocol, message persistence, client renderer, component library, tools, and artifact lifecycle as separate choices. Do not silently replace one because another changes.

## Audit the Host

Before editing, inspect:

1. The package manager, framework/router, server runtime, deployment target, and current `@openuidev/*` and `openai` versions.
2. The generation call and wire protocol: Chat Completions, Responses, AG-UI, another framework adapter, or a custom transport.
3. Who owns history and how it is stored: browser memory, host database, `restStorage`, Gateway Conversations, `previous_response_id`, or full-history requests.
4. The client adapter and message format, including whether the server returns raw `data:` SSE or an OpenAI SDK readable stream.
5. Authentication, stable user identity, authorization, rate limits, request-size limits, CSP/proxy rules, and abort behavior.
6. Existing model selection, provider credentials, tools, artifacts, attachments, component libraries, themes, slots, and tests.

Use the installed application and current first-party template as the source of truth. Do not paste a generic Next.js route over a different framework or orchestration layer.

## Shared Configuration

All Gateway generation calls need a trusted server boundary:

- Store `THESYS_API_KEY` in the deployment secret manager or an untracked server environment file. Never expose it to browser code, logs, generated output, or chat.
- If the key or Gateway account is not configured, follow [the authentication handoff](quickstart.md#complete-authentication-with-the-user) to involve the user during the task and resume verification afterward. Do not infer a self-hosted architecture or rebuild a managed feature to bypass credentials.
- Use `/v1/embed` for model generation, `/v1` for server-side Conversations access, and `/v1/artifact` for standalone artifacts following [artifacts.md](artifacts.md). Do not reuse one base URL for every API.
- Use a current `{provider}/{model}` identifier. Preserve an existing server-side allowlist; reject arbitrary browser-supplied model ids.
- Authenticate and rate-limit application routes independently. Page or layout authentication does not automatically protect API routes.
- Validate request content type, byte size, message/item shapes, and role allowlists before forwarding.
- Propagate abort signals and close streams on success, failure, and cancellation without rewriting the upstream event protocol.
- Keep trusted application instructions separate from user content. Never concatenate user text into system or developer instructions.

Install only packages required by the selected runbook and installed peer ranges. `@openuidev/lang-core` owns current prompt generation, including `generateSystemPrompt({ cloud: true })`. `@openuidev/thesys` supplies managed client libraries and artifact renderers. `@openuidev/thesys-server` is required when using server artifact helpers such as `artifactTool()`.

## Configure BYOK

BYOK is available on every plan, including the free tier. Add provider credentials from the [BYOK page in the Thesys console](https://console.thesys.dev/byok), not to the generated application or client code.

Use the credential form currently shown for the chosen provider. The current BYOK guide lists OpenAI, Anthropic, and Google but does not specify Google's required credential fields. Do not prescribe a Google AI Studio key, GCP service-account JSON, region, or IAM roles from older instructions without checking the current console or first-party provider setup guidance.

Use this human checkpoint:

1. Explain the provider-specific credential shape and console URL without requesting a secret.
2. Ask the user to enter and save the credential directly in the console, then wait for confirmation.
3. Resume with non-secret verification such as selecting the model or testing an authorized request.

Only organization admins can add or update provider credentials. BYOK is provider-specific: requests to other providers use Gateway credits and can fail when the organization has none. The application still authenticates to OpenUI Gateway with its server-side `THESYS_API_KEY`.

Model availability changes independently of this skill. The [Models guide](https://www.openui.com/docs/gateway/models) now points to a broader model catalog. Verify the full model id, modalities, tool support, account availability, and installed template before changing a production allowlist; a catalog listing is not proof that every provider option is supported by Gateway.

## Component Libraries

The model-facing prompt and browser renderer must use the same component contract. Use the managed `chatLibrary` on the client with the managed built-in Gateway prompt, or pass a generated custom-library spec to the prompt helper and the matching runtime library to the renderer. Read [build-component-library.md](../build-component-library.md) before building, extending, or migrating a library.

Gateway's HTTP APIs do not require React or Agent Interface. Preserve an existing Vue, Svelte, native, or custom UI when its renderer supports the selected library. Verify framework support separately for the managed `@openuidev/thesys` components and artifact viewers; do not promise those React surfaces on every runtime.

## Reliability and Observability

Managed generative UI validates and corrects eligible OpenUI Lang errors against the selected component contract. This does not validate business data, execute application tools safely, or repair arbitrary JSON/component code. Preserve the selected stream contract; do not add a second blind stream-rewriting layer. Provider fallback behavior depends on model compatibility and account configuration; verify it for strict model or data-residency requirements.

Managed correction does not make model output deterministic. Run a representative prompt set repeatedly against the application's actual library and compare structural failures, partial renders, latency, and cost. In development, use OpenUI DevTools to inspect settled streams and parser/renderer errors.

OpenUI Observability is independent of Gateway and also works with direct-provider generation. For production monitoring, follow the [installation guide](https://www.openui.com/docs/observability/installation), use `@openuidev/observability-cloud`, and create a separate [client instrumentation key](https://console.thesys.dev/client-api-keys). That key is browser-visible; `THESYS_API_KEY` remains server-only. Initialize once before rendering and verify runtime events in the dashboard. Observability detects errors; it does not correct them.

For prompt-cost or latency work, consult [Prompt Caching](https://www.openui.com/docs/gateway/prompt-caching). Keep reusable prompt/spec prefixes stable, check the selected provider's caching behavior, and do not invent support for per-breakpoint TTLs.

## Shared Verification

1. Run the host formatter, typecheck, tests, and production build.
2. Confirm the server key is absent from client source and the built browser bundle.
3. Confirm authentication, rate limiting, request bounds, provider failures, cancellation, and stream closure.
4. Confirm the adapter and message format match the selected upstream protocol.
5. Confirm the selected history owner receives exactly the intended history—neither duplicated nor silently discarded.
6. Confirm the model and renderer use the same component library contract.
7. Exercise each declared tool through its real execution owner and ensure undeclared tools cannot run.
8. Test user isolation for whichever persistence layer the application retains.

## First-Party References

- `https://www.openui.com/docs/gateway`
- `https://www.openui.com/docs/gateway/api/responses`
- `https://www.openui.com/docs/gateway/api/chat-completions`
- `https://www.openui.com/docs/gateway/api/conversations`
- `https://www.openui.com/docs/gateway/authentication`
- `https://www.openui.com/docs/gateway/models`
- `https://www.openui.com/docs/gateway/byok`
- `https://www.openui.com/docs/gateway/reliability`
- `https://www.openui.com/docs/gateway/generate-openui-lang`
- `https://www.openui.com/docs/agent/reference/adapters-and-formats`
