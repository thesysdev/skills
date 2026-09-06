# Generate Standalone OpenUI Cloud Artifacts

Read [the shared Cloud integration guide](integration.md) first. Use this reference for the specialized Artifact Chat Completions endpoint when generating or explicitly editing a standalone OpenUI Cloud slide deck or report. This endpoint is separate from the Responses-versus-Chat-Completions choice for agent applications.

## Choose the Artifact Lifecycle

| Desired lifecycle | Use |
| --- | --- |
| Standalone slide/report generation, with the application storing the returned program | Artifact Chat Completions in this guide |
| Explicit edit where the application sends the current program and an edit instruction | Artifact Chat Completions in this guide |
| Artifact generated, stored, reopened, and edited inside a persistent agent conversation | Responses with `artifactTool()`; read [responses.md](chat/responses.md) and [conversations.md](chat/conversations.md) |

Do not use the standalone artifact endpoint as an Agent Interface conversation store. It returns an OpenUI Lang artifact program; the application owns that program and its surrounding persistence, authorization, and version history.

## Configure the Client

Keep `THESYS_API_KEY` on the application server and configure the stock OpenAI SDK for the artifact base URL:

```ts
import OpenAI from "openai";

const artifactClient = new OpenAI({
  apiKey: process.env.THESYS_API_KEY,
  baseURL: "https://api.thesys.dev/v1/artifact",
});
```

The generation endpoint is:

```text
POST https://api.thesys.dev/v1/artifact/chat/completions
```

Use a current `{provider}/{model}` id from trusted server configuration. Preserve the host's model allowlist rather than accepting arbitrary browser-supplied model ids.

## Generate an Artifact

Every request includes `metadata.thesys` as a JSON string. Supply a stable application artifact id and a `c1_artifact_type` of `"slides"` or `"report"`:

```ts
const artifact = await artifactClient.chat.completions.create({
  model,
  messages: [
    { role: "user", content: "Create a three-slide deck on Q4 results." },
  ],
  metadata: {
    thesys: JSON.stringify({
      id: artifactId,
      c1_artifact_type: "slides",
    }),
  },
});

const program = artifact.choices[0]?.message.content;
if (!program) throw new Error("Artifact generation returned no program");
```

`metadata.thesys` is a stringified object, not a nested metadata object. Validate the artifact id and type before constructing it.

The response content is a validated OpenUI Lang program rooted at `SlideShow` for slides or `ReportView` for reports. Set `stream: true` to receive the program progressively, preserve the upstream Chat Completions event shape, and pass `isStreaming` to the viewer while accumulating content.

## Render the Program

Render the returned program with the matching managed viewer:

```tsx
import { Presentation, Report } from "@openuidev/thesys";
import "@openuidev/thesys/styles.css";

export function Artifact({
  kind,
  program,
  isStreaming,
}: {
  kind: "slides" | "report";
  program: string;
  isStreaming?: boolean;
}) {
  return kind === "slides" ? (
    <Presentation response={program} preview={false} isStreaming={isStreaming} />
  ) : (
    <Report response={program} preview={false} isStreaming={isStreaming} />
  );
}
```

Keep `@openuidev/thesys` imports inside the host framework's client boundary and import its stylesheet once. Preserve the product's loading, error, routing, and authorization behavior around the viewer.

## Edit an Artifact

An edit request sends the complete current program as an assistant message, follows it with the requested change, and sets `is_edit: true`:

```ts
const edited = await artifactClient.chat.completions.create({
  model,
  messages: [
    { role: "assistant", content: previousProgram },
    { role: "user", content: "Make slide 2 about European revenue." },
  ],
  metadata: {
    thesys: JSON.stringify({
      id: artifactId,
      c1_artifact_type: "slides",
      is_edit: true,
    }),
  },
  stream: true,
});
```

The response is a patch-mode OpenUI Lang program merged against the assistant-message base. Keep the artifact id and type consistent, load the authoritative prior program from application storage, and authorize access before calling Cloud. Do not trust a browser-supplied base program or artifact id when the server can load them from its own store.

## Preserve Application Ownership

The application owns standalone artifact state:

- Persist the current program and artifact type after successful generation or editing.
- Keep versioning, optimistic concurrency, authorization, and deletion in the host application.
- Bound prompt and previous-program sizes before forwarding them.
- Forward abort signals and terminate streamed responses cleanly.
- Do not log keys, sensitive prompts, or artifact programs unless the product's data policy explicitly permits it.
- Do not describe a standalone artifact as stored in Cloud Conversations.

The endpoint supports the managed `slides` and `report` types. Do not invent arbitrary `c1_artifact_type` values or route custom application artifacts through this contract.

## Verify

1. Confirm the route calls `/v1/artifact/chat/completions`, not the Embed Chat Completions or Responses endpoint.
2. Confirm `metadata.thesys` is JSON-encoded exactly once and contains the intended id and supported artifact type.
3. Generate one slide deck and one report; verify their roots and render each with the matching viewer.
4. Stream a program and confirm progressive rendering receives the correct `isStreaming` state.
5. Edit a stored artifact and confirm the authoritative prior program is sent as the assistant message with `is_edit: true`.
6. Verify unauthorized users cannot load or edit another user's stored program or artifact id.
7. Test missing configuration, invalid types, oversized inputs, empty output, Cloud failures, cancellation, and stream closure.
8. Run the host formatter, typecheck, tests, and production build.

## First-Party References

- `https://www.openui.com/docs/openui-cloud/api/artifacts`
- `https://www.openui.com/docs/openui-cloud/api/overview`
- `https://www.openui.com/docs/openui-cloud/api/responses`
- `https://www.openui.com/docs/openui-cloud/how-it-works`
