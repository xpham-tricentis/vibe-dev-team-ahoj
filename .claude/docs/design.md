# Aura Design System Guide

## What is this?

Aura is Tricentis's production design system. This document is the design reference for anyone building with Aura tokens — whether in a React application like Catch The Vibe or in a standalone HTML prototype.

The canonical token values are extracted from the production `aura-ui` MUI theme.

**Source of truth:** `aura-ui/src/constants/themeOptions.tsx`
**Figma library:** `https://www.figma.com/design/zQUSAXYLgX94Ib5CyjNQX0`
**Product Figma (Aria):** `https://www.figma.com/design/mhpSVhQcyRTVJjT7Z1Actu/Aria`

---

## Using Aura in Catch The Vibe (React)

Catch The Vibe is a React SPA built with Vite. It does **not** use `aura-tokens.css` or CSS custom properties. Instead, Aura tokens are implemented as a JavaScript object (`T`) at the top of `src/App.jsx`, and all components use inline styles referencing `T.{tokenName}`.

### Why inline styles instead of CSS

This is a project constraint, not a preference. The project rules specify: no CSS files, no Tailwind, no styled-components. Inline styles with a centralized token object give us design system consistency without adding a styling dependency.

### The T object

The `T` object at the top of App.jsx maps token names to their Aura values:

```jsx
const T = {
  bgPage: "#F4F4F5",        // --aura-bg-page
  bgPaper: "#FFFFFF",        // --aura-bg-paper
  primary: "#3078C0",        // --aura-primary-main
  textPrimary: "#18181B",    // --aura-text-primary
  divider: "#D4D4D8",        // --aura-divider
  radiusMd: 8,               // --aura-radius-md
  font: "'Inter', sans-serif",
  // ... full list in the token mapping table below
};
```

### How to use tokens in components

```jsx
// Always reference T — never hardcode hex values
<div style={{
  background: T.bgPaper,
  border: `1px solid ${T.divider}`,
  borderRadius: T.radiusMd,
  color: T.textPrimary,
  fontFamily: T.font,
}}>
  Content
</div>

// Buttons
<button style={{
  background: T.primary,
  color: T.textOnPrimary,
  border: "none",
  borderRadius: T.radiusXs,
  fontWeight: 600,
}}>
  Action
</button>
```

### Icons

Catch The Vibe uses **lucide-react** for all icons — not Material Icons or Material Symbols. This is a project-level decision. Do not mix icon libraries.

```jsx
import { Shield, Rocket, CheckCircle } from "lucide-react";

<Shield size={20} color={T.primary} />
```

### Logo

The `VibeCodeLogo` component is a custom inline SVG showing a terminal with conversation bubbles, a person figure, and an AI orb. It is not part of the Aura design system — it is project-specific.

```jsx
<VibeCodeLogo size={48} />   // Nav bar
<VibeCodeLogo size={112} />  // Homepage hero
```

The logo uses `currentColor` for terminal/person elements (inherits text color from context) and `T.primary` for AI elements (Aura blue).

### Constraints specific to Catch The Vibe

- **No localStorage or sessionStorage** — not supported in the deployment environment
- **Single-file architecture** — all components live in `src/App.jsx`
- **No CSS files** — all styling is inline via the `T` object
- **No Material Icons** — lucide-react only
- **Light mode only** — Aura default, no dark mode toggle

---

## Token Reference

### Aura-to-T Mapping

This table shows how each Aura CSS custom property maps to the `T` object property used in Catch The Vibe's App.jsx. If you update a token value, update it in both places.

