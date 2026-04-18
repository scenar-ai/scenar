# Shell Chrome Realism & Universal CSS Architecture

**Date**: April 18, 2026

## Summary

Upgraded all three shell chrome components (BrowserView, TerminalView, CodeEditorView) with realistic proportions and missing UI furniture, then introduced a universal CSS architecture that makes `@scenar/react` fully self-contained for non-Tailwind hosts while preserving the existing Tailwind-native integration path.

## Problem Statement

Shell chrome components rendered at miniaturized proportions that looked like thumbnail sketches rather than realistic application frames. Traffic lights were 8–10px instead of 12px, tab text was 9–10px, and critical UI landmarks (new-tab button, profile avatar, kebab menu, status bar) were missing. Additionally, the package shipped Tailwind utility class names in compiled JS but no corresponding CSS, making it unusable for hosts without Tailwind.

### Pain Points

- BrowserView lacked new-tab button, profile avatar, bookmark star, and three-dot menu — missing the most recognizable Chrome landmarks
- TerminalView had 8–9px tab text and 8px icons — toy-sized relative to the 11px terminal body
- CodeEditorView had 8px traffic lights (smallest of all), no status bar, and 9px explorer text
- Non-Tailwind hosts could not use the package — no pre-built CSS bundle
- Player controls used shadcn semantic tokens (`text-foreground`, `bg-card`, etc.) that only resolved if the host defined specific CSS variables
- Stigmer's `StigmerDemoViewport` never applied the `scenar` class scope, so `--scenar-*` variables didn't resolve (worked by accident)

## Solution

Two-part approach: (1) scale up all shell chrome to realistic proportions and add missing UI elements, (2) ship a pre-built `styles.css` alongside the existing `theme.css` with an expanded token contract that bridges `--scenar-*` to Tailwind's `--color-*` namespace.

## Implementation Details

### Shell realism (3 components, ~130 lines changed)

**BrowserView** — Traffic lights 10px → 12px; tab text `text-[10px]` → `text-xs`; tab max-width 140px → 180px; nav icons 14px → 16px; address bar padding `py-1` → `py-1.5` with `text-xs` URL; added `Plus` new-tab button, profile avatar circle, `Star` bookmark, extension dot, and `EllipsisVertical` kebab menu.

**TerminalView** — Traffic lights 10px → 12px; title text `text-[10px]` → `text-xs`; tab text `text-[9px]` → `text-[11px]`; all tab icons 8px → 12px.

**CodeEditorView** — Traffic lights 8px → 12px; title text `text-[9px]` → `text-xs`; activity bar width 28px → 36px with 16px icons; explorer text 9–10px → 11px; editor tab text `text-[10px]` → `text-xs`; file tree icons 12px → 14px; added VS Code-style blue status bar with branch, line/col, encoding, and language indicators.

### Universal CSS architecture (4 new/modified files)

| File | Change |
|------|--------|
| `src/theme/tokens.css` | Expanded from 3 to 8 `--scenar-*` tokens (added muted-foreground, card, accent, primary, ring) |
| `src/styles/index.css` | New Tailwind v4 entry point with `@source` scanning and `@theme inline` bridge |
| `package.json` | Added `@tailwindcss/cli` + `tailwindcss` devDeps, `styles.css` export, extended build script |
| `README.md` | Documented both integration modes with code examples and token reference table |

### Two integration modes

**Self-contained (any host):** Import `@scenar/react/styles.css` (24KB minified). Contains all Tailwind utilities, the `--scenar-*` tokens, and a bridge that maps `--scenar-foreground` → `--color-foreground`, etc. Wrap content in `.scenar` class.

**Tailwind-native (existing hosts):** Import `@scenar/react/theme.css` (645B). Host's Tailwind generates utilities via `@source`. Host's own theme tokens drive player control appearance.

### Token architecture

```
@layer scenar {
  .scenar {
    --scenar-surface      →  shell content background
    --scenar-border       →  shell border color
    --scenar-foreground   →  primary text
    --scenar-muted-foreground →  secondary text
    --scenar-card         →  dropdown/popover background
    --scenar-accent       →  hover state
    --scenar-primary      →  active indicator
    --scenar-ring         →  focus ring
  }
}
```

In `styles.css`, these bridge to Tailwind's `--color-*` namespace via `@theme inline`, making semantic classes like `text-foreground` and `bg-card` resolve against Scenar's own tokens inside the `.scenar` scope.

## Benefits

- **Realistic chrome**: All three shells now pass the "looks like the real app" test at demo scale — proper traffic light sizing, complete toolbar furniture, VS Code status bar
- **Universal portability**: Any React host can embed Scenar scenarios with a single CSS import — no Tailwind, shadcn, or special build config required
- **Backward compatible**: Existing `theme.css` import path expanded additively; all component props and behavior unchanged
- **Clean token contract**: 8 well-documented CSS custom properties cover every semantic color the package needs
- **Build verified**: 21 tests pass across 8 test files; both CSS files compile cleanly

## Impact

- `@scenar/react` public API gains `./styles.css` export (additive)
- `theme.css` token set expands from 3 → 8 (additive, no breaking change)
- Shell chrome visually improved across all Stigmer demo scenarios
- Package is now viable for third-party hosts without Tailwind
- Total `@scenar/react` shipped CSS: 645B tokens-only + 24KB self-contained

## Related Work

- **T04** (Chrome Shell Primitives): Extracted shells into `@scenar/react` — this session improves their visual fidelity
- **Stigmer integration**: `StigmerDemoViewport` updated to apply `scenar dark` class scope (separate commit in Stigmer repo)

---

**Status**: ✅ Production Ready
**Timeline**: Single session
