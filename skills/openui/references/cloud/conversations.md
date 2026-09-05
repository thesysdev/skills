# Integrate OpenUI Cloud Conversations

Read [existing-project.md](existing-project.md) first. Use this reference when an application needs persistent named Responses threads, conversation-item access, browser thread storage, scoped frontend tokens, or multi-user/multi-app isolation.

Conversations is a storage and identity plane, not a generation protocol. It integrates with the Responses API. Embed Chat Completions applications keep and resend their own `messages` history; do not add the Conversations API to them without an explicit protocol and storage migration.

## Choose the State Model First

Use Conversations only for the named-thread Responses pattern:

```ts
const response = await embedClient.responses.create({
  model,
  conversation: conversationId,
  input: latestTurn,
  instructions,
  store: true,
  stream: true,
});
```

The application sends only the new turn because Cloud supplies the earlier items from the named conversation. Do not also resend full history.

Responses can instead use application-owned full `input` history or a `previous_response_id` chain. Those modes do not require a Conversations client, frontend-token route, `useOpenuiCloudStorage()`, or conversation ownership checks. Read [responses.md](responses.md) for all three generation patterns.

## Choose the Access Plane

Use the access plane that matches the caller:

| Caller | Credential | Recommended path |
| --- | --- | --- |
| Application server | Server-side `THESYS_API_KEY` | Stock OpenAI SDK with base URL `https://api.thesys.dev/v1`, or direct server calls to `/v1/conversations*` |
| Browser `AgentInterface` | Short-lived scoped frontend token | `useOpenuiCloudStorage()`; do not expose the server key or proxy every browser storage read by default |

The generation proxy remains an application server route using the server key. A browser frontend token does not authorize that proxy automatically.

## Work with Conversations and Items

Configure a dedicated SDK client for the Conversations base URL:

```ts
import OpenAI from "openai";

const conversationClient = new OpenAI({
  apiKey: process.env.THESYS_API_KEY,
  baseURL: "https://api.thesys.dev/v1",
});
```

Create the named conversation before the first stored response, then use its id as the Responses `conversation` value:

```ts
const conversation = await conversationClient.conversations.create({
  metadata: { workspace: "acme" },
});
```

Inspect stored messages, tool calls, and tool outputs through the item collection:

```ts
const page = await conversationClient.conversations.items.list(conversation.id, {
  order: "asc",
  limit: 100,
});

for (const item of page.data) {
  console.log(item.type, item.id);
}
```

Current operations are:

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` / `POST` | `/v1/conversations` | List or create conversations |
| `GET` / `POST` / `DELETE` | `/v1/conversations/{conversation_id}` | Retrieve, update, or delete a conversation |
| `GET` / `POST` | `/v1/conversations/{conversation_id}/items` | List or add items |
| `GET` / `DELETE` | `/v1/conversations/{conversation_id}/items/{item_id}` | Retrieve or delete one item |

Prefer the installed OpenAI SDK methods for these operations. Verify current pagination, update fields, item unions, and delete behavior before implementing custom raw HTTP helpers.

## Connect Agent Interface Storage

In a React client module, use Cloud storage for thread listing, item reload, and managed artifact state:

```tsx
"use client";

import { AgentInterface } from "@openuidev/react-ui";
import { useOpenuiCloudStorage } from "@openuidev/thesys";

export function CloudAgent() {
  const storage = useOpenuiCloudStorage({
    token: "/api/frontend-token",
    apiBaseUrl: "https://api.thesys.dev",
    features: { artifact: true },
  });

  return <AgentInterface llm={llm} storage={storage} />;
}
```

Keep the `@openuidev/thesys` import in the host framework's client boundary. Preserve the product's existing shell, theme, slots, routing, and authentication guard. Adding Cloud storage does not require replacing a working chat UI or component library.

## Mint Frontend Tokens

The application server mints a short-lived token with `POST https://api.thesys.dev/v1/frontend-tokens`. Derive `user_id` from authenticated server state and optionally include one stable `app_id`. Never accept the authoritative identity from the request body.