| Aura CSS Variable | T Object Property | Value | Use for |
|-------------------|-------------------|-------|---------|
| `--aura-primary-main` | `T.primary` | `#3078C0` | Primary actions, links, selected states |
| `--aura-primary-dark` | `T.primaryDark` | `#245E9A` | Hover state for primary actions |
| `--aura-primary-light` | `T.primaryLight` | `#D5E8F6` | Light primary backgrounds, borders |
| — | `T.primarySubtle` | `hsla(210,100%,95%,1)` | Very light primary tint (cards, selected nav items) |
| `--aura-secondary-main` | — | `#52525B` | Secondary actions (not currently in T) |
| `--aura-error-main` | `T.error` | `#D32F2F` | Destructive actions, error states |
| — | `T.errorLight` | `#FEF3F2` | Error background tint |
| — | `T.errorBorder` | `#FECDCA` | Error border color |
| `--aura-success-main` | `T.success` | `#2D8630` | Success states, confirmations |
| — | `T.successLight` | `#ECFDF3` | Success background tint |
| — | `T.successBorder` | `#ABEFC6` | Success border color |
| `--aura-warning-main` | `T.warning` | `hsla(23,80%,45%,1)` | Warnings |
| — | `T.warningLight` | `#FFFAEB` | Warning background tint |
| — | `T.warningBorder` | `#FEDF89` | Warning border color |
| — | `T.purple` | `#6941C6` | Purple accent (zone badges, featured items) |
| — | `T.purpleLight` | `#F3EEFB` | Purple background tint |
| — | `T.pink` | `#C11574` | Pink accent |
| `--aura-text-primary` | `T.textPrimary` | `#18181B` | Primary/default text |
| `--aura-text-secondary` | `T.textSecondary` | `#71717A` | Secondary/label text |
| `--aura-text-disabled` | `T.textDisabled` | `#A1A1AA` | Disabled/muted text |
| — | `T.textOnPrimary` | `#FFFFFF` | Text on primary-colored backgrounds (buttons, chat header) |
| `--aura-divider` | `T.divider` | `#D4D4D8` | Borders, dividers |
| — | `T.dividerLight` | `#E4E4E7` | Subtle/lighter borders |
| — | `T.inputStroke` | `#D4D4D8` | Input field borders |
| `--aura-bg-page` | `T.bgPage` | `#F4F4F5` | Page/section backgrounds |
| `--aura-bg-paper` | `T.bgPaper` | `#FFFFFF` | Cards, dialogs, panels, nav bar |
| `--aura-action-hover` | `T.bgHover` | `#FAFAFA` | Hover states |
| `--aura-action-selected` | `T.bgSelected` | `hsla(210,100%,95%,1)` | Selected items |
| — | `T.bgSubtle` | `#F8FAFC` | Subtle background variation |

### Shadows

Not in `aura-tokens.css` — these are used in App.jsx for card elevation and the chat panel:

| T Object Property | Value | Use for |
|-------------------|-------|---------|
| `T.shadowSm` | `0 1px 2px rgba(0,0,0,0.05)` | Nav bar, subtle card hover |
| `T.shadowMd` | `0 4px 12px rgba(0,0,0,0.08)` | Toggle button, elevated cards |
| `T.shadowLg` | `0 8px 24px rgba(0,0,0,0.12)` | Chat panel, modals |

### Border Radius

| Aura CSS Variable | T Object Property | Value | Use for |
|-------------------|-------------------|-------|---------|
| `--aura-radius-xs` | `T.radiusXs` | 4px | Buttons, inputs, tabs |
| `--aura-radius-sm` | `T.radiusSm` | 6px | Menus, popovers, expandable sections |
| `--aura-radius-md` | `T.radiusMd` | 8px | Cards, alerts |
| `--aura-radius-lg` | `T.radiusLg` | 10px | Dialogs, chat panel |
| `--aura-radius-circular` | `T.radiusCircular` | 9999px | Badges, pills, toggle button |

### Typography

| T Object Property | Value | Use for |
|-------------------|-------|---------|
| `T.font` | `'Inter', sans-serif` | All text |
| `T.monoFont` | `'SF Mono', 'Fira Code', 'Consolas', monospace` | Code blocks, AGENTS.md display |

### Spacing

MUI 8px base unit. These are in `aura-tokens.css` but not currently extracted into the `T` object (inline styles use literal pixel values):

| Aura CSS Variable | Value |
|-------------------|-------|
| `--aura-spacing-0_5` | 4px |
| `--aura-spacing-1` | 8px |
| `--aura-spacing-1_5` | 12px |
| `--aura-spacing-2` | 16px |
| `--aura-spacing-3` | 24px |
| `--aura-spacing-4` | 32px |
| `--aura-spacing-5` | 40px |
| `--aura-spacing-6` | 48px |

