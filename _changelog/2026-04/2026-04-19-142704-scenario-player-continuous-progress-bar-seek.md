# ScenarioPlayer continuous progress bar seek

**Date**: April 19, 2026

## Summary

Replaced discrete “jump to step start” behavior on the `ScenarioPlayer` progress bar with continuous, time-accurate seeking (similar to common video players). Clicks map directly to a wall-clock position on the precomputed step timeline; narration audio can seek within the current step’s clip; playback resumes after a bar click instead of pausing at a step boundary.

## Problem Statement

The progress bar looked like a continuous track with chapter ticks, but clicks resolved only to a **step index** and `goTo` paused at that step’s start. Users perceived a mismatch between pointer position and where playback landed.

### Pain Points

- Clicking mid-segment jumped to the **beginning** of the containing step, not the clicked time
- Intra-step position was computed then discarded before advancing state
- `usePlaybackProgress` always reset step-local elapsed time to zero on step index changes, reinforcing the snap-to-start feel
- Narration had no path to align with a mid-step seek when audio was unmuted

## Solution

Introduce a **time-based seek path** alongside the existing step-based `goTo`:

1. **`seekToTime(timeMs, stepTimeline)`** in `useStepProgression` — derives the active step with `deriveStepFromTime`, stores intra-step offset in a ref, bumps a generation counter for same-step seeks, and sets playback to **playing** (YouTube-style resume after bar click).
2. **`usePlaybackProgress`** — on step index or seek generation change, initializes `stepElapsedRef` from the offset ref (then clears the ref) instead of always zeroing.
3. **`useNarrationPlayback.seekToStep(stepIndex, offsetMs)`** — loads the clip, seeks after metadata when needed, skips audio if offset is past clip duration; a short-lived guard prevents the normal step-change effect from restarting the clip from zero on top of the imperative seek.
4. **`ScenarioControls`** — exposes `onSeekToTime(timeMs)` only; click handler passes fractional timeline time through unchanged.
5. **`ScenarioPlayer`** — `handleSeekToTime` clamps time, calls `seekToStep` then `seekToTime`, and notifies the playback coordinator so multi-player pages stay consistent.

`goTo(index)` remains for discrete navigation (e.g. future prev/next controls); Remotion / `TimeSourceProvider` paths are unchanged because controls are hidden in export mode.

## Implementation Details

### Files changed

- `packages/react/src/player/useStepProgression.ts` — `seekToTime`, `seekOffsetRef`, `seekGeneration`
- `packages/react/src/player/usePlaybackProgress.ts` — new parameters; effect deps include `seekGeneration`
- `packages/react/src/player/ScenarioControls.tsx` — `onSeekToTime`; removed `stepIndex` / `lastIndex` props
- `packages/react/src/player/ScenarioPlayer.tsx` — `handleSeekToTime`, wiring to narration and coordinator
- `packages/react/src/narration/useNarrationPlayback.ts` — `seekClip` helper, `seekToStep`, guarded step effect

### Breaking change (direct consumers of `ScenarioControls`)

`ScenarioControls` is part of the public package surface. Integrations that passed `onSeek`, `stepIndex`, or `lastIndex` must switch to `onSeekToTime(timeMs)` and drop the unused props.

### Commit

- `84b745f` — `feat(react): add continuous seek to ScenarioPlayer progress bar`

## Benefits

- Progress bar behavior matches user expectations for a “video-like” control
- Narration can stay aligned with the visual timeline when unmuted
- Same-step seeks still update the playhead correctly (generation bump)
- No changes required in `@scenar/core` timeline math (`computeStepTimeline`, `deriveStepFromTime`)

## Impact

- **Docs and embeds** using `ScenarioPlayer` only: no API change; seeking improves automatically
- **Custom UI** built on `ScenarioControls`: must adopt `onSeekToTime` and remove obsolete props
- **Video export / Remotion**: unchanged (no interactive bar; time driven by `TimeSourceProvider`)

## Related Work

- Planning discussion: continuous seek for `ScenarioPlayer` progress bar (discrete vs continuous timeline model)
- Stigmer site verification: local `link:` to `@scenar/react` / `@scenar/core` / `@scenar/preview` (revert before shipping site if not publishing yet)

---

**Status**: ✅ Production Ready
**Timeline**: ~1 session (implementation + tests + manual verification on Stigmer docs)
