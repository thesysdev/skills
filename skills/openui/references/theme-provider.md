# OpenUI React ThemeProvider

Use this runbook for `ThemeProvider`, `AgentInterface.theme`, light/dark mode, brand-token mapping, scoped themes, and custom portal theming in `@openuidev/react-ui`.

## Contents

- [Verify the installed API](#verify-the-installed-api)
- [Choose one provider owner](#choose-one-provider-owner)
- [Build stable theme objects](#build-stable-theme-objects)
- [Control light and dark mode](#control-light-and-dark-mode)
- [Theme AgentInterface](#theme-agentinterface)
- [Share one provider with AgentInterface](#share-one-provider-with-agentinterface)
- [Use nested or independent scopes](#use-nested-or-independent-scopes)
- [Theme custom portals](#theme-custom-portals)
- [Keep the CSS integration](#keep-the-css-integration)
- [Verify the result](#verify-the-result)
- [Avoid common failures](#avoid-common-failures)

## Verify the installed API

Inspect the app's `package.json`, lockfile, and installed `@openuidev/react-ui` exports before editing. Prefer the installed package's `ThemeProvider` source, type declarations, `defaultTheme`, and `utils` over examples from another release. When the task targets current `main` or installed source is unavailable, use the first-party [ThemeProvider source folder](https://github.com/thesysdev/openui/tree/main/packages/react-ui/src/components/ThemeProvider).

Import public APIs from the package root:

```tsx
import {
  ThemeProvider,
  createTheme,
  useTheme,
  type ThemeMode,
  type ThemeProps,
} from "@openuidev/react-ui";
```

On current `main`, `ThemeProvider` accepts:

| Prop | Meaning |
| --- | --- |
| `mode?: "light" \| "dark"` | Active scheme. A top-level provider defaults to `"light"`; a nested provider inherits its parent's mode when omitted. |
| `lightTheme?: Theme` | Partial overrides merged onto built-in light defaults. |
| `darkTheme?: Theme` | Partial overrides merged onto built-in dark defaults. If omitted, `lightTheme` overrides are reused over the dark defaults. |
| `theme?: Theme` | Deprecated alias for `lightTheme`; do not use in new code. |
| `cssSelector?: string` | Selector that receives injected `--openui-*` variables; defaults to `body`, with automatic scoping for nested providers. |
| `children?: ReactNode` | The themed React subtree. |

`ThemeProvider` injects runtime CSS custom properties; it does not import the OpenUI component styles, persist a preference, render a toggle, or manage application theme state.

## Choose one provider owner

Use this decision table before writing code:

| Situation | Pattern |
| --- | --- |
| Only one `AgentInterface` needs the theme | Pass `{ mode, lightTheme, darkTheme }` to `AgentInterface.theme`; keep its internal provider enabled. |
| OpenUI components or renderers across a broader app tree need the theme | Wrap that tree in `ThemeProvider`. |
| A broader `ThemeProvider` already contains `AgentInterface` | Set `disableThemeProvider` on `AgentInterface`. |
| One subtree intentionally needs another scheme or token set | Nest a second `ThemeProvider` and pass that scope's complete overrides. |
| Separate top-level roots need different themes | Give each root a unique wrapper selector through `cssSelector`; do not let multiple providers all target `body`. |

Do not wrap `AgentInterface` in an outer provider while leaving its internal provider enabled by accident. A nested provider resolves its tokens from built-in defaults plus its own overrides, not from the parent's resolved custom theme. The inner provider can therefore reset the outer brand tokens.

A top-level provider targets `body` by default, so its CSS variables are global even though React context is limited to its children. When only one top-level subtree should receive the theme, give an existing wrapper a unique `cssSelector` as shown below.

## Build stable theme objects

Create partial themes with `createTheme()` and keep them at module scope or memoize them. Do not construct theme objects inline on every render; unstable object identity makes the provider recompute and replace its injected style unnecessarily.

```tsx
import { createTheme } from "@openuidev/react-ui";

export const brandLightTheme = createTheme({
  background: "var(--app-canvas)",
  foreground: "var(--app-surface)",
  popoverBackground: "var(--app-popover)",
  textNeutralPrimary: "var(--app-text-primary)",
  textNeutralSecondary: "var(--app-text-secondary)",
  interactiveAccentDefault: "var(--app-accent)",
  interactiveAccentHover: "var(--app-accent-hover)",
  interactiveAccentPressed: "var(--app-accent-pressed)",
  interactiveAccentDisabled: "var(--app-accent-disabled)",
  textAccentPrimary: "var(--app-on-accent)",
  borderDefault: "var(--app-border)",
  chatUserResponseBg: "var(--app-user-message)",
  chatUserResponseText: "var(--app-on-user-message)",
  radiusM: "10px",
});

export const brandDarkTheme = createTheme({
  background: "var(--app-canvas-dark)",
  foreground: "var(--app-surface-dark)",
  popoverBackground: "var(--app-popover-dark)",
  textNeutralPrimary: "var(--app-text-primary-dark)",
  textNeutralSecondary: "var(--app-text-secondary-dark)",
  interactiveAccentDefault: "var(--app-accent-dark)",
  interactiveAccentHover: "var(--app-accent-hover-dark)",
  interactiveAccentPressed: "var(--app-accent-pressed-dark)",
  interactiveAccentDisabled: "var(--app-accent-disabled-dark)",
  textAccentPrimary: "var(--app-on-accent-dark)",
  borderDefault: "var(--app-border-dark)",
  chatUserResponseBg: "var(--app-user-message-dark)",
  chatUserResponseText: "var(--app-on-user-message-dark)",
});
```

Map semantic roles, not just the brand hex value. At minimum inspect surfaces, primary/secondary text, all interactive accent states, text on accent, borders, chat bubbles, typography, and radius. Check status colors and shadows when the UI uses them.

Do not expect primitive typography overrides to regenerate compound typography tokens. For example, changing only `fontBody` does not rewrite `textBodyDefault`, `textBodyDefaultHeavy`, or the other `textBody*` shorthands. When changing font family, size, weight, or line height, inspect which compound tokens the target components consume and override those tokens consistently.

Use exact installed token keys. Scalar token values are CSS strings. Chart palette fields, when supported by the installed version, are arrays consumed by chart components rather than emitted as CSS variables. `createTheme()` performs development-time key validation but does not make an unsupported token functional.

If the same brand overrides work in both schemes, omit `darkTheme`; the provider reapplies `lightTheme` overrides over the built-in dark defaults. Define `darkTheme` when any shared override would have poor contrast or the design system exposes mode-specific values.

## Control light and dark mode

Keep `ThemeProvider` controlled by the host application's source of truth:

```tsx
"use client";

import type { ReactNode } from "react";
import { ThemeProvider, type ThemeMode } from "@openuidev/react-ui";
import { brandDarkTheme, brandLightTheme } from "./openui-theme";

export function OpenUIThemeRoot({
  mode,
  children,
}: {
  mode: ThemeMode;
  children: ReactNode;
}) {
  return (
    <ThemeProvider mode={mode} lightTheme={brandLightTheme} darkTheme={brandDarkTheme}>
      {children}
    </ThemeProvider>
  );
}
```

Only pass `"light"` or `"dark"`. `ThemeProvider` does not accept `"system"` and `useTheme()` does not return a setter. Resolve system preference, user choice, and persistence in the app's existing theme manager, then pass the resolved mode. Do not import internal source files such as `useSystemThemeMode`; use only installed public exports.

In Next.js App Router, place the provider and any browser theme logic behind a client-component boundary. Keep the server and initial client mode consistent enough to avoid a hydration flash; follow the host app's established theme bootstrap instead of adding a second preference store.

## Theme AgentInterface

`AgentInterface.theme` expects the full `ThemeProps` envelope, not a token object:

```tsx
import { AgentInterface, type ThemeMode, type ThemeProps } from "@openuidev/react-ui";
import { brandDarkTheme, brandLightTheme } from "./openui-theme";

export function Chat({ mode }: { mode: ThemeMode }) {
  const theme: ThemeProps = {
    mode,
    lightTheme: brandLightTheme,
    darkTheme: brandDarkTheme,
  };

  return <AgentInterface llm={llm} theme={theme} />;
}
```

Do not pass `theme={brandLightTheme}`. `AgentInterface` spreads `theme` onto its internal `ThemeProvider`, so the object must contain provider prop names such as `mode`, `lightTheme`, and `darkTheme`.

## Share one provider with AgentInterface

When a host-level OpenUI provider should theme both `AgentInterface` and adjacent OpenUI components, disable the interface's internal provider:

```tsx
<ThemeProvider mode={mode} lightTheme={brandLightTheme} darkTheme={brandDarkTheme}>
  <ProductHeader />
  <AgentInterface llm={llm} disableThemeProvider />
  <AssistantGenUI />
</ThemeProvider>
```

Do not also pass `AgentInterface.theme` in this pattern. If a nested interface theme is intentional, keep the internal provider and pass all overrides that the inner scope needs.

## Use nested or independent scopes

Nested providers automatically scope their CSS variables with a generated class and a `display: contents` wrapper. The nested provider inherits the parent mode when `mode` is omitted, but it does not inherit the parent's custom token object. Re-pass brand overrides when a nested mode should retain the brand:

```tsx
<ThemeProvider mode="light" lightTheme={brandLightTheme} darkTheme={brandDarkTheme}>
  <MainContent />
  <ThemeProvider mode="dark" lightTheme={brandLightTheme} darkTheme={brandDarkTheme}>
    <DarkSidebar />
  </ThemeProvider>
</ThemeProvider>
```

For separate top-level scopes, use selectors that already wrap the corresponding content:

```tsx
<section className="customer-portal">
  <ThemeProvider
    cssSelector=".customer-portal"
    mode={mode}
    lightTheme={customerLightTheme}
    darkTheme={customerDarkTheme}
  >
    <CustomerApp />
  </ThemeProvider>
</section>
```

An explicit `cssSelector` bypasses nested auto-scoping. Confirm that the selector matches an ancestor of the rendered OpenUI subtree; the provider does not add the requested selector to the DOM.

## Theme custom portals

OpenUI's built-in portal components propagate the scoped theme. For an app-owned portal that renders outside the provider's DOM subtree, apply `portalThemeClassName` to the portal container:

```tsx
import { createPortal } from "react-dom";
import { useTheme } from "@openuidev/react-ui";

function CustomPortal() {
  const { portalThemeClassName } = useTheme();

  return createPortal(
    <div className={portalThemeClassName}>
      <CustomDialog />
    </div>,
    document.body,
  );
}
```

Use `useTheme()` when a custom component needs the fully resolved `theme`, active `mode`, or portal class. Without a provider, the hook returns the built-in light theme, `"light"`, and an empty portal class.

## Keep the CSS integration

Import React UI styles once even when using `ThemeProvider`:

```tsx
import "@openuidev/react-ui/components.css";
import "@openuidev/react-ui/styles/index.css";
```

Use `@openuidev/react-ui/layered/styles/index.css` instead of the unlayered styles when the host app needs cascade-layer overrides. Do not import both variants. The provider supplies variables; it does not replace component CSS.

Prefer `createTheme()` over hand-writing `--openui-*` variables. If host CSS must consume an OpenUI token, the mapping is `camelCase` to kebab case: `interactiveAccentDefault` becomes `--openui-interactive-accent-default`. Verify the key in the installed version before relying on the variable.

## Verify the result

Run the host app's typecheck and production build, then verify all of the following:

1. Inspect `<style data-openui-theme>` in `<head>` and confirm that its selector matches the intended scope.
2. Toggle the host mode and confirm that computed `--openui-*` values change without console warnings.
3. Check light and dark contrast for surfaces, body text, primary buttons in every state, inputs, user messages, dropdowns, and modals.
4. Confirm that a nested theme does not change siblings outside its scope.
5. Open every built-in and custom portal; confirm that scoped variables reach the portal container.
6. If an outer provider owns the theme, confirm that every contained `AgentInterface` uses `disableThemeProvider`.
7. Check development logs for deprecated `theme`, unknown keys, or non-string scalar values.

## Avoid common failures

- Do not pass raw token fields directly to `AgentInterface.theme`; wrap them in `lightTheme`/`darkTheme`.
- Do not add an outer provider around `AgentInterface` without disabling its internal provider unless a reset nested scope is deliberate.
- Do not use the deprecated `ThemeProvider.theme` prop in new code.
- Do not invent `mode="system"`, `setMode`, or theme persistence on the provider.
- Do not create theme objects inline on every render.
- Do not assume a nested provider inherits the parent's custom tokens.
- Do not let multiple top-level providers target `body` with different themes.
- Do not forget the portal class for app-owned portals rendered outside the scope.
- Do not remove OpenUI CSS imports after adding the provider.
- Do not copy token names from a different release without checking installed source.