---

## Using Aura in Standalone Prototypes (HTML)

> This section applies to standalone HTML prototypes hosted on GitHub Pages.
> It does **not** apply to Catch The Vibe, which uses the React/T-object approach above.

### Quick Start

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="../aura-tokens.css">
  <title>My Prototype</title>
  <style>
    /* Prototype-specific styles go here */
  </style>
</head>
<body>
  <!-- Your prototype HTML -->
</body>
</html>
```

`aura-tokens.css` handles: Google Fonts imports (Inter, Material Icons Outlined, Material Symbols Rounded), box-sizing reset and body defaults, all design tokens as CSS custom properties (`--aura-*`), typography classes (`.aura-h1` through `.aura-overline`), component classes (buttons, cards, dialogs, alerts, inputs, tabs, etc.), and dark mode via `data-theme="dark"` on `<html>` or `<body>`.

### Prototype Deployment

Standalone prototypes are hosted on **GitHub Pages** as static sites:

- **No backend.** No Node, no build step, no SSR.
- **Browser APIs only.** Use `localStorage`, `sessionStorage`, `fetch` for JSON, `URL` params.
- **Relative paths.** Each prototype lives in its own subdirectory. Reference the shared CSS with `../aura-tokens.css`.
- **CDN libraries are fine.** Load anything you need via `<script>` tags.

> **Note:** Catch The Vibe is NOT a standalone prototype. It is a React SPA deployed via CI/CD to CloudFront/S3. It does **not** use `localStorage` or `sessionStorage` (these are not supported in its deployment environment). See the React section above.

### Prototype Repo Structure

```
joseph_prototypes/
  aura-tokens.css          <-- shared design tokens
  DESIGN.md                <-- this file
  qtest_review/
    index.html             <-- a prototype
  some_other_prototype/
    index.html
    data.json              <-- local data files are fine
```

---

## Component Classes (CSS)

> These CSS classes are available in `aura-tokens.css` for standalone prototypes.
> Catch The Vibe does not use these classes — it uses inline styles with the `T` object.

### Buttons

Aura buttons **do not use outlines as borders**. The "outlined" variant in Aura actually renders with a filled background (`--aura-bg-page`) and no border, with a lighter hover state. Text is not uppercased.

```html
<!-- Primary CTA -->
<button class="aura-btn aura-btn-contained aura-btn-md">Save Changes</button>

<!-- Secondary (outlined style — filled bg, no border) -->
<button class="aura-btn aura-btn-outlined aura-btn-secondary aura-btn-sm">
  <span class="aura-icon-rounded">check</span> Accept all
</button>

<!-- Destructive -->
<button class="aura-btn aura-btn-outlined aura-btn-error aura-btn-sm">
  <span class="aura-icon-rounded">disabled_by_default</span> Reject all
</button>

<!-- Text button -->
<button class="aura-btn aura-btn-text">Cancel</button>

<!-- Icon button -->
<button class="aura-icon-btn aura-icon-btn-sm">
  <span class="aura-icon">close</span>
</button>
```

### Icons (Standalone Prototypes)

Two icon fonts are loaded by `aura-tokens.css`. Use the class that matches the style you need:

```html
<!-- Material Icons Outlined (square edges) -->
<span class="aura-icon">settings</span>

