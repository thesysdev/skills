# Customizations & Styling

This guide covers theming, chart customization, and CSS overrides for C1-generated UIs.

C1 is designed to be highly customizable. Here are the main ways to customize C1 UI to your requirements.

---

## Customizing C1Chat

### Step 1: Choosing a Form Factor

C1 offers flexibility in its form factor:

- **Full Page**: A complete page conversation interface, similar to ChatGPT
- **Side Panel**: A copilot-style conversation interface

```tsx
import { C1Chat } from "@thesysai/genui-sdk";
import "@crayonai/react-ui/styles/index.css";

export default function Home() {
  return (
    <C1Chat
      apiUrl="/api/chat"
      formFactor="full-page"  // or "side-panel"
    />
  );
}
```

### Step 2: Using Theme Presets

C1 can easily be customized through a variety of pre-built themes. Import `themePresets` from `@crayonai/react-ui` and pass it to the `theme` prop:

```tsx
import { C1Chat } from "@thesysai/genui-sdk";
import "@crayonai/react-ui/styles/index.css";
import { themePresets } from "@crayonai/react-ui";

export default function Home() {
  return (
    <C1Chat
      apiUrl="/api/chat"
      theme={themePresets.candy}
    />
  );
}
```

### Step 3: Switching Between Light and Dark Mode

Toggle between light and dark modes by setting the `mode` property in the theme object. All Crayon theme presets fully support both modes:

```tsx
import { C1Chat } from "@thesysai/genui-sdk";
import "@crayonai/react-ui/styles/index.css";
import { themePresets } from "@crayonai/react-ui";

export default function Home() {
  return (
    <C1Chat
      apiUrl="/api/chat"
      theme={{ ...themePresets.candy, mode: "dark" }}
    />
  );
}
```

### Step 4: Setting Agent Name and Logo

Set the agent name and logo using the `agentName` and `logoUrl` props. These control the agent's display name in the sidebar and the avatar shown next to its messages:

```tsx
import { C1Chat } from "@thesysai/genui-sdk";
import "@crayonai/react-ui/styles/index.css";
import { themePresets } from "@crayonai/react-ui";
import { ThemeProvider } from "@crayonai/react-ui";
import { useSystemTheme } from "./useSystemTheme";

export default function Home() {
  const systemTheme = useSystemTheme();
  return (
    <ThemeProvider {...themePresets.candy} mode={systemTheme}>
      <C1Chat
        agentName="Legal Assistant"
        logoUrl="https://placehold.co/100"
        apiUrl="/api/chat"
      />
    </ThemeProvider>
  );
}
```

### Step 5: Overriding Crayon CSS Classes

For advanced customization, override the CSS classes applied to UI components. For example, to hide the AI agent's logo next to its messages:

```css
/* custom.css */
.crayon-shell-thread-message-assistant__logo {
  display: none;
}
```

You can find the classes attached to different UI components by inspecting the elements through the browser's developer tools.

Then import those styles in your component:

```tsx
import { C1Chat } from "@thesysai/genui-sdk";
import "@crayonai/react-ui/styles/index.css";
import { themePresets, ThemeProvider } from "@crayonai/react-ui";
import { useSystemTheme } from "./useSystemTheme";
import "./custom.css";

export default function Home() {
  const systemTheme = useSystemTheme();
  return (
    <ThemeProvider {...themePresets.candy} mode={systemTheme}>
      <C1Chat
        agentName="Legal Assistant"
        logoUrl="https://placehold.co/100"
        apiUrl="/api/chat"
      />
    </ThemeProvider>
  );
}
```

### C1Chat Props Reference

