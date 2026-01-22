# Customizations & Styling

This guide covers theming, chart customization, and CSS overrides for C1-generated UIs.

---

## Styling Overview

C1 uses a layered styling approach:

1. **Theming** (Primary): Global styles via `<ThemeProvider>`
2. **Component-Specific**: Chart palettes and component options
3. **CSS Overrides** (Advanced): Direct CSS targeting

---

## Theming with `<ThemeProvider>`

### Basic Usage

```tsx
import { ThemeProvider } from "@thesysai/genui-sdk";

<ThemeProvider mode="dark" theme={lightTheme} darkTheme={darkTheme}>
  <C1Component c1Response={response} />
</ThemeProvider>
```

### Props

| Prop | Type | Description |
|------|------|-------------|
| `mode` | `'light' \| 'dark'` | Theme mode |
| `theme` | `Theme` | Light theme configuration |
| `darkTheme` | `Theme` | Dark theme configuration (falls back to `theme`) |

### Theme Object Properties

```typescript
interface Theme {
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
  theme={{
    defaultChartPalette: [
      "#34495e",
      "#16a085",
      "#f39c12",
      "#e74c3c",
      "#8e44ad",
    ],
  }}
>
  {/* All charts use this palette */}
</ThemeProvider>
```

### Chart-Specific Palettes

Override specific chart types:

```tsx
<ThemeProvider
  theme={{
    // Bar charts get custom colors
    barChartPalette: ["#1f77b4", "#ff7f0e", "#2ca02c"],
    
    // All other charts use default
    defaultChartPalette: ["#34495e", "#16a085", "#f39c12"],
  }}
>
  {/* ... */}
</ThemeProvider>
```

### Available Palette Keys

```typescript
interface ChartColorPalette {
  defaultChartPalette?: string[];  // Fallback for all charts
  barChartPalette?: string[];      // Bar charts
  lineChartPalette?: string[];     // Line charts
  areaChartPalette?: string[];     // Area charts
  pieChartPalette?: string[];      // Pie charts
  radarChartPalette?: string[];    // Radar charts
  radialChartPalette?: string[];   // Radial charts
}
```

### Light/Dark Mode Palettes

Define different palettes per mode:

```tsx
const lightTheme = {
  areaChartPalette: ["#2563eb", "#dc2626", "#16a34a"],
  defaultChartPalette: ["#374151", "#111827", "#1f2937"],
};

const darkTheme = {
  areaChartPalette: ["#3b82f6", "#ef4444", "#22c55e"],
  defaultChartPalette: ["#d1d5db", "#f3f4f6", "#e5e7eb"],
};

<ThemeProvider
  mode="dark"
  theme={lightTheme}
  darkTheme={darkTheme}
>
  {/* Charts adapt to mode */}
</ThemeProvider>
```

---

## CSS Overrides

> ⚠️ **Advanced Technique**: Class names may change between SDK versions. Review overrides after updates.

Use CSS for precise, specific tweaks not covered by theming.

### Good Use Cases

- Font weight of specific card titles
- Unique borders on elements
- Margin adjustments on buttons
- Custom hover states

### Finding CSS Classes

1. Right-click element in browser
2. Select "Inspect"
3. Find class names in the `class` attribute

### Example Override

```css
/* styles/custom.css */

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

## `<C1Chat>` Theming

Apply themes directly to C1Chat:

```tsx
<C1Chat
  apiUrl="/api/chat"
  theme={{
    mode: "dark",
    theme: lightTheme,
    darkTheme: darkTheme,
  }}
/>
```

Or wrap with ThemeProvider:

```tsx
<ThemeProvider mode="dark" theme={lightTheme} darkTheme={darkTheme}>
  <C1Chat apiUrl="/api/chat" />
</ThemeProvider>
```

---

## Customizing `<C1Chat>` Layout

### Form Factors

```tsx
<C1Chat
  formFactor="full-page"    // Default: Full screen chat
  // formFactor="side-panel" // Sidebar style
  // formFactor="bottom-tray" // Widget at bottom
/>
```

### Agent Branding

```tsx
<C1Chat
  agentName="My AI Assistant"
  logoUrl="/path/to/logo.png"
/>
```

### Scroll Behavior

```tsx
<C1Chat
  scrollVariant="once"              // Scroll once per interaction
  // scrollVariant="user-message-anchor" // Anchor to user messages
  // scrollVariant="always"          // Continuous scrolling
/>
```

### Custom Components

```tsx
<C1Chat
  customizeC1={{
    thinkComponent: CustomThinkComponent,
    responseFooterComponent: CustomFooter,
    exportAsPdf: handleExport,
    onAction: handleAction,
  }}
/>
```

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
import { ThemeProvider, C1Chat } from "@thesysai/genui-sdk";

const brandTheme = {
  primaryColor: "#6366f1",
  backgroundColor: "#ffffff",
  textColor: "#1f2937",
  fontFamily: "'Inter', sans-serif",
  borderRadius: "8px",
  
  // Charts
  defaultChartPalette: ["#6366f1", "#8b5cf6", "#a855f7", "#d946ef"],
  barChartPalette: ["#3b82f6", "#10b981", "#f59e0b"],
  pieChartPalette: ["#ef4444", "#f97316", "#eab308", "#22c55e", "#14b8a6"],
};

const darkBrandTheme = {
  primaryColor: "#818cf8",
  backgroundColor: "#111827",
  textColor: "#f9fafb",
  fontFamily: "'Inter', sans-serif",
  borderRadius: "8px",
  
  defaultChartPalette: ["#818cf8", "#a78bfa", "#c084fc", "#e879f9"],
  barChartPalette: ["#60a5fa", "#34d399", "#fbbf24"],
  pieChartPalette: ["#f87171", "#fb923c", "#facc15", "#4ade80", "#2dd4bf"],
};

function App() {
  const [isDark, setIsDark] = useState(false);
  
  return (
    <ThemeProvider
      mode={isDark ? "dark" : "light"}
      theme={brandTheme}
      darkTheme={darkBrandTheme}
    >
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
