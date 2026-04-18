# CLI Narrate: TypeScript Support, Directory Scanning, and Caching

**Date**: April 18, 2026

## Summary

Evolved the Scenar CLI's `narrate` command from a single-YAML-file tool into a full narration pipeline that supports TypeScript scenario files, batch directory scanning, and hash-based caching. Fixed a manifest format mismatch between the CLI output and the `@scenar/core` runtime. This enables consumers like Stigmer to replace bespoke narration scripts with a single `scenar narrate` invocation.

## Problem Statement

The `scenar narrate` command had three limitations that prevented real-world adoption:

### Pain Points

- Only accepted YAML scenario files — but the primary authoring format for Scenar scenarios in practice is TypeScript (`steps.ts` with `ScenarioStep<T>`)
- Could only process one file at a time — no batch mode for projects with 25+ scenarios
- No caching — every run regenerated all audio from scratch, even when narration text hadn't changed
- The manifest output format (`{ generatedAt, ttsProvider, steps: [{ index, file, durationMs }] }`) didn't match what `@scenar/core`'s `NarrationManifest` type expected (`{ steps: [{ src, durationMs } | null] }`), meaning the CLI's output was unusable by the runtime

## Solution

Extended the `narrate` command to accept three input modes: a YAML file, a TypeScript file, or a directory containing scenario subdirectories. Added SHA-256 hash-based caching and fixed the manifest output format to match the runtime contract.

## Implementation Details

**New files:**
- `packages/cli/src/util/load-ts.ts` — Duck-typed dynamic import of TypeScript steps files. Scans exported values for arrays with `delayMs` (same proven approach used in Stigmer's original narration script). Relies on the caller's TypeScript loader (e.g., `tsx`) rather than adding a TS compilation dependency.
- `packages/cli/src/util/discover-scenarios.ts` — Directory scanner that finds `*/steps.ts` subdirectories and returns scenario IDs with paths.
- `packages/cli/src/util/narration-cache.ts` — Hash-based caching: SHA-256 of `voice + narration text`, stored in `.narration-cache.json` alongside the output. Skips TTS when the hash matches and the MP3 file exists.

**Modified files:**
- `packages/cli/src/commands/narrate.ts` — Rewritten to support file/directory detection, TS/YAML loading, caching, and the corrected manifest format. Added `--base-url` option for configuring the `src` path prefix in manifests.
- `packages/cli/src/tts/types.ts` — Deprecated the old CLI-specific manifest types (kept for backward compatibility during transition).

**Manifest format fix:**
- Before: `{ generatedAt, ttsProvider, steps: [{ index, file, durationMs, text }] }` (sparse, metadata-rich)
- After: `{ steps: [{ src, durationMs } | null] }` (positional array matching `@scenar/core`'s `NarrationManifest`)

**Package exports fix (v0.0.4):**
- Added `default` condition to all package.json exports maps (`@scenar/core`, `@scenar/sdk`, `@scenar/react`) to resolve `ERR_PACKAGE_PATH_NOT_EXPORTED` when tsx's CJS resolver encounters the exports map.

## Benefits

- **Zero-config narration for TS projects**: `scenar narrate ./scenarios/ --out ./public/demos/ --tts edge-tts` processes all scenarios in one command
- **Fast re-runs**: Hash-based caching means only changed narrations trigger TTS calls — 106 cached files resolved in <2 seconds in Stigmer's test run
- **Correct runtime integration**: Manifests now match what `useNarrationManifest` and `useNarrationPlayback` actually read
- **Narration text field compatibility**: Reads both `narration` (TS convention) and `narrationText` (YAML/proto convention)

## Impact

- Stigmer replaced its 323-line bespoke narration script with a 22-line wrapper that calls `import { run } from "@scenar/cli"`
- Any Scenar consumer with TypeScript scenarios can now generate narration audio without writing custom tooling
- Published as v0.0.3 (features) and v0.0.4 (exports fix)

## Related Work

- Scenar CLI initial scaffolding (v0.0.1 — validate + narrate commands)
- Scenar CLI Edge TTS provider (v0.0.2 — same TTS engine used by Stigmer)
- Stigmer demo engine rewiring to `@scenar/react` (same session — separate changelog in Stigmer repo)

---

**Status**: Production Ready
**Timeline**: Single session (April 18, 2026)
