# Edge TTS Provider + Dead Dependency Cleanup

**Date**: April 18, 2026

## Summary

Added Microsoft Edge TTS as a third narration provider for the Scenar CLI, porting the proven approach from Stigmer's docs narration pipeline. Also removed the unused `@bufbuild/protobuf` dependency from `@scenar/sdk` and verified end-to-end narration audio generation.

## Problem Statement

The CLI's `scenar narrate` command shipped with two TTS providers — Echogarden (offline, requires heavy native install) and OpenAI (requires paid API key). Neither was readily available for quick testing or first-run experience.

### Pain Points

- Echogarden requires explicit install of a large GPL v3 package with native dependencies
- OpenAI requires an API key and costs money per request
- No zero-friction path to test narration out of the box
- `@bufbuild/protobuf` was declared as a dependency in `@scenar/sdk` but never imported — leftover from pre-simplification proto tooling

## Solution

Added `edge-tts-universal` (the same library Stigmer uses for docs narration) as a third TTS provider option. It requires only a network connection — no API keys, no native dependencies, no cost. Also cleaned up the dead dependency from the SDK.

## Implementation Details

### New: Edge TTS Provider (`packages/cli/src/tts/edge-tts.ts`)

- Implements `TtsProvider` interface with dynamic import of `edge-tts-universal`
- Duration extraction from word-boundary subtitle metadata (100-nanosecond units), same algorithm as Stigmer's `generate-narration.ts`
- Bitrate-based fallback (48 kbps mono MP3) when subtitle metadata is unavailable
- Default voice: `en-US-AndrewMultilingualNeural` (matches Stigmer)
- `isEdgeTtsAvailable()` availability check for the resolver

### Updated: Provider Resolution (`resolve-provider.ts`)

- Three-way resolution: `echogarden | edge-tts | openai`
- Error messages cross-reference all three options with install instructions
- `TtsProviderName` type updated to include `"edge-tts"`

### License Handling

- `edge-tts-universal` is AGPL-3.0 — same conflict pattern as Echogarden's GPL v3
- Added as **optional peer dependency** (identical pattern to Echogarden)
- Also added as **devDependency** for testing without affecting CLI's Apache-2.0 license

### Dead Dependency Removal

- Removed `@bufbuild/protobuf` from `@scenar/sdk` dependencies
- Confirmed zero imports across all SDK source files

## Benefits

- Zero-friction narration testing: `pnpm add edge-tts-universal` then `scenar narrate demo.yaml --tts edge-tts`
- Consistency with Stigmer's narration pipeline (same library, same voice, same duration algorithm)
- Clean dependency tree in SDK (no phantom packages)
- Three TTS tiers: free offline (Echogarden), free online (Edge TTS), paid high-quality (OpenAI)

## Impact

- **CLI users**: New `--tts edge-tts` flag for free online narration
- **SDK consumers**: Smaller install footprint (one fewer unused dependency)
- **Test suite**: 130 tests across 4 packages (up from 124), 6 new tests for Edge TTS

## Related Work

- Stigmer narration pipeline: `site/scripts/generate-narration.ts` (source of the Edge TTS pattern)
- Previous changelog: `2026-04-18-081138-scaffold-scenar-cli-with-validate-and-narrate.md` (T02 CLI scaffolding)
- Proto simplification: `2026-04-18-074219-proto-simplification-to-scenario-only.md` (T01)

---

**Status**: ✅ Production Ready
**Timeline**: Single session (~30 minutes)
