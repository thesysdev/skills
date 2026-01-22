# Migrating to Generative UI

This guide walks through migrating an existing text-based LLM application to Thesys C1 Generative UI.

---

## Overview

C1 is designed as a drop-in replacement for OpenAI's API. Migration typically requires:

1. Change the API base URL
2. Replace Markdown rendering with `<C1Component>`
3. Add streaming support (optional but recommended)
4. Enable interactivity with `onAction`
5. Persist form values with `updateMessage`

---

## Step 1: Change Base URL

Update your OpenAI client to point to C1:

### Before

```typescript
const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

const response = await client.chat.completions.create({
  model: "gpt-4o",
  messages: [{ role: "user", content: "Hello" }],
});
```

### After

```typescript
const client = new OpenAI({
  baseURL: "https://api.thesys.dev/v1/embed",
  apiKey: process.env.THESYS_API_KEY,
});

const response = await client.chat.completions.create({
  model: "c1/anthropic/claude-sonnet-4/v-20251230",
  messages: [{ role: "user", content: "Hello" }],
});
```

At this point, your application will display C1 DSL responses as plain text.

---

## Step 2: Add C1Component

Replace your Markdown renderer with `<C1Component>`:

### Install Dependencies

```bash
npm install @thesysai/genui-sdk @crayonai/react-ui @crayonai/stream @crayonai/react-core
```

### Before

```tsx
import { Markdown } from "react-markdown";

export default function App() {
  return <Markdown>{response}</Markdown>;
}
```

### After

```tsx
import { C1Component, ThemeProvider } from "@thesysai/genui-sdk";
import "@crayonai/react-ui/styles/index.css";

export default function App() {
  return (
    <ThemeProvider>
      <C1Component c1Response={response} />
    </ThemeProvider>
  );
}
```

### Add Inter Font

C1 uses Inter by default. Add to your CSS:

```css
@import url("https://fonts.googleapis.com/css2?family=Inter:wght@100;200;300;400;500;600;700;800;900&display=swap");
```

Now you'll see C1-generated UIs instead of raw DSL text.

---

## Step 3: Enable Streaming (Recommended)

Streaming improves perceived performance by progressively rendering UI.

### Backend: Stream Responses

```typescript
import { transformStream } from "@crayonai/stream";

const llmStream = await client.chat.completions.create({
  model: "c1/anthropic/claude-sonnet-4/v-20251230",
  messages: [...],
  stream: true,
});

const responseStream = transformStream(
  llmStream,
  (chunk) => chunk.choices[0].delta.content,
) as ReadableStream;

return new Response(responseStream, {
  headers: {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache, no-transform",
    Connection: "keep-alive",
  },
});
```

### Frontend: Handle Streaming

```tsx
function Chat() {
  const [response, setResponse] = useState("");
  const [isStreaming, setIsStreaming] = useState(false);

  const sendMessage = async (message: string) => {
    setIsStreaming(true);
    setResponse("");

    const res = await fetch("/api/chat", {
      method: "POST",
      body: JSON.stringify({ message }),
    });

    const reader = res.body?.getReader();
    const decoder = new TextDecoder();

    while (reader) {
      const { done, value } = await reader.read();
      if (done) break;
      setResponse((prev) => prev + decoder.decode(value));
    }

    setIsStreaming(false);
  };

  return (
    <ThemeProvider>
      <C1Component
        c1Response={response}
        isStreaming={isStreaming}
      />
    </ThemeProvider>
  );
}
```

---

## Step 4: Enable Interactivity

C1 generates buttons and forms, but they need `onAction` to function:

```tsx
<C1Component
  c1Response={response}
  isStreaming={isStreaming}
  onAction={({ llmFriendlyMessage, humanFriendlyMessage }) => {
    // llmFriendlyMessage: Send to LLM for next turn
    // humanFriendlyMessage: Display in chat UI
    
    // Trigger next conversation turn
    sendMessage(llmFriendlyMessage);
    
    // Optionally show user message
    addToMessageHistory({
      role: "user",
      content: humanFriendlyMessage,
    });
  }}
/>
```

### Understanding Message Types

- **`llmFriendlyMessage`**: Technical message for the LLM (e.g., form data as JSON)
- **`humanFriendlyMessage`**: User-friendly text for display

Store `llmFriendlyMessage` for conversation history, but display `humanFriendlyMessage`.

---

## Step 5: Persist Form Values

Form values are stored in the C1 response. Use `updateMessage` to persist:

```tsx
<C1Component
  c1Response={response}
  isStreaming={isStreaming}
  onAction={...}
  updateMessage={(updatedResponse) => {
    // updatedResponse contains form values merged into DSL
    saveToDatabase({
      messageId: currentMessageId,
      content: updatedResponse,
    });
  }}
/>
```