```ts
type FrontendToken = { token: string; expires_at: number };

export async function mintCloudFrontendToken(userId: string): Promise<FrontendToken> {
  const apiKey = process.env.THESYS_API_KEY;
  if (!apiKey) throw new Error("THESYS_API_KEY is not configured");

  const response = await fetch("https://api.thesys.dev/v1/frontend-tokens", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${apiKey}`,
    },
    body: JSON.stringify({
      user_id: userId,
      ...(process.env.APP_ID ? { app_id: process.env.APP_ID } : {}),
    }),
    cache: "no-store",
  });

  if (!response.ok) {
    throw new Error(`Cloud frontend-token mint failed with ${response.status}`);
  }

  return (await response.json()) as FrontendToken;
}
```

Protect the host route independently:

```ts
export async function POST(req: Request) {
  const userId = await getAuthenticatedUserId(req);
  if (!userId) {
    return Response.json({ error: { message: "Unauthorized" } }, { status: 401 });
  }

  try {
    const frontendToken = await mintCloudFrontendToken(userId);
    return Response.json(frontendToken, {
      headers: { "Cache-Control": "private, no-store" },
    });
  } catch (error) {
    console.error("[frontend-token] Cloud rejected token mint", error);
    return Response.json(
      { error: { message: "Unable to mint frontend token" } },
      { status: 502 },
    );
  }
}
```

The browser storage client sends the token as `x-thesys-frontend-token` and refreshes it through the configured token route. The server API key must never reach browser code, logs, or generated output.

## Scope Users and Apps

- Use a stable authenticated subject as `user_id`; never use display names, emails that can change, or browser-provided ids when the host has an immutable account id.
- Use one stable `app_id` per product surface when the same organization key serves multiple applications. Do not derive it from the API key because key rotation must not change storage identity.
- Treat a scaffold's `DEMO_USER_ID` as local-demo configuration only. Replace it before production multi-user use.
- Use conversation `metadata` for application data, not as the authorization source for ownership.
- Keep token minting, generation authorization, and any server-side conversation operations on the same identity convention.

The scoped frontend token limits browser conversation and artifact access to its user and optional app. The server key remains organization-level authority.

## Authorize the Generation Route Separately

The browser token protects direct storage calls, but the application `/api/chat` route calls Responses with the server key. Possession of a `threadId` is not proof that the authenticated user owns that conversation.

Use one verified design:

1. **Host-owned mapping:** when the host has a trusted creation or binding path, transactionally store `{ conversationId, ownerUserId, appId? }` and check it before generation.
2. **Documented Cloud membership check:** only when the installed package, current template, or current first-party documentation exposes the exact endpoint, credential, response shape, and pagination behavior for the scoped user.

Do not add a browser route that merely records ownership claimed by the browser. If neither design can establish ownership, keep the production generation route disabled and report the blocker.

The conversation-specific generation route must also:

- authenticate and rate-limit independently from the page and token route;
- treat `threadId` as untrusted and authorize it before the Responses call;
- validate and reconstruct one bounded latest user turn instead of forwarding arbitrary provider items;
- set `conversation: threadId` and `store: true` without resending full history;
- preserve the Responses stream and abort signal.

Read [responses.md](responses.md) for request construction, adapters, tools, artifacts, and streaming behavior.

## Migrate Storage Deliberately

Adding a Cloud generation endpoint does not authorize replacing the host database. When moving an existing application to Conversations:

1. Decide whether Cloud becomes the only durable store or runs beside the existing store during rollout.
2. Keep historical data accessible in its existing store unless a current first-party import API has been verified.
3. Preserve old thread identifiers or maintain an explicit mapping; do not imply that records were imported when they were not.
4. Test user and app isolation before removing the prior storage route.

## Verify

1. Confirm Chat Completions routes do not use the Conversations API.
2. Confirm named Responses calls send only the new turn with `conversation` and `store: true`.
3. Confirm the frontend-token route derives identity from the authenticated session, rate-limits requests, and never exposes `THESYS_API_KEY`.
4. Confirm the browser uses the scoped token and the server generation route separately verifies `threadId` ownership.
5. Create, retrieve, update, and delete a test conversation; list its items with pagination and verify their order.
6. Reload `AgentInterface` and confirm the intended threads, items, and artifacts return.
7. Test two users and, when applicable, two `app_id` values for disjoint thread lists and forbidden cross-user generation.
8. Test token expiry/refresh, missing configuration, Cloud failures, and logged-out calls.
9. Run the host formatter, typecheck, tests, and production build.

## First-Party References

- `https://www.openui.com/docs/openui-cloud/api/conversations`
- `https://www.openui.com/docs/openui-cloud/api/responses`
- `https://www.openui.com/docs/openui-cloud/api/overview`
- `https://www.openui.com/docs/openui-cloud/how-it-works`
- `https://www.openui.com/docs/agent/reference/agentinterface-props`
