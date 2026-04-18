# Player controls visibility and Tailwind v4 scanning

**Date**: April 18, 2026

## Summary

Fixed a class of invisible-styling bugs where `@scenar/react` player components (progress bar, play button overlays, transport controls) rendered correct HTML but had no CSS in consumer sites. The root cause was that `@scenar/react` did not self-register its source files for Tailwind v4 scanning, unlike the sibling `@stigmer/react` and `fumadocs-ui` packages. Additionally improved play button visibility on dark backgrounds, removed an overly aggressive poster dimming overlay, and fixed a viewport size flicker on page load.

## Problem Statement

Interactive demos embedded on the Stigmer docs site showed invisible progress bars, faint play buttons, and a jarring viewport size jump on page refresh. Multiple attempted fixes (opacity bumps, manual `@source inline()` safelists) failed because they didn't address the root cause.

### Pain Points

- Progress bar track, fill, and playhead completely invisible after playback starts
- Play/pause overlay buttons hard to see on dark demo backgrounds
- Poster overlay dimmed the demo content so heavily it looked unappealing before playback
- Viewport flashed from full-width to correct scaled size on every page load
- Manual `@source inline()` safelists were brittle — every new class required consumer-side updates

## Solution

### Tailwind scanning (the core fix)

Added `@source "./**/*.js"` to `@scenar/react`'s `theme.css` so the package self-registers its compiled JS files for Tailwind v4 class extraction. This mirrors the pattern used by `@stigmer/react` (`@source "./**/*.{ts,tsx}"` in `styles.css`) and `fumadocs-ui` (`@source '../dist/**/*.js'` in `preset.css`). Consumer sites only need to `@import "@scenar/react/theme.css"` — no `@source` configuration required.

### Play button visibility

Added a white glow (`shadow-[0_0_30px_rgba(255,255,255,0.3)]`) and ring (`ring-1 ring-white/30`) to the play/pause button circles so they stand out regardless of underlying content brightness.

### Poster overlay

Removed the `bg-black/50 backdrop-blur-sm` dimming from `ScenarioPoster` and `bg-black/40` from `ScenarioPauseOverlay`. The demo content is now fully visible before playback — only the play button circle sits on top as the click affordance.

### Viewport flicker

Added `overflow-hidden` to the `DemoViewport` outer wrapper. The inner div renders immediately at `zoom: 1`, any first-frame overflow is silently clipped by the container, and the ResizeObserver corrects the zoom within milliseconds. An earlier attempt using `visibility: hidden` was reverted because it caused the entire demo area to vanish until JS hydrated.

### Progress bar stability

Kept the `ScenarioControls` DOM always mounted (opacity animation instead of mount/unmount) so `progressTrackRef` and `playheadRef` stay alive across auto-hide cycles. Bumped track opacity from `/15` to `/25` for better visibility.

## Implementation Details

### Files changed

- `packages/react/src/theme/tokens.css` — added `@source "./**/*.js"`
- `packages/react/src/player/ScenarioPoster.tsx` — removed background tint, blur, added glow/ring to play button
- `packages/react/src/player/ScenarioControls.tsx` — kept DOM mounted, bumped opacity values
- `packages/react/src/viewport/DemoViewport.tsx` — added `overflow-hidden` to wrapper class

### Key architectural decision

Each Tailwind-consuming package should ship its own `@source` directive inside its CSS entry point. This makes scanning self-contained and package-relative — consumer sites don't need to know where `node_modules` lives relative to their CSS file.

### Versions published

- v0.1.8: play button glow + ring
- v0.1.9: stronger controls visibility
- v0.1.10: `@source` in theme.css (the core fix)
- v0.1.11: poster dimming removed, viewport visibility:hidden (later reverted)
- v0.1.12: overflow-hidden viewport fix

## Benefits

- All `@scenar/react` utility classes now reliably appear in consumer Tailwind output
- No consumer-side `@source` configuration or safelist maintenance needed
- Progress bar, play button, and transport controls are fully visible on all backgrounds
- Demo content is clearly visible before playback — no dimming overlay
- No viewport size flash on page load
- Pattern is consistent across all three Tailwind-using packages in the ecosystem

## Impact

- **Scenar consumers**: any project importing `@scenar/react/theme.css` gets automatic Tailwind class scanning
- **Stigmer docs site**: all interactive demo player controls now render with full styling
- **Future development**: new classes added to `@scenar/react` are automatically detected — no manual safelist updates

## Related Work

- Stigmer OSS changelog: `2026-04-18-173249-fix-scenar-tailwind-class-detection.md`
- Stigmer commit `b8f1d126`: consumer-side cleanup (removed dead `@source` lines)

---

**Status**: ✅ Production Ready
**Timeline**: ~3 hours (including root cause investigation across two repos)
