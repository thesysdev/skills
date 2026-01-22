# Custom Actions, Components & Thinking States

This guide covers implementing custom actions, creating custom components, and displaying thinking states in C1.

---

## Custom Actions

Custom actions allow you to go beyond built-in behaviors and trigger your application's unique functions directly from C1-generated UI. This enables powerful workflows such as:

- Downloading a generated report
- Opening a product-specific checkout modal
- Triggering functions like creating a new project or sending an email

### Step 1: Define the Custom Action (Backend)

To make the LLM aware of your custom action, define its name and parameters by passing a `c1_custom_actions` object in the `metadata` of your API call. Use a schema library like Zod (TypeScript) or Pydantic (Python) to define your action's parameters:

```typescript
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://api.thesys.dev/v1/embed",
  apiKey: process.env.THESYS_API_KEY,
});

const messages = [
  { role: "system", content: "When the user asks to download data, offer them a 'download_report' button." },
  { role: "user", content: "Can I get a copy of our quarterly sales data?" }
];

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
            .describe("The specific quarter for the report, e.g., 'Q3 2025'."),
        })),
      }
    }),
  },
});
```

When the LLM generates a UI that includes a "Download Report" button, it will attach the action type `download_report` and the corresponding parameters to it.

### Step 2: Handle the Custom Action (Frontend)

#### Using `<C1Component>`

Use the `onAction` callback and add a `case` to your `switch` statement for your custom action type:

```tsx
<C1Component
  c1Response={c1Response}
  isStreaming={isLoading}
  onAction={(event) => {
    switch (event.type) {
      // Your custom action case
      case "download_report":
        // Optionally add a message to the UI to confirm the action
        pushUserMessageToChat(`Downloading ${event.params.reportType} report...`);
        // Trigger your application's download logic with the action's parameters
        downloadReport(event.params);
        break;

      // Built-in action cases
      case "open_url":
        window.open(event.params.url, "_blank", "noopener,noreferrer");
        break;

      case "continue_conversation":
      default:
        const { llmFriendlyMessage, humanFriendlyMessage } = event.params;
        pushUserMessageToChat(humanFriendlyMessage);
        callApi(llmFriendlyMessage);
        break;
    }
  }}
/>
```

#### Using `<C1Chat>`

The `<C1Chat>` component handles built-in actions (`continue_conversation` and `open_url`) automatically, so its `onAction` prop is used exclusively for your custom actions:

```tsx
<C1Chat
  apiUrl="/api/chat"
  onAction={(event) => {
    // C1Chat handles 'continue_conversation' and 'open_url',
    // so you only need to handle your custom actions.
    switch (event.type) {
      case "download_report":
        // Trigger your application's logic
        downloadReport(event.params);
        break;
    }
  }}
/>
```

#### Using `useThreadManager`

If you are using persistence with the `useThreadManager` hook, pass the `onAction` callback directly to the hook:

```tsx
const threadManager = useThreadManager({
  threadListManager,
  apiUrl: "/api/chat",
  onAction: (event) => {
    if (event.type === "download_report") {
      downloadReport(event.params);
    }
  },
});

<C1Chat threadManager={threadManager} threadListManager={threadListManager} />
```

### Action Event Properties

```typescript
interface ActionEvent {
  type: string;                  // Action name (e.g., "download_report")
  params: Record<string, any>;   // Parameters from the action
  llmFriendlyMessage: string;    // Message to send back to LLM
  humanFriendlyMessage: string;  // Message to display to user
}
```

---

## Custom Components

Custom components allow you to provide your own React components for the C1 API to render, enabling tailor-made interfaces for your application.

### Minimum Versions Required

| Package             | Minimum Version |
|---------------------|-----------------|
| @crayonai/react-ui  | 0.8.31          |
| @thesysai/genui-sdk | 0.6.34          |
| C1-Version          | v-20250915      |

### Step 1: Define Your React Components

Create your custom React component using `useOnAction` for handling user interactions and `useC1State` for managing component state:

