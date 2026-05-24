# Typewriter Theme Port — Design Spec

**Date:** 2026-05-23
**Status:** Approved

## Overview

Port the [Obsidian Typewriter theme](https://github.com/crashmoney/obsidian-typewriter) to Quartz so the personal site at ralphhaddad.xyz matches the Obsidian writing environment. The source theme is installed locally at `content/.obsidian/themes/Typewriter/theme.css`.

## Approach

Full fidelity: iA Writer fonts (extracted from the Obsidian theme CSS as embedded base64) + full color palette port. No external font downloads or hosting required.

## Files Changed

| File | Change |
|------|--------|
| `quartz.config.ts` | `fontOrigin: "local"`, typography names updated, all color values replaced |
| `quartz/styles/custom.scss` | 6 `@font-face` blocks extracted from Obsidian theme CSS |

No other files change.

## Colors

### Light Mode

| Variable | Value | Role |
|----------|-------|------|
| `light` | `#fcf5e4` | Warm parchment page background |
| `lightgray` | `#e4dcc8` | Sidebar background, borders |
| `gray` | `#b8b0a0` | Subtle borders, graph lines |
| `darkgray` | `#595959` | Body text |
| `dark` | `#262626` | Headings, strong emphasis |
| `secondary` | `#519D5C` | Links, active nav items |
| `tertiary` | `#6db87a` | Hover states |
| `highlight` | `rgba(81, 157, 92, 0.1)` | Background highlight |
| `textHighlight` | `#e5b56788` | Text selection / mark highlight (warm amber) |

### Dark Mode

| Variable | Value | Role |
|----------|-------|------|
| `light` | `#262626` | Near-black page background |
| `lightgray` | `#3a342e` | Warm dark sidebar |
| `gray` | `#646464` | Borders |
| `darkgray` | `#c5b8a1` | Warm cream body text |
| `dark` | `#d9cebc` | Headings (slightly brighter than body) |
| `secondary` | `#6db87a` | Links (brighter green for dark-mode readability) |
| `tertiary` | `#84a998` | Hover states |
| `highlight` | `rgba(81, 157, 92, 0.15)` | Background highlight |
| `textHighlight` | `#e5b56744` | Text highlight (amber, more subtle) |

Color derivation: light mode background is `hsl(44, 79%, 94%)` = `#fcf5e4` from the theme's `--background-primary`. Dark mode text is `rgb(197, 184, 161)` = `#c5b8a1` from `--text-normal`. Green accent `#519D5C` is the theme's default `--color-accent` (hsl 129, 31.9%, 46.7%).

## Typography

| Quartz slot | Font | Source in Obsidian theme |
|-------------|------|--------------------------|
| `header` | iA Writer Quattro S | `--font-text-theme` |
| `body` | iA Writer Quattro S | `--font-text-theme` |
| `code` | iA Writer Mono V | `--font-editor-theme` |

`fontOrigin` is set to `"local"` to disable Google Fonts CDN. Fonts are loaded via `@font-face` in `custom.scss`.

## Font Extraction

The Obsidian theme CSS at lines 22–78 contains 6 `@font-face` blocks with fonts embedded as base64 woff2 `data:` URLs:

- Line 22: `iA Writer Mono V` (regular)
- Line 34: `iA Writer Quattro S` (regular)
- Line 42: `iA Writer Quattro S` (italic)
- Line 50: `iA Writer Quattro S` (bold)
- Line 58: `iA Writer Quattro S` (bold italic)
- Line 67: `JetBrains Mono` (not needed — skip)

Extract lines 22–66 (the 5 iA Writer blocks, excluding JetBrains Mono) and prepend them to `custom.scss`.

## quartz.config.ts Changes

```ts
theme: {
  fontOrigin: "local",        // was "googleFonts"
  cdnCaching: true,           // unchanged
  typography: {
    header: "iA Writer Quattro S",   // was "Schibsted Grotesk"
    body: "iA Writer Quattro S",     // was "Source Sans Pro"
    code: "iA Writer Mono V",        // was "IBM Plex Mono"
  },
  colors: {
    lightMode: {
      light: "#fcf5e4",
      lightgray: "#e4dcc8",
      gray: "#b8b0a0",
      darkgray: "#595959",
      dark: "#262626",
      secondary: "#519D5C",
      tertiary: "#6db87a",
      highlight: "rgba(81, 157, 92, 0.1)",
      textHighlight: "#e5b56788",
    },
    darkMode: {
      light: "#262626",
      lightgray: "#3a342e",
      gray: "#646464",
      darkgray: "#c5b8a1",
      dark: "#d9cebc",
      secondary: "#6db87a",
      tertiary: "#84a998",
      highlight: "rgba(81, 157, 92, 0.15)",
      textHighlight: "#e5b56744",
    },
  },
},
```

## Out of Scope

- Obsidian-specific UI chrome (sidebars, panels, modals) — Quartz has a different layout
- Style Settings plugin variables
- Active line highlight, vim cursor styling
- JetBrains Mono (not used in Quartz)
