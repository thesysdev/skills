# Custom Actions, Components & Thinking States

This guide covers implementing custom actions, creating custom components, and displaying thinking states.

---

## Custom Actions

Custom actions connect C1-generated UI to your application logic—downloading reports, opening modals, triggering functions.

### Backend Definition

Define actions using `c1_custom_actions` in the API call metadata:

```typescript
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://api.thesys.dev/v1/embed",
  apiKey: process.env.THESYS_API_KEY,
});

const response = await client.chat.completions.create({
  model: "c1/anthropic/claude-sonnet-4/v-20251230",
  messages: messages,
  metadata: {
    thesys: JSON.stringify({
      c1_custom_actions: {
        download_report: zodToJsonSchema(z.object({
          reportType: z.enum(["sales", "marketing", "inventory"])
            .describe("The category of the report."),
          format: z.enum(["csv", "pdf"]).default("pdf")
            .describe("The file format for the download."),
          quarter: z.string().optional()
            .describe("The specific quarter, e.g., 'Q3 2025'."),
        })),
        open_modal: zodToJsonSchema(z.object({
          modalType: z.string().describe("Type of modal to open"),
          data: z.any().optional().describe("Data to pass to modal"),
        })),
      }
    }),
  },
});
```

### Frontend Handling

Handle actions with `onAction` callback:

```tsx
<C1Component
  c1Response={c1Response}
  isStreaming={isLoading}
  onAction={(event) => {
    switch (event.type) {
      case "download_report":
        downloadReport(event.params);
        // Optionally continue conversation
        sendMessage(event.llmFriendlyMessage);
        break;
      case "open_modal":
        openModal(event.params.modalType, event.params.data);
        break;
      default:
        // Handle built-in actions
        sendMessage(event.llmFriendlyMessage);
    }
  }}
/>
```

### With `<C1Chat>`

Use `useThreadManager` hook:

```tsx
const threadManager = useThreadManager({
  threadListManager,
  apiUrl: "/api/chat",
  customizeC1: {
    onAction: (event) => {
      if (event.type === "download_report") {
        downloadReport(event.params);
      }
    },
  },
});

<C1Chat threadManager={threadManager} threadListManager={threadListManager} />
```

### Action Event Properties

```typescript
interface ActionEvent {
  type: string;              // Action name (e.g., "download_report")
  params: Record<string, any>; // Parameters from the action
  llmFriendlyMessage: string;  // Message to send back to LLM
  humanFriendlyMessage: string; // Message to display to user
}
```

---

## Custom Components

Extend C1's component library with your own React components.

### Using `useC1State` Hook

Integrate custom component state into C1's state system:

```tsx
import { useC1State } from '@thesysai/genui-sdk';

function CustomRating() {
  // 'rating-value' is the unique key for this state
  const [rating, setRating] = useC1State('rating-value');

  return (
    <div className="flex gap-1">
      {[1, 2, 3, 4, 5].map((star) => (
        <button
          key={star}
          onClick={() => setRating(star)}
          className={star <= (rating || 0) ? 'text-yellow-500' : 'text-gray-300'}
        >
          ★
        </button>
      ))}
    </div>
  );
}
```

### Benefits of `useC1State`

- State tracked automatically by C1
- Persisted via `updateMessage` callback
- Available in action parameters
- Restored when reloading conversations

### Custom Think Component

Override the default thinking indicator:

```tsx
import { ThinkComponent } from "@thesysai/genui-sdk";

const CustomThink: ThinkComponent = ({ thinkItems, thinkingInProgress }) => {
  return (
    <div className="think-container">
      <div className="think-title">
        {thinkingInProgress ? "Processing..." : "Done!"}
      </div>
      <div className="think-items">
        {thinkItems.map((item) => (
          <div key={item.title} className="think-item">
            <span className="font-bold">{item.title}</span>
            {item.content && <p>{item.content}</p>}
          </div>
        ))}
      </div>
    </div>
  );
};

// With C1Chat
<C1Chat
  apiUrl="/api/chat"
  customizeC1={{ thinkComponent: CustomThink }}
/>

// With useThreadManager
const threadManager = useThreadManager({
  // ...
  customizeC1: { thinkComponent: CustomThink },
});
```

### ThinkComponent Props

```typescript
interface ThinkComponentProps {
  thinkItems: ThinkItem[];      // Array of thinking states
  thinkingInProgress: boolean;  // Whether processing is active
}

interface ThinkItem {
  title: string;      // Title of the thinking state
  content?: string;   // Description/content
  ephemeral?: boolean; // Temporary or persistent
}
```

---

## Thinking States

Display intermediate states while the LLM processes—useful for long-running tasks or tool calls.

### Node.js Implementation