| Prop | Type | Description |
|------|------|-------------|
| `apiUrl` | `string` | Backend endpoint for chat API |
| `formFactor` | `'full-page' \| 'side-panel'` | Layout style |
| `theme` | `Theme` | Theme configuration or preset |
| `agentName` | `string` | Display name in chat |
| `logoUrl` | `string` | Agent logo URL |
| `scrollVariant` | `'once' \| 'user-message-anchor' \| 'always'` | Scroll behavior |
| `customizeC1` | `CustomizeC1Props` | Custom components (e.g., `thinkComponent`) |
| `threadManager` | `ThreadManager` | Custom thread management |
| `threadListManager` | `ThreadListManager` | Thread list management |

### Custom Components in C1Chat

```tsx
<C1Chat
  customizeC1={{
    thinkComponent: CustomThinkComponent,
    responseFooterComponent: CustomFooter,
    customComponents: { MyComponent },
    onAction: handleAction,
  }}
/>
```

---

## Theming with `<ThemeProvider>`

For more control, use `ThemeProvider` to wrap your components:

### Basic Usage

```tsx
import { ThemeProvider } from "@crayonai/react-ui";
import { C1Component } from "@thesysai/genui-sdk";

<ThemeProvider mode="dark" {...themeConfig}>
  <C1Component c1Response={response} />
</ThemeProvider>
```

### Theme Object Properties

```typescript
interface Theme {
  // Mode
  mode?: 'light' | 'dark';
  
  // Colors
  primaryColor?: string;
  backgroundColor?: string;
  textColor?: string;
  
  // Typography
  fontFamily?: string;
  
  // Spacing & Layout
  borderRadius?: string;
  spacing?: string;
  
  // Chart Palettes
  defaultChartPalette?: string[];
  barChartPalette?: string[];
  lineChartPalette?: string[];
  areaChartPalette?: string[];
  pieChartPalette?: string[];
  radarChartPalette?: string[];
  radialChartPalette?: string[];
}
```

### Font Recommendation

C1 uses Inter by default. Import in your CSS:

```css
@import url("https://fonts.googleapis.com/css2?family=Inter:wght@100;200;300;400;500;600;700;800;900&display=swap");
```

---

## Customizing Charts

### Global Chart Palette

Set a default palette for all chart types:

```tsx
<ThemeProvider
  defaultChartPalette={[
    "#34495e",
    "#16a085",
    "#f39c12",
    "#e74c3c",
    "#8e44ad",
  ]}
>
  {/* All charts use this palette */}
</ThemeProvider>
```

### Chart-Specific Palettes

Override specific chart types:

```tsx
<ThemeProvider
  barChartPalette={["#1f77b4", "#ff7f0e", "#2ca02c"]}
  defaultChartPalette={["#34495e", "#16a085", "#f39c12"]}
>
  {/* Bar charts get custom colors, others use default */}
</ThemeProvider>
```

### Available Palette Keys

| Key | Description |
|-----|-------------|
| `defaultChartPalette` | Fallback for all charts |
| `barChartPalette` | Bar charts |
| `lineChartPalette` | Line charts |
| `areaChartPalette` | Area charts |
| `pieChartPalette` | Pie charts |
| `radarChartPalette` | Radar charts |
| `radialChartPalette` | Radial charts |

### Light/Dark Mode Palettes

Define different palettes per mode using theme presets:

```tsx
import { themePresets, ThemeProvider } from "@crayonai/react-ui";

const lightTheme = {
  ...themePresets.candy,
  mode: "light" as const,
  areaChartPalette: ["#2563eb", "#dc2626", "#16a34a"],
};

const darkTheme = {
  ...themePresets.candy,
  mode: "dark" as const,
  areaChartPalette: ["#3b82f6", "#ef4444", "#22c55e"],
};

function App() {
  const [isDark, setIsDark] = useState(false);
  
  return (
    <ThemeProvider {...(isDark ? darkTheme : lightTheme)}>
      {/* Charts adapt to mode */}
    </ThemeProvider>
  );
}
```

---

## CSS Overrides

> **Note**: Class names may change between SDK versions. Review overrides after updates.

Use CSS for precise tweaks not covered by theming.

