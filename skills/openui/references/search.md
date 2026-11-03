# Search the web or images without a model (Search API)

Use this when the app needs search results in its own code: a search box, a research step in a pipeline, or results to cache or pass to any model. If the model should decide when to search during a conversation, use the hosted `web_search` and `image_search` tools instead; see [gateway-responses.md](gateway-responses.md#hosted-tools).

Search needs `THESYS_API_KEY` on the server, created by the user (see [SKILL.md](../SKILL.md#never-handle-api-keys)). Call it from the server, never from the browser.

For pricing, link https://www.openui.com/docs/gateway/pricing-credits.

## Call the API

The endpoints take plain JSON over HTTP. They are not OpenAI-compatible, so use `fetch` or any HTTP client, not the OpenAI SDK.

```ts
async function search(kind: "web" | "images", body: Record<string, unknown>) {
  const res = await fetch(`https://api.thesys.dev/v1/search/${kind}`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.THESYS_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });
  if (!res.ok) {
    const { error } = await res.json();
    throw new Error(`${res.status} ${error.type}: ${error.message}`);
  }
  return res.json();
}

const web = await search("web", { query: "nestjs 11 release notes", num_results: 5 });
const images = await search("images", { query: "golden gate bridge at night", num_results: 3 });
```

Request fields:

- `POST https://api.thesys.dev/v1/search/web`: `query` (required, 1 to 2,048 characters), `num_results` (1 to 10, default 5), `include_domains` (an array of domains to keep), `user_location` (a two-letter country code such as `"FR"`; anything else is a `400`).
- `POST https://api.thesys.dev/v1/search/images`: `query` and `num_results`.

The response is `{ query, results, usage: { search_units: 1 } }`:

- Web results: `title`, `url`, and, when known, `text`, `published_date`, `author`, `image`, `favicon`.
- Image results: `title`, `image_url`, `page_url`, `source`, and, when known, `width`, `height`, `thumbnail_url`, `mime_type`.

A search can return fewer results than `num_results`. Handle every optional field being absent.

Errors look like `{ error: { message, type, param } }`:

- `400`: the body is invalid. `message` is a list of every problem.
- `401`: the key is missing or invalid.
- `429`: the organization is out of credits, has reached its spend limit, or has billing suspended. Retrying does not help; tell the user to add credits.
- `500`: the search provider failed. Nothing is charged; retry once.

## Use the results

- **Grounding a model:** put each result's `title`, `url`, and trimmed `text` into the prompt, and ask the model to cite the URLs it uses.
- **Showing images:** show `thumbnail_url` (or `image_url` when there is no thumbnail) and link it to `page_url`. The image belongs to that page's site.

## Verify

1. A web search returns `200` with results and `usage.search_units` set to 1.
2. An image search returns `image_url`, `page_url`, and `source` for each result.
3. A body with `num_results: 20` returns `400`, and the app shows a useful error.
4. `THESYS_API_KEY` appears in no client file.

## Sources

- [Search API](https://www.openui.com/docs/gateway/api/search)
- [Hosted tools](https://www.openui.com/docs/gateway/api/responses/hosted-tools)
- [Pricing](https://www.openui.com/docs/gateway/pricing-credits)