```typescript
// app/api/chat/route.ts
import { NextRequest, NextResponse } from "next/server";
import OpenAI from "openai";
import { transformStream } from "@crayonai/stream";
import { makeC1Response } from "@thesysai/genui-sdk/server";

export async function POST(req: NextRequest) {
  const c1Response = makeC1Response();

  // Write initial thinking state
  c1Response.writeThinkItem({
    title: "Thinking...",
    description: "Analyzing your request.",
  });

  const { prompt, threadId, responseId } = await req.json();

  const client = new OpenAI({
    baseURL: "https://api.thesys.dev/v1/embed",
    apiKey: process.env.THESYS_API_KEY,
  });

  const llmStream = await client.chat.completions.create({
    model: "c1/anthropic/claude-sonnet-4/v-20251230",
    messages: [{ role: "user", content: prompt }],
    stream: true,
  });

  transformStream(
    llmStream,
    (chunk) => {
      const content = chunk.choices[0].delta.content;
      if (content) {
        c1Response.writeContent(content);
      }
      return content;
    },
    {
      onEnd: () => {
        c1Response.end();  // Stop loading state
      },
    }
  );

  return new NextResponse(c1Response.responseStream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache, no-transform",
      Connection: "keep-alive",
    },
  });
}
```

### With Tool Calls

Add thinking states during long-running tool executions:

```typescript
// app/api/chat/tools.ts
export const getWebSearchTool = (writeThinkingState: () => void) => ({
  type: "function",
  function: {
    name: "webSearch",
    description: "Search the web for information.",
    parse: JSON.parse,
    parameters: zodToJsonSchema(
      z.object({ query: z.string().describe("Search query") })
    ),
    function: async ({ query }) => {
      writeThinkingState();  // Show "Searching..." state
      const results = await searchAPI.search(query);
      return JSON.stringify(results);
    },
  },
});

// In route.ts
const llmStream = await client.beta.chat.completions.runTools({
  model: "c1/anthropic/claude-sonnet-4/v-20251230",
  messages: [...],
  stream: true,
  tools: [
    getWebSearchTool(() => {
      c1Response.writeThinkItem({
        title: "Searching the web...",
        description: "Finding relevant information.",
      });
    }),
  ],
});
```

### Python (FastAPI) Implementation

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel
from thesys_genui_sdk.fast_api import with_c1_response
from thesys_genui_sdk.context import write_content, write_think_item, get_assistant_message
import openai
import os

app = FastAPI()

openai_client = openai.OpenAI(
    api_key=os.getenv("THESYS_API_KEY"),
    base_url="https://api.thesys.dev/v1/embed",
)

class ChatRequest(BaseModel):
    prompt: str
    threadId: str
    responseId: str

@app.post("/chat")
@with_c1_response()
async def chat(request: ChatRequest):
    # Write thinking state
    await write_think_item(
        title="Thinking...",
        description="Analyzing your request."
    )
    
    stream = openai_client.chat.completions.create(
        model="c1/anthropic/claude-sonnet-4/v-20251230",
        messages=[{"role": "user", "content": request.prompt}],
        stream=True,
    )
    
    for chunk in stream:
        content = chunk.choices[0].delta.content
        if content:
            await write_content(content)
    
    # Get full response for history
    assistant_message = get_assistant_message()
```

### Framework-Independent Python

```python
import asyncio
from thesys_genui_sdk import C1Response
import openai

openai_client = openai.OpenAI(
    api_key=os.getenv("THESYS_API_KEY"),
    base_url="https://api.thesys.dev/v1/embed",
)

async def generate_response(c1_response: C1Response, prompt: str):
    await c1_response.write_think_item(
        title="Thinking...",
        description="Analyzing your request."
    )
    
    stream = openai_client.chat.completions.create(
        model="c1/anthropic/claude-sonnet-4/v-20251230",
        messages=[{"role": "user", "content": prompt}],
        stream=True,
    )
    
    for chunk in stream:
        content = chunk.choices[0].delta.content
        if content:
            await c1_response.write_content(content)
    
    await c1_response.end()

async def main():
    c1_response = C1Response()
    asyncio.create_task(generate_response(c1_response, "Hello!"))
    
    # Stream response
    async for item in c1_response.stream():
        print(item, end="")

asyncio.run(main())
```

---

## C1 Response Builder Reference

### Node.js SDK (`@thesysai/genui-sdk/server`)

```typescript
import { makeC1Response } from "@thesysai/genui-sdk/server";

const c1Response = makeC1Response();

// Methods:
c1Response.writeContent(content: string)      // Pipe C1 API response
c1Response.writeCustomMarkdown(markdown: string) // Add custom markdown
c1Response.writeThinkItem({ title, description }) // Add thinking state
c1Response.end()                               // Signal stream complete
c1Response.responseStream                      // ReadableStream to return
```

### Python SDK (`thesys_genui_sdk`)

```python
from thesys_genui_sdk import C1Response
# or with FastAPI decorator:
from thesys_genui_sdk.fast_api import with_c1_response
from thesys_genui_sdk.context import write_content, write_think_item, get_assistant_message

# Methods:
await write_content(content)
await write_think_item(title, description)
get_assistant_message()  # Get full response for history
```

---

## Component Metadata Options

Control which components C1 can generate:

### Include Specific Components

```typescript
const response = await client.chat.completions.create({
  model: "c1/anthropic/claude-sonnet-4/v-20251230",
  messages: [...],
  metadata: {
    thesys: JSON.stringify({
      c1_included_components: ["EditableTable"],  // Enable EditableTable
    }),
  },
});
```

### Exclude Components

```typescript
metadata: {
  thesys: JSON.stringify({
    c1_excluded_components: ["Layout"],  // Disable advanced layouts
  }),
}
```

Useful for:
- Copilot or width-constrained UIs
- Controlling which components appear
- Enabling experimental components