### Finding CSS Classes

1. Right-click element in browser
2. Select "Inspect"
3. Find class names in the `class` attribute

### Example Overrides

```css
/* custom.css */

/* Hide agent logo next to messages */
.crayon-shell-thread-message-assistant__logo {
  display: none;
}

/* Make card titles uppercase */
.crayon-header {
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

/* Custom button styling */
.crayon-button {
  border-radius: 8px;
  font-weight: 600;
}

/* Adjust table header */
.crayon-table-header {
  background-color: #f0f0f0;
}
```

### Best Practices

- **Be Specific, Not Too Specific**: Avoid `div > span > p.c1-title`
- **Target Stable Classes**: Use `.crayon-header` not generated IDs
- **Start with Theming**: Only use CSS for what themes can't handle
- **Document Overrides**: Note SDK version and purpose

---

## Component Metadata

Control which C1 components are available:

### Include Components

```typescript
const response = await client.chat.completions.create({
  model: "c1/anthropic/claude-sonnet-4/v-20251230",
  messages: [...],
  metadata: {
    thesys: JSON.stringify({
      c1_included_components: ["EditableTable"],
    }),
  },
});
```

### Exclude Components

```typescript
metadata: {
  thesys: JSON.stringify({
    c1_excluded_components: ["Layout"],
  }),
}
```

Useful for:
- Width-constrained UIs (exclude Layout)
- Enabling experimental components
- Simplifying available components

---

## Complete Theme Example

```tsx
import { useState } from "react";
import { C1Chat } from "@thesysai/genui-sdk";
import { ThemeProvider, themePresets } from "@crayonai/react-ui";
import "@crayonai/react-ui/styles/index.css";

// Extend a preset with custom chart palettes
const brandLightTheme = {
  ...themePresets.candy,
  mode: "light" as const,
  defaultChartPalette: ["#6366f1", "#8b5cf6", "#a855f7", "#d946ef"],
  barChartPalette: ["#3b82f6", "#10b981", "#f59e0b"],
  pieChartPalette: ["#ef4444", "#f97316", "#eab308", "#22c55e", "#14b8a6"],
};

const brandDarkTheme = {
  ...themePresets.candy,
  mode: "dark" as const,
  defaultChartPalette: ["#818cf8", "#a78bfa", "#c084fc", "#e879f9"],
  barChartPalette: ["#60a5fa", "#34d399", "#fbbf24"],
  pieChartPalette: ["#f87171", "#fb923c", "#facc15", "#4ade80", "#2dd4bf"],
};

function App() {
  const [isDark, setIsDark] = useState(false);
  
  return (
    <ThemeProvider {...(isDark ? brandDarkTheme : brandLightTheme)}>
      <button onClick={() => setIsDark(!isDark)}>
        Toggle Theme
      </button>
      <C1Chat
        apiUrl="/api/chat"
        agentName="Brand Assistant"
        logoUrl="/logo.svg"
        formFactor="full-page"
      />
    </ThemeProvider>
  );
}
```

Or use the simpler approach with C1Chat's `theme` prop:

```tsx
import { useState } from "react";
import { C1Chat } from "@thesysai/genui-sdk";
import { themePresets } from "@crayonai/react-ui";
import "@crayonai/react-ui/styles/index.css";

function App() {
  const [isDark, setIsDark] = useState(false);
  
  return (
    <C1Chat
      apiUrl="/api/chat"
      agentName="Brand Assistant"
      logoUrl="/logo.svg"
      formFactor="full-page"
      theme={{
        ...themePresets.candy,
        mode: isDark ? "dark" : "light",
      }}
    />
  );
}
```

---

## Error Handling UI

Handle rendering errors with `onError`:

```tsx
<C1Component
  c1Response={response}
  onError={({ code, c1Response }) => {
    console.error(`UI Error ${code}:`, c1Response);
    // Show fallback UI or error message
  }}
/>
```