### Database Update Endpoint

Create a PATCH/PUT endpoint for message updates:

```typescript
// app/api/messages/[id]/route.ts
export async function PATCH(req, { params }) {
  const { content } = await req.json();
  await db.messages.update({
    where: { id: params.id },
    data: { content },
  });
  return Response.json({ success: true });
}
```

---

## Complete Migration Example

### Before: Text-Based Chat

```tsx
// Old implementation
import { Markdown } from "react-markdown";

function OldChat() {
  const [messages, setMessages] = useState([]);
  
  const sendMessage = async (text) => {
    const res = await fetch("/api/chat", {
      method: "POST",
      body: JSON.stringify({ messages: [...messages, { role: "user", content: text }] }),
    });
    const data = await res.json();
    setMessages([...messages, { role: "user", content: text }, data.message]);
  };

  return (
    <div>
      {messages.map((m, i) => (
        <div key={i}>
          {m.role === "assistant" ? <Markdown>{m.content}</Markdown> : m.content}
        </div>
      ))}
    </div>
  );
}
```

### After: Generative UI Chat

```tsx
// New implementation with C1
import { C1Component, ThemeProvider } from "@thesysai/genui-sdk";
import "@crayonai/react-ui/styles/index.css";

function NewChat() {
  const [messages, setMessages] = useState([]);
  const [currentResponse, setCurrentResponse] = useState("");
  const [isStreaming, setIsStreaming] = useState(false);

  const sendMessage = async (text) => {
    setIsStreaming(true);
    setCurrentResponse("");
    
    // Add user message
    const newMessages = [...messages, { role: "user", content: text }];
    setMessages(newMessages);

    const res = await fetch("/api/chat", {
      method: "POST",
      body: JSON.stringify({ messages: newMessages }),
    });

    const reader = res.body?.getReader();
    const decoder = new TextDecoder();
    let fullResponse = "";

    while (reader) {
      const { done, value } = await reader.read();
      if (done) break;
      fullResponse += decoder.decode(value);
      setCurrentResponse(fullResponse);
    }

    setMessages([...newMessages, { role: "assistant", content: fullResponse }]);
    setIsStreaming(false);
  };

  return (
    <ThemeProvider>
      <div>
        {messages.map((m, i) => (
          <div key={i}>
            {m.role === "assistant" ? (
              <C1Component c1Response={m.content} />
            ) : (
              <div className="user-message">{m.content}</div>
            )}
          </div>
        ))}
        
        {isStreaming && (
          <C1Component
            c1Response={currentResponse}
            isStreaming={true}
          />
        )}
      </div>
      
      <input
        onKeyDown={(e) => {
          if (e.key === "Enter") {
            sendMessage(e.target.value);
            e.target.value = "";
          }
        }}
      />
    </ThemeProvider>
  );
}
```

---

## Migration Checklist

- [ ] Update OpenAI client `baseURL` to `https://api.thesys.dev/v1/embed`
- [ ] Set `THESYS_API_KEY` environment variable
- [ ] Change model to C1 model (e.g., `c1/anthropic/claude-sonnet-4/v-20251230`)
- [ ] Install SDK packages: `@thesysai/genui-sdk @crayonai/react-ui @crayonai/stream @crayonai/react-core`
- [ ] Import CSS: `@crayonai/react-ui/styles/index.css`
- [ ] Add Inter font to CSS
- [ ] Replace Markdown component with `<C1Component>`
- [ ] Wrap with `<ThemeProvider>`
- [ ] Add `isStreaming` prop for streaming support
- [ ] Implement `onAction` callback for interactivity
- [ ] Implement `updateMessage` callback for persistence
- [ ] Test all interactive elements (buttons, forms, inputs)

---

## Common Issues

### DSL Showing as Text

Ensure you're using `<C1Component>` wrapped in `<ThemeProvider>` and CSS is imported.

### Buttons Not Working

Add `onAction` callback to handle interactions.

### Form Values Lost on Refresh

Implement `updateMessage` to persist state to database.

### Styling Doesn't Match Brand

Use `<ThemeProvider>` with custom theme object.

### Components Not Rendering

Check that SDK version matches model version. See API changelog for minimum SDK versions.

---

## Resources

- **Video Tutorial**: [How to integrate C1 in just 2 steps](https://www.youtube.com/watch?v=5VvMP1fiMc8)
- **Example**: [HuggingFace Chat UI with C1](https://github.com/rabisg/chat-ui)
- **Full Examples**: https://github.com/thesysdev/examples
