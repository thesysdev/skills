# Integrate OpenUI Cloud

Read this reference first for shared OpenUI Cloud integration requirements. It owns host discovery, server-side configuration, component compatibility, security, reliability, and verification. Then read only the guide for the requested workload:

| Workload | Continue with |
| --- | --- |
| Agent generation | [Choose an agent API](agent/api-selection.md), then read either [Responses](agent/responses.md) or [Chat Completions](agent/chat-completions.md) |
| Cloud-managed persistent threads for a Responses agent | [Responses](agent/responses.md) plus [Conversations](agent/conversations.md) |
| Standalone slide or report generation and explicit program-based edits | [Standalone artifacts](artifacts.md) |
| New Cloud agent scaffold | [Cloud quickstart](quickstart.md), then the generated template |
| Existing self-hosted/OpenUI OSS application moving to Cloud | [OSS migration](oss-migration.md) |

Responses and Chat Completions are alternative agent-generation protocols. Conversations is an optional persistence and identity layer for Responses; it is not a third generation protocol. Artifact Chat Completions is a specialized standalone artifact endpoint; it is not an agent protocol.

## Preserve Existing Architecture

In an existing application, do not migrate from Chat Completions to Responses merely because Responses is the default for new Cloud agent applications. Preserve the host protocol unless the user requests a migration or requires a Responses-only capability such as Cloud-managed conversations, hosted web/image search, remote MCP, or artifacts inside the agent stream.

Treat generation protocol, message persistence, client renderer, component library, tools, and artifact lifecycle as separate choices. Do not silently replace one because another changes.

## Audit the Host

Before editing, inspect:

1. The package manager, framework/router, server runtime, deployment target, and current `@openuidev/*` and `openai` versions.
2. The generation call and wire protocol: Chat Completions, Responses, AG-UI, another framework adapter, or a custom transport.
3. Who owns history and how it is stored: browser memory, host database, `restStorage`, Cloud Conversations, `previous_response_id`, or full-history requests.
4. The client adapter and message format, including whether the server returns raw `data:` SSE or an OpenAI SDK readable stream.
5. Authentication, stable user identity, authorization, rate limits, request-size limits, CSP/proxy rules, and abort behavior.
6. Existing model selection, provider credentials, tools, artifacts, attachments, component libraries, themes, slots, and tests.

Use the installed application and current first-party template as the source of truth. Do not paste a generic Next.js route over a different framework or orchestration layer.

## Shared Configuration

All Cloud generation calls need a trusted server boundary:

- Store `THESYS_API_KEY` in the deployment secret manager or an untracked server environment file. Never expose it to browser code, logs, generated output, or chat.
- Use the endpoint family selected by the workload: `/v1/embed` for agent generation, `/v1/artifact` for standalone artifacts, and `/v1` for server-side Conversations access. Do not reuse one base URL for every Cloud API.
- Use a current `{provider}/{model}` identifier. Preserve an existing server-side allowlist; reject arbitrary browser-supplied model ids.
- Authenticate and rate-limit application routes independently. Page or layout authentication does not automatically protect API routes.
- Validate request content type, byte size, message/item shapes, and role allowlists before forwarding.
- Propagate abort signals and close streams on success, failure, and cancellation without rewriting the upstream event protocol.
- Keep trusted application instructions separate from user content. Never concatenate user text into system or developer instructions.

Install only packages required by the selected runbook and installed peer ranges. `@openuidev/lang-core` owns current prompt generation, including `generateSystemPrompt({ cloud: true })`. `@openuidev/thesys` supplies managed client libraries and artifact renderers. `@openuidev/thesys-server` is required when using server artifact helpers such as `artifactTool()`.

## Configure BYOK

BYOK is available on every plan, including the free tier. Add provider credentials from the [BYOK page in the Thesys console](https://console.thesys.dev/byok), not to the generated application or client code.

OpenAI and Anthropic accept API keys. Google-hosted Gemini models require the current console credential shape, including the full GCP service-account JSON and a `region`; do not substitute a Google AI Studio key or invent IAM roles, fields, or regions.

Use this human checkpoint:

1. Explain the provider-specific credential shape and console URL without requesting a secret.
2. Ask the user to enter and save the credential directly in the console, then wait for confirmation.
3. Resume with non-secret verification such as selecting the model or testing an authorized request.

Only organization admins can add or update provider credentials. BYOK is provider-specific: requests to other providers use Cloud credits and can fail when the organization has none. The application still authenticates to OpenUI Cloud with its server-side `THESYS_API_KEY`.

Model availability changes independently of this skill. Verify identifiers against the current Models and BYOK documentation, console, and installed template before changing a production allowlist.

## Component Libraries

The model-facing prompt and browser renderer must use the same component contract. Use the managed `chatLibrary` on the client with the managed built-in Cloud prompt, or pass a generated custom-library spec to the prompt helper and the matching runtime library to the renderer. Read [build-component-library.md](../build-component-library.md) before building, extending, or migrating a library.

## Reliability and Observability

Managed generative UI validates and repairs output against the selected component contract. Preserve the Cloud response stream and matching OpenUI adapter; do not add a second blind stream-rewriting layer.

Managed correction does not make model output deterministic. Run a representative prompt set repeatedly against the application's actual library and compare structural failures, partial renders, latency, and cost. In development, use OpenUI DevTools to inspect settled streams and parser/renderer errors.

When production monitoring is required, use the current `@openuidev/observability-cloud` setup and a separate client instrumentation key created in the Thesys Console. It is intentionally visible in browser code; keep `THESYS_API_KEY` server-only. Initialize observability once and verify events reach the Reliability dashboard.

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

- `https://www.openui.com/docs/openui-cloud/api/overview`
- `https://www.openui.com/docs/openui-cloud/api/responses`
- `https://www.openui.com/docs/openui-cloud/api/chat-completions`
- `https://www.openui.com/docs/openui-cloud/api/artifacts`
- `https://www.openui.com/docs/openui-cloud/api/conversations`
- `https://www.openui.com/docs/openui-cloud/models-and-byok`
- `https://www.openui.com/docs/openui-cloud/build/component-library`
- `https://www.openui.com/docs/agent/reference/adapters-and-formats`