<!-- Material Symbols Rounded (rounded edges — used in Figma) -->
<span class="aura-icon-rounded">disabled_by_default</span>
```

The Figma library uses **Material Symbols Rounded weight 300** for most icons. Use `.aura-icon-rounded` to match.

> **Catch The Vibe uses lucide-react instead** — see the React section above.

### Other Components

See the CSS file for: `.aura-card`, `.aura-dialog`, `.aura-alert`, `.aura-input`, `.aura-tabs`/`.aura-tab`, `.aura-chip`, `.aura-badge`, `.aura-list-item`, `.aura-accordion`, `.aura-divider`, `.aura-toggle-btn`.

---

## Shared JS Components

Two reusable JS components live alongside `aura-tokens.css`. They self-inject their own styles and expose a simple `init()` API. Both require `aura-tokens.css` to be loaded on the page.

> These are for standalone prototypes. Catch The Vibe has its own React component equivalents (Card, Badge, Expandable).

### App Bar (`aura-app-bar.js`)

The product-level app bar that sits at the top of every app. Fixed position, z-index 100.

```html
<link rel="stylesheet" href="../aura-tokens.css">
<div id="app-bar"></div>
<script src="../aura-app-bar.js"></script>
<script>
  const appBar = AuraAppBar.init('app-bar', {
    product: 'Tosca',
    workspace: 'My Workspace',
    showSearch: true,
    showAgents: true,
    showDownload: false,
    showHelp: true,
    showSettings: true,
    showNotifications: true,
    notificationCount: 1,
    user: { name: 'Tom Sinclair', initials: 'TS' },
    onAction: (action) => console.log(action),
  });

  // Runtime updates:
  appBar.updateBadge(5);
  appBar.updateWorkspace('New Workspace');
</script>
```

The bar renders at `position: fixed; top: 0`. Add `<div class="aura-app-bar-spacer"></div>` below it (or use `padding-top: 48px`) to avoid overlap.

### Nav Rail (`aura-nav-rail.js`)

The left-side navigation rail. Collapsed = 49px icons-only. Expands to 240px with labels and sub-items.

```html
<link rel="stylesheet" href="../aura-tokens.css">
<div id="nav-rail"></div>
<script src="../aura-nav-rail.js"></script>
<script>
  const rail = AuraNavRail.init('nav-rail', {
    items: [
      { id: 'home', text: 'Home', icon: 'home' },
      { id: 'divider', variant: 'divider' },
      { id: 'inventory', text: 'Inventory', icon: 'assignment', items: [
        { id: 'test-cases', text: 'Test Cases', icon: 'add_to_photos' },
        { id: 'shared-actions', text: 'Shared Actions', icon: 'note_add' },
      ]},
      { id: 'agents', text: 'Agents', icon: 'support_agent' },
      { id: 'runs', text: 'Runs', icon: 'directions_run', disabled: true },
    ],
    selected: 'home',
    clipped: true,
    pinned: false,
    onSelect: (id) => console.log('selected:', id),
  });

  // Runtime:
  rail.select('agents');
  rail.expand();
  rail.collapse();
</script>
```

**Behavior (matches aura-ui NavRail.tsx):**
- **Collapsed (default):** 49px wide, icons only, blue selection indicator on right edge
- **Expanded:** 240px wide, rounded right corners, drop shadow, labels + sub-item expand/collapse
- **Toggle button:** Chevron at bottom toggles open/closed
- **Sub-items:** Expand inline when rail is open; show a small dropdown arrow indicator when collapsed
- **`clipped: true`:** Offsets top by `--aura-appbar-height` to sit below the app bar

**Icon names** are Material Symbols Rounded ligatures (loaded by `aura-tokens.css`). Find icons at [fonts.google.com/icons](https://fonts.google.com/icons?icon.set=Material+Symbols&icon.style=Rounded).

### Typical Page Skeleton

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="../aura-tokens.css">
  <title>My Prototype</title>
  <style>
    .app { display: flex; flex-direction: column; height: 100vh; }
    .content-row { display: flex; flex: 1; overflow: hidden; padding-top: 48px; }
    .main { flex: 1; margin-left: 49px; overflow: auto; padding: 16px; }
  </style>
</head>
<body>
  <div class="app">
    <div id="app-bar"></div>
    <div class="content-row">
      <div id="nav-rail"></div>
      <div class="main">
        <!-- your content -->
      </div>
    </div>
  </div>
  <script src="../aura-app-bar.js"></script>
  <script src="../aura-nav-rail.js"></script>
  <script>
    AuraAppBar.init('app-bar', { product: 'My App', workspace: 'Dev' });
    AuraNavRail.init('nav-rail', {
      items: [
        { id: 'home', text: 'Home', icon: 'home' },
        { id: 'settings', text: 'Settings', icon: 'settings' },
      ],
      selected: 'home',
    });
  </script>
</body>
</html>
```

---

