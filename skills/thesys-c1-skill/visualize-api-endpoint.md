# Visualize Endpoint

The **Visualize** endpoint is specifically designed for the "C1 as Presentation Layer" integration pattern. It takes data from your existing LLM and generates the optimal UI representation for it.

**Endpoint:** `POST https://api.thesys.dev/v1/visualize/chat/completions`

## Key Characteristics

- **Data-to-UI conversion**: Given a data object, generates the best UI representation for it
- **No tool calling**: Unlike the Embed endpoint, Visualize does NOT support tool calling - it's purely for UI generation
- **Semantic design steering**: Customize the UI style and behavior via system prompts
- **Visual customization**: Style the output using [Crayon](https://crayonai.org)

## When to Use Visualize vs Embed

| Feature | Embed (`/v1/embed`) | Visualize (`/v1/visualize`) |
|---------|---------------------|----------------------------|
| Tool Calling | ✅ Supported | ❌ Not supported |
| Use Case | Gateway LLM (handles logic + UI) | Presentation Layer (UI only) |
| Best For | New applications, full C1 adoption | Existing LLM pipelines |

## Two-Step Visualize Pattern

Use Visualize when you want to keep your existing LLM infrastructure but add Generative UI capabilities:

```typescript
// Step 1: Get text response from your existing LLM
const textResponse = await yourExistingLLM.generate({
  messages: [{ role: "user", content: userQuery }],
  tools: yourExistingTools,
});

// Step 2: Convert to interactive UI with C1 Visualize
const visualizeClient = new OpenAI({
  baseURL: "https://api.thesys.dev/v1/visualize",
  apiKey: process.env.THESYS_API_KEY,
});

const uiResponse = await visualizeClient.chat.completions.create({
  model: "c1/anthropic/claude-sonnet-4/v-20251230",
  messages: [
    { role: "system", content: "Generate a clean, interactive UI for this data." },
    { role: "user", content: textResponse }
  ],
  stream: true,
});
```

## Request & Response

**Request**: Supports both streaming and non-streaming payloads
- Accepts data via the `messages` property
- Does not accept the `tools` property

**Response**: Returns streaming chunks in streaming mode or a message object in non-streaming mode

## Trade-offs

**Pros:**
- Works with any LLM (custom models, fine-tuned models, etc.)
- Minimal changes to existing infrastructure
- Separates business logic from presentation

**Cons:**
- Additional latency (must wait for full text response before UI generation starts)
- Less context for UI generation (C1 doesn't see your tools/state)

For most applications, using C1 as the Gateway LLM (via the Embed endpoint) is preferred as it avoids extra latency and provides better context for UI generation.
