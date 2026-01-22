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
  const [value, setValue] = useC1State('toggle-enabled');

  return (
    <button onClick={() => setValue(!value)}>
      {value ? 'On' : 'Off'}
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

## Conversation Persistence with Firebase

### Step 1: Set Up Firebase

```typescript
// src/firebaseConfig.ts
import { initializeApp, getApp, getApps } from "firebase/app";
import { getFirestore } from "firebase/firestore";

const firebaseConfig = {
  apiKey: "YOUR_FIREBASE_API_KEY",
  authDomain: "YOUR_AUTH_DOMAIN",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_STORAGE_BUCKET",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_APP_ID",
};

let app;
if (!getApps().length) {
  app = initializeApp(firebaseConfig);
} else {
  app = getApp();
}

export const db = getFirestore(app);
```

### Step 2: Create Thread Service

```typescript
// src/services/threadService.ts
import { db } from "../firebaseConfig";
import { collection, addDoc, doc, getDoc, updateDoc, serverTimestamp } from "firebase/firestore";

const THREADS_COLLECTION = "threads";

export const createThread = async (name: string): Promise<Thread> => {
  const newThreadRef = await addDoc(collection(db, THREADS_COLLECTION), {
    name,
    messages: [],
    createdAt: serverTimestamp(),
  });
  return {
    threadId: newThreadRef.id,
    title: name,
    createdAt: new Date(),
  };
};

export const addMessages = async (threadId: string, ...messages: Message[]) => {
  const threadRef = doc(db, THREADS_COLLECTION, threadId);
  const threadSnap = await getDoc(threadRef);
  
  if (!threadSnap.exists()) {
    throw new Error(`Thread ${threadId} not found`);
  }
  
  const existingMessages = threadSnap.data()?.messages ?? [];
  await updateDoc(threadRef, {
    messages: existingMessages.concat(messages),
  });
};

export const getUIThreadMessages = async (threadId: string) => {
  const threadRef = doc(db, THREADS_COLLECTION, threadId);
  const threadSnap = await getDoc(threadRef);
  return threadSnap.data()?.messages ?? [];
};

export const getLLMThreadMessages = async (threadId: string) => {
  // Return messages in OpenAI-compatible format
  const messages = await getUIThreadMessages(threadId);
  return messages.map(({ id, ...rest }) => rest);
};
```

### Step 3: Integrate with Frontend

```tsx
// src/app/page.tsx
import { useThreadManager, useThreadListManager, C1Chat } from "@thesysai/genui-sdk";
import {
  createThread,
  deleteThread,
  updateThread,
  getThreadList,
  getUIThreadMessages,
  updateMessage,
} from "@/services/threadService";

export default function Home() {
  const threadListManager = useThreadListManager({
    fetchThreadList: () => getThreadList(),
    deleteThread: (threadId) => deleteThread(threadId),
    updateThread: (t) => updateThread({ threadId: t.threadId, name: t.title }),
    onSwitchToNew: () => router.push('/'),
    onSelectThread: (threadId) => router.push(`?threadId=${threadId}`),
    createThread: (message) => createThread(message.message!),
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

### Step 4: Backend API with Persistence

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

  // Fetch existing messages
  const llmMessages = await getLLMThreadMessages(threadId);

  const llmStream = await client.chat.completions.create({
    model: "c1/anthropic/claude-sonnet-4/v-20251230",
    messages: [
      ...llmMessages,
      { role: "user", content: prompt.content },
    ],
    stream: true,
  });

  const responseStream = transformStream(
    llmStream,
    (chunk) => chunk.choices[0].delta.content,
    {
      onEnd: async ({ accumulated }) => {
        const message = accumulated.filter(Boolean).join("");
        // Save user prompt and assistant response
        await addMessages(threadId, prompt, {
          role: "assistant",
          content: message,
          id: responseId,
        });
      },
    }
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
