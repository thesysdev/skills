# Generate Standalone OpenUI Gateway Artifacts

Read [the shared Gateway integration guide](integration.md) first. This bundled reference includes the client setup, generation, rendering, streaming, and editing examples for standalone OpenUI Gateway slides/reports. Use the snippets below as the implementation guide; fetching an external guide or waiting for a documentation update is not required. Adapt them to the target application's installed SDK versions and verify generation/editing before claiming runtime support.

## Choose the Artifact Lifecycle

| Desired lifecycle | Use |
| --- | --- |
| Standalone slide/report generation, with the application storing the returned program | Artifact Chat Completions in this guide |
| Explicit edit where the application sends the current program and an edit instruction | Artifact Chat Completions in this guide |
| Artifact generated, stored, reopened, and edited inside a persistent agent conversation | Responses with `artifactTool()`; read [responses.md](chat/responses.md) and [conversations.md](chat/conversations.md) |

Do not use the standalone artifact endpoint as an Agent Interface conversation store. It returns an OpenUI Lang artifact program; the application owns that program and its surrounding persistence, authorization, and version history.

## Configure the Client

Install `openai` for server requests and `@openuidev/thesys` for the React viewers using the application's package manager. Create an API key in the [Thesys console](https://console.thesys.dev/keys), following the [authentication handoff](quickstart.md#complete-authentication-with-the-user) if user action is needed. Keep `THESYS_API_KEY` on the application server and configure the stock OpenAI SDK for the artifact base URL:

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

Call this client from an authenticated application server route, not from browser code. In the examples, `model` is the allowed model id and `artifactId` is a stable application-owned id for the artifact being generated or edited.

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

The response content is a validated OpenUI Lang program rooted at `SlideShow` for slides or `ReportView` for reports. Use `c1_artifact_type: "report"` to generate a report with the same request shape. For progressive output, follow [Stream an Artifact](#stream-an-artifact).

## Render the Program

Render the returned program with the matching managed viewer:

```tsx
"use client";

import { Presentation, Report } from "@openuidev/thesys";
import "@openuidev/thesys/styles.css";

export function Artifact({
  kind,
  program,
  isStreaming = false,
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

## Stream an Artifact

Use `stream: true` to generate a new artifact progressively. This example generates a report; `artifactId` identifies that report. Accumulate the content deltas in order on the server:

```ts
const stream = await artifactClient.chat.completions.create({
  model,
  messages: [{ role: "user", content: "Create a report on Q4 results." }],
  metadata: {
    thesys: JSON.stringify({
      id: artifactId,
      c1_artifact_type: "report",
    }),
  },
  stream: true,
});

let program = "";
for await (const chunk of stream) {
  program += chunk.choices[0]?.delta?.content ?? "";
  // Forward the accumulated program to your application's viewer.
}
```

Forward updates from the server route to the browser using the application's streaming transport. If proxying raw Chat Completions events, preserve their shape and accumulate content in the browser instead; do not confuse those events with the accumulated program string. Pass the accumulated program as the viewer's `response`, keep `isStreaming={true}` while receiving updates, and set it to `false` on completion or failure. Use `Presentation` for `"slides"` and `Report` for `"report"`. Preserve cancellation and error handling; persist the complete program only after successful generation.

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

The response is a patch-mode OpenUI Lang program merged against the assistant-message base. Keep the artifact id and type consistent, load `previousProgram` from authoritative application storage, and authorize access before calling Gateway. Interpret the edit in the context of that complete base program, then persist the resulting complete program for subsequent views and edits. Do not trust a browser-supplied base program or artifact id when the server can load them from its own store.

## Preserve Application Ownership

The application owns standalone artifact state:

- Persist the current program and artifact type after successful generation or editing.
- Keep versioning, optimistic concurrency, authorization, and deletion in the host application.
- Bound prompt and previous-program sizes before forwarding them.
- Forward abort signals and terminate streamed responses cleanly.
- Do not log keys, sensitive prompts, or artifact programs unless the product's data policy explicitly permits it.
- Do not describe a standalone artifact as stored in Gateway Conversations.

The endpoint supports the managed `slides` and `report` types. Do not invent arbitrary `c1_artifact_type` values or route custom application artifacts through this contract.

## Verify

1. Confirm the route calls `/v1/artifact/chat/completions`, not the Embed Chat Completions or Responses endpoint.
2. Confirm `metadata.thesys` is JSON-encoded exactly once and contains the intended id and supported artifact type.
3. Generate one slide deck and one report; verify their roots and render each with the matching viewer.
4. Stream a program and confirm progressive rendering receives the correct `isStreaming` state.
5. Edit a stored artifact and confirm the authoritative prior program is sent as the assistant message with `is_edit: true`.
6. Verify unauthorized users cannot load or edit another user's stored program or artifact id.
7. Test missing configuration, invalid types, oversized inputs, empty output, Gateway failures, cancellation, and stream closure.
8. Run the host formatter, typecheck, tests, and production build.

## Supplementary First-Party References

The standalone implementation examples are included above. These sources cover related in-conversation workflows and do not replace the standalone instructions:

- `https://www.openui.com/docs/gateway/api/responses`
- Current in-conversation implementation: `https://github.com/thesysdev/openui/blob/main/templates/openui-cloud/src/app/api/chat/route.ts`
- Current managed renderer wiring: `https://github.com/thesysdev/openui/blob/main/templates/openui-cloud/src/components/cloud-chat.tsx`