```tsx
// src/app/components.tsx
import { useOnAction, useC1State } from "@thesysai/genui-sdk";

interface Flight {
  flightNumber: string;
  departure: string;
  arrival: string;
  layover?: string;
  price: number;
}

export const FlightList = ({ flights }: { flights: Flight[] }) => {
  const onAction = useOnAction();

  // State management for component's internal state
  const { getValue, setValue } = useC1State("FlightList");

  return (
    <div>
      <h3>Available Flights</h3>
      {flights.map((flight) => (
        <FlightCard
          key={flight.flightNumber}
          flight={flight}
          onSelect={() =>
            onAction(
              "Select Flight",                              // Human-friendly message
              `User selected flight ${flight.flightNumber}` // LLM-specific message with context
            )
          }
        />
      ))}
    </div>
  );
};
```

### Step 2: Define the Component Prop Schema

Define a Zod-based schema to inform C1 about the expected props for your component:

```typescript
// src/app/api/chat/route.ts
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";

// Schema of a flight
const flightSchema = z
  .object({
    flightNumber: z.string(),
    departure: z.string(),
    arrival: z.string(),
    layover: z.string().optional(),
    price: z.number(),
  })
  .describe(
    "Represents a single flight option including schedule, price, and stops."
  );

// Schema of the FlightList component props
const FlightListSchema = z
  .object({
    flights: z.array(flightSchema),
  })
  // Always include descriptive text for the LLM to understand the component
  .describe(
    "Displays a list of available flights. Renders rich cards with airline, route, times, stops, and price. Includes a clear 'Select' action for each item."
  );
```

### Step 3: Pass the Schema to the C1 API

Convert the schema to JSON and send it in your C1 API payload using `c1_custom_components`:

```typescript
// src/app/api/chat/route.ts
import OpenAI from "openai";

// Schema to be passed to the C1 API
// Key must match the React component name exactly
const CUSTOM_COMPONENT_SCHEMAS = {
  FlightList: zodToJsonSchema(FlightListSchema),
  // ... other custom components
};

export async function POST(req: NextRequest) {
  const client = new OpenAI({
    baseURL: "https://api.thesys.dev/v1/embed/",
    apiKey: process.env.THESYS_API_KEY,
  });

  const llmStream = client.beta.chat.completions.runTools({
    model: "c1/anthropic/claude-sonnet-4/v-20250915",
    messages: [...],
    stream: true,
    // Pass custom component schema
    metadata: {
      thesys: JSON.stringify({
        c1_custom_components: CUSTOM_COMPONENT_SCHEMAS,
      }),
    },
  });

  // ... stream handling
}
```

### Step 4: Pass React Components to GenUI SDK for Rendering

#### With `<C1Chat>`

```tsx
// src/app/page.tsx
import { C1Chat } from "@thesysai/genui-sdk";
import { FlightList } from "./components";

export default function Home() {
  return (
    <C1Chat
      apiUrl="/api/chat"
      customizeC1={{
        customComponents: { FlightList },
      }}
    />
  );
}
```

#### With `<C1Component>`

```tsx
import { C1Component } from "@thesysai/genui-sdk";
import { FlightList } from "./components";

<C1Component
  c1Response={response}
  isStreaming={isLoading}
  customComponents={{ FlightList }}
  onAction={handleAction}
/>
```

### Custom Component Hooks Reference

#### `useOnAction`

Returns a callback to record a user action with both a user-facing label and an LLM-oriented description.

```typescript
const onAction = useOnAction();

// Usage
onAction(humanFriendlyMessage, llmFriendlyMessage);
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `humanFriendlyMessage` | `string` | Visible to the user; concise, human-readable label for the action. E.g., "Submit response" |
| `llmFriendlyMessage` | `string` | Sent to the LLM; richer context describing what happened. E.g., "User selected destination: Paris, dates: Jan 1-7" |

#### `useC1State`

Access a named piece of component state with getter and setter helpers.

```typescript
const { getValue, setValue } = useC1State("my-state-key");

// Read state
const currentValue = getValue();

