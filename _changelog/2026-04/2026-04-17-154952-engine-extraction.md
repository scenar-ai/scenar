# T03: Engine Extraction — @scenar/core and @scenar/react

**Date**: April 17, 2026

## Summary

Extracted the generic scenario playback engine from the Stigmer repo into two standalone TypeScript packages in the Scenar monorepo. The original 15-file, 2,880-LOC engine was decomposed into 41 source files (2,873 LOC) across `@scenar/core` (pure TS) and `@scenar/react` (React + Framer Motion), with zero `@stigmer/*` dependencies and 35 passing tests.

## Problem Statement

The Stigmer demo engine lived inside Stigmer's site codebase (`site/src/components/docs/demos/engine/`). It was tightly coupled to Stigmer's proto types, token system, and asset conventions. Every file imported from `@stigmer/*` packages or site-specific paths. Two components were single-responsibility violations: ScenarioPlayer (681 LOC) and useStepInteractions (997 LOC).

### Pain Points

- Engine was not reusable outside of Stigmer's site
- Hard dependency on `@stigmer/protos`, `@stigmer/react`, and site tokens
- ScenarioPlayer was a 681-LOC god object mixing step progression, audio, progress bar, controls UI, poster overlay, intersection observer, and coordinator
- useStepInteractions was a 997-LOC monolith with 8 action types x 2 code paths (browser + video)
- Adding a new interaction type required editing two giant switch/if trees
- No unit tests existed for any engine code

## Solution

Created a pnpm monorepo with two packages following the layered architecture pattern:

- **`@scenar/core`** — Pure TypeScript types, timing constants, timeline computation, DOM scroll utilities, cursor position math, and a centralized data-attribute contract. Zero framework dependencies.
- **`@scenar/react`** — React components (ScenarioPlayer, Cursor, DemoViewport, ViewportTransformLayer) and hooks (useStepInteractions, useNarrationPlayback, useStepProgression) that import from `@scenar/core`.

## Implementation Details

### Package architecture
- `@scenar/core` has no `peerDependencies` — usable from any JS environment
- `@scenar/react` peers on `react`, `react-dom`, `framer-motion`, and optionally `lucide-react`
- TypeScript project references ensure correct build order

### ScenarioPlayer decomposition (681 LOC -> 8 files)
- `useStepProgression` — step advancement logic (timer + time-source dual path)
- `usePlaybackProgress` — RAF-driven 60fps progress bar
- `ScenarioPoster` — play overlay
- `ScenarioControls` — transport bar with progress, play/pause, mute, speed
- `SpeedMenu` — speed selector popover
- `ScenarioPlayer` — slim orchestrator (~243 LOC)

### useStepInteractions decomposition (997 LOC -> 15 files)
- 8 per-action effect files (`effects/click.ts`, `effects/type.ts`, etc.)
- `InteractionContext` interface replaces scattered callback threading
- `useBrowserStepInteractions` — setTimeout path
- `useTimeSourceStepInteractions` — frame-driven path
- `useStepInteractions` — thin dispatcher (~50 LOC)

### DemoViewport decoupled
- Removed dependency on Stigmer's `shared/tokens.ts`
- All layout constants (canonicalWidth, minZoom, shellHeight, wrapperClassName) are now optional props with sensible defaults

### Cursor icons genericized
- Replaced `hsl(var(--foreground))` / `hsl(var(--background))` with `currentColor` / `var(--scenar-cursor-stroke, white)`
- Host apps override cursor appearance via CSS custom properties

### Narration URL configurable
- `useNarrationManifest(id, resolveUrl?)` — second parameter overrides the default `/demos/{id}/manifest.json` convention

## Benefits

- Engine is now a reusable open-source package — any React app can embed scenario playback
- Clean dependency boundaries: core has zero deps, react peers on standard libs
- Adding a new interaction type is a one-file change (create effect + register)
- 35 unit tests provide a correctness baseline that didn't exist before
- No file exceeds 300 LOC (vs. 681 and 997 before)
- Stigmer rewiring (T06) becomes a mechanical import-path change, not an architectural one

## Impact

- **Scenar repo**: Goes from proto-only to a buildable, testable TypeScript monorepo
- **Stigmer repo**: No changes yet — rewiring happens in T06
- **Future consumers**: Any React application can `pnpm add @scenar/react` and get a full scenario playback engine
- **Future tasks**: T04 (shells), T05 (SDK/createScenario), T06 (rewire) are all unblocked

## Related Work

- T01: Proto contract definition (prerequisite, completed)
- T02: Buf configuration (completed via T01)
- T04: Shell extraction (next, depends on T03)
- T05: SDK with createScenario() (depends on T01 + T03)

---

**Status**: Complete
**Timeline**: Single session
