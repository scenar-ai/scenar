# Future Work: Migrate Stigmer from DemoTransport to PreviewProvider + MSW

**Status**: Not started
**Depends on**: `@scenar/preview` (completed, v0.1.2)

## Context

Stigmer's `@stigmer/react/demo` module provides a product-specific mock
infrastructure for demo scenarios. Now that `@scenar/preview` offers a
generic, protocol-agnostic alternative via `PreviewProvider` + MSW, the
Stigmer-specific mock layer can be incrementally retired.

## What exists today in `@stigmer/react/demo`

| Module | Purpose | Can be replaced? |
|--------|---------|-----------------|
| `DemoTransport` | Fake Connect-RPC transport that intercepts RPCs by `service/method` key | Yes — MSW intercepts at HTTP level, covering Connect and any other protocol |
| `createDemoClient` | Builds a fake `Stigmer` client with all 19 resource clients wired to `DemoTransport` | Yes — `PreviewProvider` + MSW replaces the need for a fake client entirely |
| `fixtures` | Registry of `FixtureSpec` entries tied to Stigmer's proto service descriptors (`fixtures.apiKey.findAll`, `fixtures.agent.getByReference`, etc.) | Yes — MSW handlers replace these with HTTP-level mocks |
| `buildScenario` | Composes `FixtureSpec[]` into a `DemoScenario`, with search multiplexing | Yes — no longer needed when fixtures are HTTP-level |
| `samples` | Factory functions that generate realistic Stigmer protobuf objects (`samples.agent()`, `samples.agentExecution()`, etc.) | **No** — these are domain-specific test data builders, not mock infrastructure. They're used by every scenario's `steps.ts` for step payloads. |

## Migration plan

### Phase 1: Migrate one scenario as proof of concept

Pick a simple scenario (e.g. `quickstart-playback` or `first-skill-tour`)
and convert it from:

```tsx
// Old: product-specific mock transport
const client = createDemoClient(buildScenario(
  fixtures.apiKey.findAll(() => getApiKeyList()),
));
<StigmerProvider client={client}>...</StigmerProvider>
```

To:

```tsx
// New: generic HTTP mocking
import { PreviewProvider } from "@scenar/preview/runtime";
import { http, HttpResponse } from "msw";

<PreviewProvider
  providers={PreviewProviders}
  fixtures={[
    http.post('*/ApiKeyController/FindAll', () =>
      HttpResponse.json({ entries: [...] })
    ),
  ]}
>
  ...
</PreviewProvider>
```

**Validate**: Confirm the scenario renders identically with MSW-backed
data vs the old `DemoTransport`-backed data.

### Phase 2: Migrate remaining scenarios

Convert all ~25 scenarios from `createDemoClient` + `StigmerProvider`
to `PreviewProvider` + MSW fixtures. Each scenario's `index.tsx`
changes; `steps.ts` files are unaffected (they use `samples`, not
fixtures).

### Phase 3: Extract `samples` into a test utility

Move `samples` out of `@stigmer/react/demo` into a standalone package
or `@stigmer/react/test` subpath export. This module is useful beyond
demos — it's also valuable for unit tests, Storybook, and Playwright.

### Phase 4: Delete `@stigmer/react/demo`

Once all scenarios use `PreviewProvider` and `samples` has been
extracted, delete:

- `sdk/react/src/demo/transport.ts` (DemoTransport)
- `sdk/react/src/demo/client.ts` (createDemoClient)
- `sdk/react/src/demo/fixtures.ts` (fixture registry)
- `sdk/react/src/demo/types.ts` (DemoScenario, FixtureEntry, etc.)
- The `"./demo"` subpath export from `sdk/react/package.json`

Keep:
- `samples.ts` (moved to `@stigmer/react/test` or similar)

## Files affected in Stigmer

### `sdk/react/src/demo/` (to be deleted eventually)

- `client.ts` — 78 lines
- `transport.ts` — 116 lines
- `fixtures.ts` — 408 lines
- `types.ts` — 50 lines
- `samples.ts` — keep (extract to test utilities)
- `index.ts` — barrel (update, then delete)

### `site/src/components/docs/demos/` (to be migrated)

Every scenario `index.tsx` that imports from `@stigmer/react/demo`:

- `agent-creation-tour/index.tsx`
- `api-key-setup/index.tsx`
- `approval-flow-playback/index.tsx`
- `byoa-setup/index.tsx`
- `connect-playback/index.tsx`
- `connect-tools-tour/index.tsx`
- `create-agent-tour/index.tsx`
- `first-skill-tour/index.tsx`
- `marketplace-connect-tour/index.tsx`
- `mcp-server-creation-tour/index.tsx`
- `oauth-connect-flow/index.tsx`
- `quickstart-playback/index.tsx`
- `quickstart-tour/index.tsx`
- `session-memory-playback/index.tsx`
- `skill-creation-tour/index.tsx`
- `tool-calls-playback/index.tsx`

Plus `site/src/components/docs/demos/fixtures.ts` (shared helpers).

## Risks and considerations

- **MSW service worker**: The Stigmer docs site would need MSW's
  service worker registered. This is a one-time setup in the site's
  public directory.
- **Connect-RPC over HTTP**: Connect uses POST requests with protobuf
  or JSON encoding. MSW handlers need to match the actual HTTP paths
  that Connect generates (e.g. `/stigmer.iam.apikey.v1.ApiKeyController/FindAll`).
- **Binary protobuf**: If Connect uses binary protobuf encoding, MSW
  handlers need to return binary responses. JSON mode is simpler for
  demos.
- **Incremental migration**: Scenarios can be migrated one at a time.
  Both patterns can coexist during the transition.

## Decision

Not started. This is a follow-up initiative that can be tackled
incrementally. The `@scenar/preview` infrastructure is ready;
the migration is a matter of converting each scenario.