// Update state
setValue(newValue);
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string` | The state field key you wish to read/write |

**Returns:**
- `getValue: () => any` - Function that returns the current state for the given name
- `setValue: (value: any) => void` - Updates the field and triggers any save/persist callbacks

**Benefits of `useC1State`:**
- State tracked automatically by C1
- Persisted via `updateMessage` callback
- Available in action parameters
- Restored when reloading conversations

---

## Thinking States

Thinking states are a great way to let the user know what the model is doing. This is especially useful for long-running tasks or when the user is waiting for a response, such as when the agent is performing a slow tool call.

### Node.js Implementation

#### Step 1: Create a c1Response Object

Use the `makeC1Response` function from `@thesysai/genui-sdk/server` to create a response object and write LLM content to it:

```typescript
// app/api/chat/route.ts
import { NextRequest, NextResponse } from "next/server";
import OpenAI from "openai";
import type { ChatCompletionMessageParam } from "openai/resources.mjs";
import { transformStream } from "@crayonai/stream";
import { getMessageStore } from "./messageStore";
import { makeC1Response } from "@thesysai/genui-sdk/server";

export async function POST(req: NextRequest) {
  const c1Response = makeC1Response();

  const { prompt, threadId, responseId } = (await req.json()) as {
    prompt: ChatCompletionMessageParam;
    threadId: string;
    responseId: string;
  };

  const client = new OpenAI({
    baseURL: "https://api.thesys.dev/v1/embed",
    apiKey: process.env.THESYS_API_KEY,
  });

  const messageStore = getMessageStore(threadId);
  messageStore.addMessage(prompt);

  const llmStream = await client.chat.completions.create({
    model: "c1/anthropic/claude-sonnet-4/v-20251230",
    messages: messageStore.getOpenAICompatibleMessageList(),
    stream: true,
  });

  // Unwrap the OpenAI stream to a C1 stream
  transformStream(
    llmStream,
    (chunk) => {
      const contentDelta = chunk.choices[0].delta.content;
      if (contentDelta) {
        c1Response.writeContent(contentDelta);
      }
      return contentDelta;
    },
    {
      onEnd: ({ accumulated }) => {
        c1Response.end(); // Necessary to stop showing the "loading" state
        const message = accumulated.filter((chunk) => chunk).join("");
        messageStore.addMessage({
          id: responseId,
          role: "assistant",
          content: message,
        });
      },
    }
  ) as ReadableStream<string>;

  return new NextResponse(c1Response.responseStream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache, no-transform",
      Connection: "keep-alive",
    },
  });
}
```

#### Step 2: Write a Thinking State

Add a thinking state using the `writeThinkItem` method on the `c1Response` object:

```typescript
export async function POST(req: NextRequest) {
  const c1Response = makeC1Response();

  // Write thinking state immediately
  c1Response.writeThinkItem({
    title: "Thinking...",
    description: "Diving into the digital depths to craft you an answer.",
  });

  // ... rest of the handler
}
```

#### Step 3: Use Thinking States with Tool Calls (Optional)

For long-running tool calls, add thinking states inside the tool handler. First, define your tool:

```typescript
// app/api/chat/tools.ts
import type { RunnableToolFunctionWithParse } from "openai/lib/RunnableFunction.mjs";
import { z } from "zod";
import { zodToJsonSchema } from "zod-to-json-schema";
import type { JSONSchema } from "openai/lib/jsonschema.mjs";
import Exa from "exa-js";

const exa = new Exa(process.env.EXA_API_KEY!);

