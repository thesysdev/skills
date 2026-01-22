# Migrating to Generative UI

This is an advanced guide that assumes you already have a working text-based LLM application. If you are starting from scratch, see the Quickstart guide in the main skill.md file.

C1 by Thesys is designed to be a drop-in replacement for OpenAI's API. By making a few changes to your existing application, you can start using C1 to upgrade your regular LLM workflows to Generative UI.

---

## Step 1: Change baseURL to C1

The first step is to change the OpenAI SDK instantiation to point to C1 rather than the default OpenAI endpoint. This change is generally only required in one place in your application - where you instantiate the Gateway LLM (the first LLM that is invoked when the user interacts with your application).

```typescript
const client = new OpenAI({
  baseURL: "https://api.thesys.dev/v1/embed",
  apiKey: process.env.THESYS_API_KEY,
});
```

And then change the model name to use one of the supported models:

```typescript
const response = await client.chat.completions.create({
  // model: "gpt-4o",  // Old model
  model: "c1/anthropic/claude-sonnet-4/v-20251230",  // New C1 model
  messages: [{ role: "user", content: "Hello" }],
});
```

Once you've done this successfully, you will be able to see your application rendering Thesys DSL responses as plain text.

---

## Step 2: Add the C1Component to Your Application

Now that we are able to see the DSL responses, we can add the C1Component to our application to render the DSL responses as a live UI.

First, install the necessary packages for C1 integration:

```bash
npm install --save @thesysai/genui-sdk @crayonai/react-ui @crayonai/stream @crayonai/react-core
```

Now simply replace the `Markdown` component with the `C1Component` component:

```tsx
// Before
import { Markdown } from "react-markdown";

// After
import { C1Component, ThemeProvider } from "@thesysai/genui-sdk";
import "@crayonai/react-ui/styles/index.css";

export default function App() {
  // other app related logic

  return (
    // <Markdown>{response}</Markdown>  // Old approach
    <ThemeProvider>
      <C1Component c1Response={response} />
    </ThemeProvider>
  );
}
```

The preferred font family for C1 is `Inter`. You can import this font in your CSS file as follows:

```css
@import url("https://fonts.googleapis.com/css2?family=Inter:wght@100;200;300;400;500;600;700;800;900&display=swap");
```

At this point, you should be able to see your application rendering the DSL responses as a live micro-frontend.

---

## Step 3: Streaming the Responses (Optional)

To improve the user experience, you can stream the responses from the backend to the frontend. This reduces the perceived latency of the application and makes it feel more responsive.

`<C1Component />` supports streaming the responses and progressively rendering the UI by passing the `isStreaming` prop. This prop should be set to `true` when the response is being streamed and `false` when the response is done streaming.

```tsx
<C1Component
  c1Response={response}
  isStreaming={isStreaming}
/>
```

You can easily implement this by storing `isStreaming` as a state variable that is set to `true` when the `fetch` request is made and set to `false` when the request is complete.

### Example Implementation

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

At this point, you should be able to see your application streaming the responses and progressively rendering the UI.

---

## Step 4: Enabling Interactivity

At this stage, you should be able to see the application rendering buttons and forms but they won't be functional. In order to make them functional, you need to pass the `onAction` callback to the `C1Component` and implement the logic to handle the action.

In most cases, you will want to treat the `onAction` callback as a way to trigger the next turn of the conversation. This means it will function as if the user had typed in a query and hit enter.

```tsx
<C1Component
  c1Response={response}
  isStreaming={isStreaming}
  onAction={({ llmFriendlyMessage, humanFriendlyMessage }) => {
    // - Trigger the next turn of the LLM
    // - Send llmFriendlyMessage to the LLM so that the next turn can be triggered
    // - Show humanFriendlyMessage in the chat UI as the user's message
  }}
/>
```

### Understanding Message Types

It is important to have this distinction to improve the user experience:

- **`llmFriendlyMessage`**: Technical message for the LLM (e.g., form data as JSON). Would not be suitable for humans to read in most cases.
- **`humanFriendlyMessage`**: User-friendly text for display in the chat UI.

Your message store probably won't have the `llmFriendlyMessage` in it, but in the long term, it is a good idea to store it in the message store.

Once you've implemented this, all the buttons and forms should be functional.

---

## Step 5: Saving Form Values

While your chat interface is working, the form values don't persist when you refresh the page. This is because the form values are also stored in the `c1Response` object. To enable persistence, you can pass the `updateMessage` callback to the `C1Component` and implement the logic to persist the form values.

Typically `PUT` or `PATCH` requests are used to update the message in the database. You might have to implement this endpoint in your backend if it's not already implemented.

```tsx
<C1Component
  c1Response={response}
  isStreaming={isStreaming}
  onAction={({ llmFriendlyMessage, humanFriendlyMessage }) => {...}}
  updateMessage={(message) => {
    // Update the message to the database
  }}
/>
```

### Example Database Update Endpoint

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

Once you've implemented this, you should be able to see UI generations, have functional buttons and forms, and the form values would persist when you refresh the page.

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

## Example: HuggingFace Chat UI with C1

C1Component is a powerful abstraction that can be used to embed Generative UI within your application. Below is an example of HuggingFace Chat UI integrated with C1Component, demonstrating how existing chat interfaces can be enhanced with Generative UI capabilities.

---

## Resources

- **Video Tutorial**: [How to integrate C1 in just 2 steps](https://www.youtube.com/watch?v=5VvMP1fiMc8)
- **Example**: [HuggingFace Chat UI with C1](https://github.com/rabisg/chat-ui)
- **Full Examples**: https://github.com/thesysdev/examples
