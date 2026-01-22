# Conversational UI & Persistence

This guide covers UI state management, conversational API setup, and persisting conversations to a database.

---

## C1 State Management

### Introduction to C1 State

When C1 generates UI, it tracks values inside components—form inputs, toggles, selections. This is called **C1 State**.

**Why it matters:**
- Form fields hold values until submission
- Inputs/dropdowns remember user selections
- State persists when reloading from history

### Automatic State Management

For built-in C1 components (forms, text inputs, toggles), state is managed automatically. No `useState` or `onChange` handlers needed.

### Persisting State Across Sessions

By default, C1 State is in browser memory and lost on refresh. Use `updateMessage` callback to persist:

```tsx
<C1Component
  c1Response={initialC1Response}
  updateMessage={(updatedC1Response) => {
    // updatedC1Response is the full C1 DSL with state merged
    saveToDatabase({ content: updatedC1Response });
  }}
/>
```

> **Note:** The `updateMessage` callback provides the complete C1 DSL with state already merged. Store this single string.

### State in Custom Components

Use `useC1State` hook to integrate custom component state:

```tsx
import { useC1State } from '@thesysai/genui-sdk';

function CustomToggle() {
  // 'toggle-enabled' is the unique key for this state value
  const { getValue, setValue } = useC1State('toggle-enabled');

  return (
    <button onClick={() => setValue(!getValue())}>
      {getValue() ? 'On' : 'Off'}
    </button>
  );
}
```

---

## Conversational UI with `<C1Chat>`

### What is `<C1Chat>`?

A pre-built component providing:
- Complete chat UI (message list, composer, loading indicators)
- Thread and message management
- Generative UI rendering
- Built-in streaming

### Basic Usage

```tsx
import { C1Chat } from "@thesysai/genui-sdk";
import "@crayonai/react-ui/styles/index.css";

<C1Chat
  apiUrl="/api/chat"
  agentName="My Assistant"
  logoUrl="/logo.png"
  formFactor="full-page"  // or "side-panel" or "bottom-tray"
  theme={{ mode: "dark" }}
/>
```

### Key Props

| Prop | Type | Description |
|------|------|-------------|
| `apiUrl` | `string` | Backend endpoint for chat API |
| `threadManager` | `ThreadManager` | Custom thread management (from `useThreadManager`) |
| `threadListManager` | `ThreadListManager` | Thread list management (from `useThreadListManager`) |
| `formFactor` | `'full-page' \| 'side-panel' \| 'bottom-tray'` | Layout style |
| `agentName` | `string` | Display name in chat |
| `logoUrl` | `string` | Agent logo URL |
| `scrollVariant` | `'once' \| 'user-message-anchor' \| 'always'` | Scroll behavior |
| `customizeC1` | `CustomizeC1Props` | Custom components (e.g., `thinkComponent`) |

---

## Thread & Message State Management

### `useThreadManager`

Controls a single conversation thread:

```typescript
import { useThreadManager, useThreadListManager } from "@thesysai/genui-sdk";

const threadListManager = useThreadListManager({ /* ... */ });

const threadManager = useThreadManager({
  threadListManager,
  loadThread: async (threadId) => {
    // Return messages for the thread
    return await fetchMessagesFromDB(threadId);
  },
  onUpdateMessage: async ({ message }) => {
    // Save message updates (e.g., form submissions)
    await updateMessageInDB(threadId, message);
  },
  apiUrl: "/api/chat",
  // OR use processMessage for custom control:
  // processMessage: async ({ threadId, messages, responseId, abortController }) => {
  //   return await fetch("/api/chat", { ... });
  // }
});
```

### `useThreadListManager`

Manages the list of conversation threads:

```typescript
const threadListManager = useThreadListManager({
  fetchThreadList: async () => {
    return await getThreadListFromDB();
  },
  createThread: async (firstMessage) => {
    return await createThreadInDB(firstMessage.message);
  },
  deleteThread: async (threadId) => {
    await deleteThreadFromDB(threadId);
  },
  updateThread: async (thread) => {
    await updateThreadInDB(thread);
  },
  onSelectThread: (threadId) => {
    router.push(`?threadId=${threadId}`);
  },
  onSwitchToNew: () => {
    router.push('/chat');
  },
});
```