export const getWebSearchTool = (
  writeThinkingState: () => void
): RunnableToolFunctionWithParse<{ query: string }> => ({
  type: "function",
  function: {
    name: "webSearch",
    description: "Use this tool to perform a web search.",
    parse: JSON.parse,
    parameters: zodToJsonSchema(
      z.object({
        query: z.string().describe("The query to search for."),
      })
    ) as JSONSchema,
    function: async ({ query }: { query: string }) => {
      writeThinkingState(); // Show thinking state before long operation
      return await exa.search(query, { numResults: 5 });
    },
    strict: true,
  },
});
```

Then pass the `writeThinkingState` function when using the tool:

```typescript
// app/api/chat/route.ts
const llmStream = await client.beta.chat.completions.runTools({
  model: "c1/anthropic/claude-sonnet-4/v-20251230",
  messages: [
    { role: "system", content: systemPrompt },
    ...messageStore.getOpenAICompatibleMessageList(),
  ],
  stream: true,
  tools: [
    getWebSearchTool(() => {
      c1Response.writeThinkItem({
        title: "Searching the web...",
        description: "Scouring the digital universe for the most relevant and up-to-date insights.",
      });
    }),
  ],
  toolChoice: "auto",
});
```

#### Step 4: Add a Custom Think Component (Optional)

Override the default thinking indicator by passing a custom component to `C1Chat` or `useThreadManager`:

```tsx
import { ThinkComponent } from "@thesysai/genui-sdk";
import styles from "./styles.module.css";

const CustomThink: ThinkComponent = ({ thinkItems, thinkingInProgress }) => {
  return (
    <div className={styles.thinkContainer}>
      <div className={styles.thinkTitle}>
        {thinkingInProgress ? "Processing..." : "Processing complete!"}
      </div>
      <div className={styles.thinkItems}>
        {thinkItems.map((item) => (
          <div key={item.title} className={styles.thinkItem}>
            {item.title}
          </div>
        ))}
      </div>
    </div>
  );
};
```

Pass the custom component to `C1Chat`:

```tsx
<C1Chat
  apiUrl="/api/chat"
  customizeC1={{ thinkComponent: CustomThink }}
/>
```

Or to `useThreadManager`:

```tsx
const threadManager = useThreadManager({
  threadListManager,
  apiUrl: "/api/chat",
  customizeC1: { thinkComponent: CustomThink },
});
```

### ThinkComponent Props

| Prop | Type | Description |
|------|------|-------------|
| `thinkItems` | `ThinkItem[]` | Array of thinking state items |
| `thinkingInProgress` | `boolean` | Indicates if a thinking state is active. Use this to display a loader or shimmer while processing. |

### ThinkItem Properties

| Property | Type | Description |
|----------|------|-------------|
| `title` | `string` | The title of the thinking state |
| `content` | `string` | The content/description of the thinking state |
| `ephemeral` | `boolean` | Whether this thinking state should be temporary or persist after the response is done streaming |

### Python Implementation

#### FastAPI with Decorator

```python
# main.py
from pydantic import BaseModel
from fastapi import FastAPI
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
@with_c1_response()  # Decorator adds c1_response in context and returns stream
async def chat(request: ChatRequest):
    await generate_llm_response(request)

async def generate_llm_response(request: ChatRequest):
    # Write thinking state
    await write_think_item(
        title="Thinking...",
        description="Diving into the digital depths to craft you an answer."
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

    # Get full response to store for message history
    assistant_message_for_history = get_assistant_message()
```

#### Thinking States with Python Tool Calls

```python
async def web_search(query: str):
    await write_think_item(
        title="Searching the web...",
        description=f"Looking for information on '{query}'"
    )
    # ... perform web search
    results = await some_search_api(query)
    return results
```

#### Framework-Independent Python

```python
import asyncio
from thesys_genui_sdk import C1Response
import openai
import os

openai_client = openai.OpenAI(
    api_key=os.getenv("THESYS_API_KEY"),
    base_url="https://api.thesys.dev/v1/embed",
)

async def generate_response(c1_response: C1Response, prompt: str):
    await c1_response.write_think_item(
        title="Thinking...",
        description="Diving into the digital depths to craft you an answer."
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
c1Response.writeContent(content: string)           // Pipe C1 API response
c1Response.writeCustomMarkdown(markdown: string)   // Add custom markdown
c1Response.writeThinkItem({ title, description })  // Add thinking state
c1Response.end()                                   // Signal stream complete
c1Response.responseStream                          // ReadableStream to return
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