## Working with Third-Party Libraries

These apply to both standalone prototypes and React applications. Apply Aura tokens wherever possible.

### General

- In standalone prototypes: load libraries from CDN via `<script>` tags.
- In React (Catch The Vibe): use packages from the Tricentis private registry only.
- Apply Aura token values to library configs (colors, fonts, border-radius).
- Wrap library containers in Aura-styled panels to maintain visual consistency.

### React Flow / Xyflow

```html
<script src="https://cdn.jsdelivr.net/npm/@xyflow/react/dist/umd/index.js"></script>
<link href="https://cdn.jsdelivr.net/npm/@xyflow/react/dist/style.css" rel="stylesheet">
```

Style custom nodes with Aura tokens. Use `--aura-primary-main` / `T.primary` for selected edges, `--aura-divider` / `T.divider` for default edges.

### Recharts

```html
<script src="https://cdn.jsdelivr.net/npm/recharts/umd/Recharts.min.js"></script>
```

Map chart colors to Aura palette: primary for main series, error/success/warning for status indicators.

### Leaflet

```html
<link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css">
<script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
```

Style popups and controls with Aura tokens via CSS overrides.

### D3 / Arborist / Tree Views

For tree components, match the aura-ui tree view styles:
- Selected: `--aura-action-selected` / `T.bgSelected` background, `--aura-primary-main` / `T.primary` text
- Hover: `--aura-action-hover` / `T.bgHover` background
- Use `--aura-spacing-*` for indent levels

---

## Dark Mode

Add `data-theme="dark"` to `<html>` or `<body>`. All `--aura-*` CSS tokens automatically switch to their dark values.

```js
document.documentElement.toggleAttribute('data-theme');
// or
document.documentElement.dataset.theme =
  document.documentElement.dataset.theme === 'dark' ? '' : 'dark';
```

> **Catch The Vibe uses light mode only.** The `T` object contains light-mode values. If dark mode support is added in the future, the `T` object would need to be made reactive (e.g., read from a context provider or media query).

---

## Keeping Tokens in Sync

If the Aura theme changes upstream, re-extract tokens from `aura-ui/src/constants/themeOptions.tsx`. The CSS file has a "Last synced" date comment at the top. The token naming is stable — color semantics and spacing scale rarely change.

**For Catch The Vibe:** also update the `T` object in `src/App.jsx`. Use the Aura-to-T mapping table in this document to ensure both representations stay aligned.

---

## Figma Workflow

When building from Figma designs:

1. Use the Figma MCP tools (`get_design_context`, `get_screenshot`) to pull design specs
2. Map Figma's CSS variable names to `--aura-*` tokens (for prototypes) or `T.*` properties (for Catch The Vibe)
3. Figma uses `Inter` font — loaded by `aura-tokens.css` for prototypes, specified in `T.font` for React
4. Figma icons come from Material Symbols — loaded by `aura-tokens.css` for prototypes. **Catch The Vibe uses lucide-react instead** — find the equivalent icon by name.

### Common Figma-to-token mappings

| Figma variable | CSS token | T object property |
|---------------|-----------|-------------------|
| `--aura-background-paper` | `--aura-bg-paper` | `T.bgPaper` |
| `--aura-background-default` | `--aura-bg-page` | `T.bgPage` |
| `--aura-text-default` | `--aura-text-primary` | `T.textPrimary` |
| `--aura-text-secondary` | `--aura-text-secondary` | `T.textSecondary` |
| `--aura-text-disabled` | `--aura-text-disabled` | `T.textDisabled` |
| `--aura-borders-divider` | `--aura-divider` | `T.divider` |
| `--aura-borders-input-stroke` | `--aura-divider` | `T.inputStroke` |
| `--aura-actions-primary-default` | `--aura-primary-main` | `T.primary` |
| `--aura-actions-secondary-default` | `--aura-secondary-main` | — |
| `--aura-severity-OK-muted` | `--aura-alert-success-bg` | `T.successLight` |
| `--aura-background-active-focused` | `--aura-action-selected` | `T.bgSelected` |

---

*Last updated: April 2026 — aligned with Catch The Vibe v2.1*