---

## Conversational API (Basic Setup)

Create a chat API endpoint that handles streaming responses from the C1 model. This section shows the basic setup with an in-memory message store.

### Step 1: Install Required Dependencies

```bash
npm install openai @crayonai/stream
```

### Step 2: Create the Message Store

Create a simple in-memory message store to manage conversation history. This store maintains the list of messages for a given `threadId`, including messages that are not sent to the client like tool call messages.

```typescript
// app/api/chat/messageStore.ts
import OpenAI from "openai";

export type DBMessage = OpenAI.Chat.ChatCompletionMessageParam & {
  id?: string;
};

const messagesStore: {
  [threadId: string]: DBMessage[];
} = {};

export const getMessageStore = (threadId: string) => {
  if (!messagesStore[threadId]) {
    messagesStore[threadId] = [];
  }
  const messageList = messagesStore[threadId];
  return {
    addMessage: (message: DBMessage) => {
      messageList.push(message);
    },
    getOpenAICompatibleMessageList: () => {
      return messageList.map((m) => {
        const message = { ...m };
        delete message.id;
        return message;
      });
    },
  };
};
```

### Step 3: Create the Chat Endpoint

Create the main API endpoint that handles incoming chat requests with streaming:

```typescript
// app/api/chat/route.ts
import { NextRequest, NextResponse } from "next/server";
import OpenAI from "openai";
import { transformStream } from "@crayonai/stream";
import { DBMessage, getMessageStore } from "./messageStore";

export async function POST(req: NextRequest) {
  const { prompt, threadId, responseId } = (await req.json()) as {
    prompt: DBMessage;
    threadId: string;
    responseId: string;
  };

  // Initialize the OpenAI client
  const client = new OpenAI({
    baseURL: "https://api.thesys.dev/v1/embed/",
    apiKey: process.env.THESYS_API_KEY,
  });

  // Get message store and add user message
  const messageStore = getMessageStore(threadId);
  messageStore.addMessage(prompt);

  // Create streaming chat completion
  const llmStream = await client.chat.completions.create({
    model: "c1/anthropic/claude-sonnet-4/v-20251230",
    messages: messageStore.getOpenAICompatibleMessageList(),
    stream: true,
  });

  // Transform the response stream
  const responseStream = transformStream(
    llmStream,
    (chunk) => {
      return chunk.choices[0].delta.content;
    },
    {
      onEnd: ({ accumulated }) => {
        const message = accumulated.filter((message) => message).join("");
        messageStore.addMessage({
          role: "assistant",
          content: message,
          id: responseId,
        });
      },
    }
  ) as ReadableStream;

  // Return the streaming response
  return new NextResponse(responseStream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache, no-transform",
      Connection: "keep-alive",
    },
  });
}
```

### Step 4: Set Your API Key

Set your Thesys API key as an environment variable:

```bash
export THESYS_API_KEY=<your-api-key>
```

Or in Next.js `.env.local`:

```bash
THESYS_API_KEY=<your-api-key>
```

Your API endpoint is now ready to handle streaming chat conversations with the C1 model!

---

## Conversation Persistence with Firebase

This section provides step-by-step instructions to implement chat persistence using Firebase Firestore. By following this guide, you'll learn how to store, retrieve, and manage chat threads and messages, enabling a seamless user experience across sessions.

### Step 1: Set Up Firebase Project & SDK

First, ensure your Firebase project is ready:

1. Go to the [Firebase Console](https://console.firebase.google.com/) and create a new project (or use existing)
2. Navigate to "Firestore Database" and click "Create database"
3. Choose **Test mode** for easier setup (note: not for production)
4. Go to "Project settings" > "General" > "Your apps" > Add Web app
5. Copy the `firebaseConfig` object

```typescript
// src/firebaseConfig.ts
import { initializeApp, getApp, getApps } from "firebase/app";
import { getFirestore } from "firebase/firestore";

// IMPORTANT: Replace these with your actual Firebase project configuration!
const firebaseConfig = {
  apiKey: "YOUR_FIREBASE_API_KEY",
  authDomain: "YOUR_AUTH_DOMAIN",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_STORAGE_BUCKET",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_APP_ID",
  measurementId: "YOUR_MEASUREMENT_ID",
};

// Initialize Firebase
let app;
if (!getApps().length) {
  app = initializeApp(firebaseConfig);
} else {
  app = getApp();
}

const db = getFirestore(app);
export { db };
```

### Step 2: Create Thread Service for Firebase

Implement a service layer that handles persistence logic using Firestore:

```typescript
// src/services/threadService.ts
import { db } from "../firebaseConfig";
import {
  collection,
  addDoc,
  doc,
  getDoc,
  updateDoc,
  serverTimestamp,
} from "firebase/firestore";

const THREADS_COLLECTION = "threads";

// Create a new thread
export const createThread = async (name: string): Promise<Thread> => {
  const newThreadRef = await addDoc(collection(db, THREADS_COLLECTION), {
    name: name,
    messages: [],
    createdAt: serverTimestamp(),
  });
  return {
    threadId: newThreadRef.id,
    title: name,
    createdAt: new Date(),
  };
};

// Add messages to a thread
export const addMessages = async (threadId: string, ...messages: Message[]) => {
  if (!threadId) throw new Error("threadId is required for addMessages");
  const threadRef = doc(db, THREADS_COLLECTION, threadId);
  const threadSnap = await getDoc(threadRef);

  if (!threadSnap.exists()) {
    throw new Error(`Thread with id ${threadId} not found.`);
  }

  const existingMessages = (threadSnap.data()?.messages as Message[]) ?? [];
  const newMessages = existingMessages.concat(messages);

  await updateDoc(threadRef, {
    messages: newMessages,
  });
};

// Get messages for UI display
export const getUIThreadMessages = async (threadId: string) => {
  const threadRef = doc(db, THREADS_COLLECTION, threadId);
  const threadSnap = await getDoc(threadRef);
  return threadSnap.data()?.messages ?? [];
};

// Get messages in OpenAI-compatible format for LLM
export const getLLMThreadMessages = async (threadId: string) => {
  const messages = await getUIThreadMessages(threadId);
  return messages.map(({ id, ...rest }) => rest);
};
```

### Step 3: Integrate Service with Frontend

Configure the C1Chat component to use the `threadService` functions for data operations:

```tsx
// src/app/page.tsx
"use client";

import { useThreadManager, useThreadListManager, C1Chat } from "@thesysai/genui-sdk";
import { useRouter, usePathname, useSearchParams } from "next/navigation";
import {
  createThread,
  deleteThread,
  updateThread,
  getThreadList,
  getUIThreadMessages,
  updateMessage,
} from "@/services/threadService";

export default function Home() {
  const { replace } = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  const threadListManager = useThreadListManager({
    fetchThreadList: () => getThreadList(),
    deleteThread: (threadId) => deleteThread(threadId),
    updateThread: (t) => updateThread({ threadId: t.threadId, name: t.title }),
    onSwitchToNew: () => {
      replace(`${pathname}`);
    },
    onSelectThread: (threadId) => {
      const newSearch = `?threadId=${threadId}`;
      replace(`${pathname}${newSearch}`);
    },
    createThread: (message) => {
      return createThread(message.message!);
    },
  });

  const threadManager = useThreadManager({
    threadListManager,
    loadThread: async (threadId) => await getUIThreadMessages(threadId),
    onUpdateMessage: async ({ message }) => {
      await updateMessage(threadListManager.selectedThreadId!, message);
    },
    apiUrl: "/api/chat",
  });

  return (
    <C1Chat
      threadManager={threadManager}
      threadListManager={threadListManager}
    />
  );
}
```

### Step 4: Add Backend Chat API Route

Add the chat API route that uses the Thesys C1 API along with `threadService` functions for fetching historical messages and saving new ones:

```typescript
// app/api/chat/route.ts
import { NextRequest, NextResponse } from "next/server";
import OpenAI from "openai";
import { transformStream } from "@crayonai/stream";
import { getLLMThreadMessages, addMessages } from "@/services/threadService";

export async function POST(req: NextRequest) {
  const { prompt, threadId, responseId } = await req.json();

  const client = new OpenAI({
    baseURL: "https://api.thesys.dev/v1/embed",
    apiKey: process.env.THESYS_API_KEY,
  });

  // Fetch existing messages for the thread
  const llmMessages = await getLLMThreadMessages(threadId);

  const runToolsResponse = client.beta.chat.completions.runTools({
    model: "c1/anthropic/claude-sonnet-4/v-20251230",
    messages: [
      ...llmMessages, // Add previous messages
      {
        role: "user",
        content: prompt.content!,
      },
    ],
    stream: true,
    tools: [], // Add your tools here if needed
  });

  // Track all messages generated during this turn
  const allRunToolsMessages: any[] = [];
  let isError = false;

  runToolsResponse.on("message", (message) => {
    allRunToolsMessages.push(message);
  });

  runToolsResponse.on("error", () => {
    isError = true;
  });

  // Save messages when stream ends
  runToolsResponse.on("end", async () => {
    if (isError) {
      return;
    }
    // Assign IDs to messages, using responseId for the final assistant message
    const runToolsMessagesWithId = allRunToolsMessages.map((m, index) => {
      const id =
        allRunToolsMessages.length - 1 === index // For last message (shown to user), use responseId
          ? responseId
          : crypto.randomUUID();
      return {
        ...m,
        id,
      };
    });
    const messagesToStore = [prompt, ...runToolsMessagesWithId];
    await addMessages(threadId, ...messagesToStore);
  });

  const responseStream = transformStream(
    runToolsResponse,
    (chunk) => chunk.choices[0].delta.content
  ) as ReadableStream;

  return new NextResponse(responseStream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache, no-transform",
      Connection: "keep-alive",
    },
  });
}
```

### Running and Testing

1. Ensure your Firebase credentials are correctly set in `src/firebaseConfig.ts`
2. Ensure your `THESYS_API_KEY` is set in `.env`
3. Run `npm run dev`
4. Test creating new chats, sending messages, switching between chats, and refreshing the page to verify history persists
5. Check the Firebase console to see data being written to Firestore

---

## Sharing Conversations

### Share an Entire Thread

Use `C1ShareThread` component:

```tsx
import { C1ShareThread } from "@thesysai/genui-sdk";

<C1ShareThread
  generateShareLink={async () => {
    const baseUrl = window.location.origin;
    return `${baseUrl}/shared/${selectedThreadId}`;
  }}
/>

// Custom trigger:
<C1ShareThread
  customTrigger={<button>Share Thread</button>}
  generateShareLink={...}
/>
```

### Share a Single Message

Use `ResponseFooter.ShareButton`:

```tsx
import { ResponseFooter } from "@thesysai/genui-sdk";

const Footer = () => (
  <ResponseFooter.Container>
    <ResponseFooter.ShareButton
      generateShareLink={async (message) => {
        return `${window.location.origin}/shared/${threadId}/${message.id}`;
      }}
    />
  </ResponseFooter.Container>
);

<C1Chat
  customizeC1={{ responseFooterComponent: Footer }}
/>
```

### Render Shared Conversations

```tsx
import { C1ChatViewer } from "@thesysai/genui-sdk";

function ViewShared({ messages }) {
  return <C1ChatViewer messages={messages} />;
}
```

---

## Concepts Reference

### Message Types

Messages follow OpenAI format with `role`:
- **`user`**: User input
- **`assistant`**: AI response (text or C1 DSL)
- **`system`**: Instructions (not visible in UI)

### Thread

An ordered list of messages representing a single conversation.

### ThreadList

Collection of all threads for a user, typically shown in a sidebar.

### Flow of a Conversation

1. User types prompt and sends
2. App creates user message, adds to thread
3. Frontend sends to backend
4. Backend constructs payload, calls C1 API
5. C1 returns response, stored as assistant message
6. Frontend displays in UI
