# @scenar/sdk — Typed Authoring Surface and Engine/Proto Reconciliation

**Date**: April 17, 2026

## Summary

Built `@scenar/sdk`, a pure TypeScript authoring layer that bridges the Scenar proto contract with the playback engine. As a prerequisite, reconciled two drifts between the proto contract (T01) and the engine (T03): inlined interactions onto `ScenarioStep<T>` and aligned ActionType string literals with proto enum names. The SDK ships `createScenario()` for typed TS-first authoring and `loadScenarioFromProto()` for the YAML/hosted-service ingestion path.

## Problem Statement

After T01 defined the proto contract and T03 extracted the engine, the two didn't agree on two fundamental shapes — making the SDK's bridge role unnecessarily complex and lossy.

### Pain Points

- **Interactions shape mismatch**: Proto embeds `interactions: StepAction[]` on each `Step`. Engine used a separate `Record<number, StepAction[]>` map keyed by step index — fragile on reorder and requiring a conversion layer.
- **ActionType naming mismatch**: Proto enum values are snake_case (`set_cursor`). Engine used kebab-case (`"set-cursor"`). Every proto-to-engine conversion would need a bidirectional string mapping table.
- **No typed authoring surface**: Scenario authors had to manually construct `ScenarioStep<T>[]` arrays and wire `StepInteractions` maps by hand, with no type safety between view identifiers and their props.
- **No proto ingestion path**: No way to load a proto-serialized `Scenario` message into the engine's type system.

## Solution

Three-phase approach: reconcile the engine first, then build the SDK on a clean foundation.

**Phase 1 — Engine reconciliation**: Migrate engine types to match proto shapes exactly. `ScenarioStep<T>` gains `interactions?: readonly StepAction[]` inline. `ActionType` strings become snake_case. The `StepInteractions` type is removed entirely.

**Phase 2 — @scenar/sdk**: A new package with zero React dependency that provides:
- `createScenario()` — typed builder where `props` is statically validated against the component registered under each `view`
- `loadScenarioFromProto()` — proto adapter using structural typing (no generated stubs dependency)

**Phase 3 — Tests**: 31 unit tests covering the builder, all 8 action types, config oneof mapping, error paths, and edge cases.

## Implementation Details

### Engine changes (packages/core, packages/react)

- `ScenarioStep<T>` in `types.ts` — added `readonly interactions?: readonly StepAction[]` with import of `StepAction`; all fields made `readonly`
- `step-action.ts` — `ActionType` literals changed to snake_case; `StepInteractions` type removed; `UseStepInteractionsOptions.interactions` field removed (interactions read from steps)
- `useBrowserStepInteractions.ts` / `useTimeSourceStepInteractions.ts` — read `steps[stepIndex]?.interactions` instead of `interactions[stepIndex]`; all action-type matchers updated
- `warnings.ts`, `dom-helpers.ts`, effect files — updated console.warn prefixes from `[StepInteractions]` to `[scenar]`; action-type string literals updated

### SDK package (packages/sdk)

| File | LOC | Purpose |
|------|-----|---------|
| `author/types.ts` | 75 | `ViewRegistry`, `PropsOf`, `StepInput`, `AuthoredScenario` types |
| `author/createScenario.ts` | 60 | Typed builder with runtime view validation |
| `proto/proto-types.ts` | 100 | Structural interfaces matching protoc-gen-es v2 output |
| `proto/action-mapper.ts` | 75 | Proto StepAction → engine StepAction with oneof expansion |
| `proto/load-scenario.ts` | 95 | Proto Scenario → AuthoredScenario with envelope validation |
| `proto/errors.ts` | 15 | `InvalidScenarioError` with `.path` and `.reason` |
| `index.ts` | 30 | Barrel export |

### Key type-level design

```ts
type ViewRegistry = Record<string, (props: never) => unknown>;

type StepInput<Views extends ViewRegistry> = {
  [K in keyof Views & string]: {
    readonly view: K;
    readonly props: PropsOf<Views[K]>;
    // ...
  };
}[keyof Views & string];
```

This distributes over view keys so TypeScript narrows `props` based on the `view` discriminant — full static guarantees with zero `any` or `JsonObject`.

### Design decisions documented

- DD-006: Engine interactions inline migration
- DD-007: ActionType snake-case alignment
- DD-008: SDK framework agnosticism
- DD-009: Scenar wrapper component deferred to post-T06

## Benefits

- **Type safety**: `createScenario()` catches props mismatches at compile time — no runtime surprises when a view expects `{ org: string }` but gets `{ org: 123 }`
- **Proto/engine alignment**: Zero mapping code between proto and engine ActionType values — direct string name copy
- **Framework agnosticism**: SDK has zero React dependency; usable by CLI tools, build scripts, and non-React renderers
- **Structural proto typing**: No dependency on generated stubs; consumers pass proto-generated objects and TypeScript validates structurally
- **Error precision**: `InvalidScenarioError` reports the exact JSON path (`spec.steps[2].interactions[0].type`) and reason, not a generic "invalid scenario" message

## Impact

- **@scenar/core**: Breaking change — `StepInteractions` type removed, `ActionType` strings changed to snake_case. Existing consumers (Stigmer demos) need to update in T06.
- **@scenar/react**: Breaking change — `useStepInteractions` no longer accepts a separate `interactions` map. Interactions are read from `ScenarioStep.interactions` inline.
- **New package**: `@scenar/sdk` (0.0.1) added to the monorepo with `@scenar/core` and `@bufbuild/protobuf` as dependencies.
- **Test count**: 79 total (27 core + 21 react + 31 sdk), up from 48.

## Related Work

- T01: Proto contract definition (the canonical schema this SDK bridges to)
- T03: Engine extraction (the runtime types this SDK produces)
- T04: Chrome shell primitives (used by scenarios authored with this SDK)
- T06 (next): Rewire Stigmer demos using `@scenar/sdk` imports

---

**Status**: Production Ready
**Timeline**: Single session
