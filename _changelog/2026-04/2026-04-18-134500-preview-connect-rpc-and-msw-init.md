# @scenar/preview: Connect-RPC handler builder and MSW service worker init

**Date**: 2026-04-18
**Packages**: `@scenar/preview`, `@scenar/cli`

## What changed

### `@scenar/preview/connect` — new subpath export

A generic Connect-RPC MSW handler builder that works with any
`@bufbuild/protobuf` service descriptor. Produces correctly-pathed
MSW `http.post(...)` handlers matching the Connect-RPC URL convention
(`{base}/{ServiceTypeName}/{MethodName}`).

- `connectHandler(service, method, handler, options?)` — single method
- `connectHandlers(service, { method1: fn, method2: fn })` — batch

This is protocol-specific, not product-specific: zero knowledge of any
particular application's protos. Any Scenar user with Connect-RPC
services can use it.

New peer dependency: `@bufbuild/protobuf >=2.0.0` (optional).

### `scenar preview init` — MSW service worker generation

`scenar preview init` now automatically generates `mockServiceWorker.js`
in the project's public directory (detected from the framework config).
This eliminates the manual `npx msw init public/` step.

- If MSW CLI is available, delegates to `npx msw init`
- Falls back to writing a minimal compatible service worker script
- Preserves existing service worker files (user-owned pattern)
- Framework-aware: Next.js, Vite, CRA, Remix all resolve to `public/`

## Migration

No breaking changes. Existing `@scenar/preview` and `@scenar/preview/runtime`
imports are unaffected. The new `./connect` subpath is opt-in.
