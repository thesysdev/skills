# Theme OpenUI's React components

Use this for colors, fonts, radii, and light/dark mode in `@openuidev/react-ui`: `ThemeProvider`, `AgentInterface`'s `theme` prop, and theme tokens. Check the installed `ThemeProvider` types before using a token name; they change between versions.

## The API

```tsx
import { ThemeProvider, createTheme, useTheme, type ThemeMode, type ThemeProps } from "@openuidev/react-ui";
```

| `ThemeProvider` prop | Meaning |
| --- | --- |
| `mode` | `"light"` or `"dark"`. Defaults to `"light"` at the top level; a nested provider inherits its parent's mode. |
| `lightTheme` | Token overrides merged onto the built-in light theme |
| `darkTheme` | Token overrides merged onto the built-in dark theme. When omitted, `lightTheme`'s overrides apply to both modes. |
| `cssSelector` | Where the `--openui-*` CSS variables are injected. Default `body`; nested providers scope themselves automatically. |
| `theme` | Deprecated alias for `lightTheme` |

`ThemeProvider` injects CSS variables. It does not load the component styles; keep the stylesheet import.

## Pick one owner

| Situation | Do this |
| --- | --- |
| Only `AgentInterface` needs theming | Pass `theme={{ mode, lightTheme, darkTheme }}` to it |
| OpenUI components outside `AgentInterface` need the same theme | Wrap the tree in `ThemeProvider`, and set `disableThemeProvider` on `AgentInterface` |
| One subtree needs a different scheme | Nest another `ThemeProvider` and pass that scope's full overrides |
| Separate roots need different themes | Give each a unique `cssSelector` on an element that wraps it |

A nested provider starts from the built-in defaults plus its own overrides, not from the parent's custom tokens. Pass the brand overrides again in nested providers.

`AgentInterface.theme` takes the provider's props (`{ mode, lightTheme, darkTheme }`), not a token object. `theme={brandTheme}` is wrong.

## Build theme objects once

Create themes with `createTheme()` at module scope, or memoize them. A new object on every render makes the provider rewrite its styles.

```tsx
import { createTheme } from "@openuidev/react-ui";

export const brandLight = createTheme({
  background: "var(--app-canvas)",
  foreground: "var(--app-surface)",
  textNeutralPrimary: "var(--app-text)",
  interactiveAccentDefault: "var(--app-accent)",
  interactiveAccentHover: "var(--app-accent-hover)",
  textAccentPrimary: "var(--app-on-accent)",
  borderDefault: "var(--app-border)",
  chatUserResponseBg: "var(--app-user-bubble)",
  radiusM: "10px",
});
```

- Map roles, not only the brand color: surfaces, primary and secondary text, every accent state, text on accent, borders, chat bubbles.
- Changing `fontBody` alone does not update compound typography tokens such as `textBodyDefault`; override those too.
- To read a token in the app's own CSS, convert camelCase to kebab case: `interactiveAccentDefault` is `--openui-interactive-accent-default`.

## Light, dark, and system mode

`ThemeProvider` takes only `"light"` or `"dark"`. To offer Light / Dark / System with a saved choice, use `useThemeModePreference()`:

```tsx
import { AgentInterface, useThemeModePreference } from "@openuidev/react-ui";

export function Chat() {
  const { preference, setPreference, mode } = useThemeModePreference();
  return <AgentInterface llm={llm} theme={{ mode, lightTheme: brandLight, darkTheme: brandDark }} />;
}
```

- `preference` is `"light"`, `"dark"`, or `"system"`. `setPreference` saves it to localStorage and sets `data-openui-theme` on `<html>`. `mode` is the resolved scheme to pass on.
- `useSystemThemeMode()` returns only the OS scheme.
- `AgentInterface.ThemeModeToggle` is a ready-made toggle for the sidebar.
- If the app already manages its theme, pass its resolved mode instead of adding a second preference store.

## Portals

OpenUI's own popovers and dialogs carry the theme. For an app-owned portal rendered outside the provider's tree, add `useTheme().portalThemeClassName` to the portal container.

## CSS

Import `@openuidev/react-ui/styles/index.css` once, or `@openuidev/react-ui/layered/styles/index.css` when the app uses Tailwind v4 or cascade layers. Do not import both.

## Verify

1. A `<style data-openui-theme>` element is in `<head>`, and its selector matches the intended scope.
2. Switching mode changes the computed `--openui-*` values, with no console warnings about unknown or deprecated keys.
3. Text, buttons in every state, inputs, chat bubbles, menus, and dialogs have readable contrast in both modes.
4. A nested theme does not leak to its siblings, and portals pick up the theme.
5. Run the project's typecheck and build.

## Sources

- [ThemeProvider source](https://github.com/thesysdev/openui/tree/main/packages/react-ui/src/components/ThemeProvider)
- [React UI CSS setup](https://www.openui.com/docs/api-reference/react-ui#tailwind-v4)
