# @scenar/preview — Automated Component Discovery and View Registry Generation

**Date**: April 18, 2026

## Summary

Introduced `@scenar/preview`, a new package that scans React projects and auto-generates a view registry for Scenar scenarios. This eliminates the need for users to hand-write view wrappers or product-specific mock clients. The package includes a CLI (`scenar preview init` / `scenar preview sync`), a component scanner built on ts-morph, a split-ownership code generator, and a runtime `PreviewProvider` with MSW-based network mocking.

## Problem Statement

Creating demos with Scenar required users to manually:
- Build wrapper view components for their app's screens
- Create product-specific mock data layers (e.g. Stigmer's `DemoTransport` — 500+ lines)
- Register views by hand in a view registry
- Set up providers manually

This was high-friction, product-specific work that each adopter had to repeat from scratch.

### Pain Points

- Stigmer's `preview-configs.ts` alone was 1,100+ lines of hand-maintained fixture data
- Each product needed its own mock transport implementation
- No way to discover components automatically
- Source and output had to be in the same project (no monorepo support)

## Solution

A new `@scenar/preview` package that shifts Scenar's pitch from "bring your components" to "point us at your project, we'll do the rest."

### Core capabilities

- **TypeScript AST scanner** (ts-morph): discovers exported React components, extracts prop types, classifies by category (page/layout/component/primitive)
- **Framework detection**: auto-detects Next.js, Vite, CRA, Remix projects and their entry points
- **Provider chain detection**: identifies provider wrappers from app entry points
- **Split-ownership code generation**: scanner-owned files are safe to regenerate; user-owned files (`views.custom.tsx`, `providers.tsx`) are never overwritten
- **Cross-project scanning**: `--source` and `--output` flags enable scanning one project and outputting into another (monorepo support)
- **PreviewProvider runtime**: MSW lifecycle management for both browser (`setupWorker`) and Node (`setupServer`) environments
- **YAML fixture syntax**: declarative HTTP mock definitions convertible to MSW handlers

## Implementation Details

### New package: `packages/preview/`

Two entry points:
- `@scenar/preview` — Scanner, generator, config (`defineConfig()`)
- `@scenar/preview/runtime` — `PreviewProvider`, MSW bridge, YAML fixture converter

### CLI commands added to `@scenar/cli`

- `scenar preview init --source <path> --output <path>` — Full project scan + `.scenar/` scaffold
- `scenar preview sync --source <path> --output <path>` — Re-scan preserving user files

### Generated `.scenar/` directory (split ownership)

| File | Owner | Re-scan behavior |
|------|-------|-----------------|
| `views.generated.ts` | Scanner | Always overwritten |
| `views.custom.tsx` | User | Never touched |
| `views.ts` | Scanner | Always overwritten (merge barrel) |
| `providers.tsx` | User | Never touched |
| `preview.tsx` | Scanner | Always overwritten |
| `report.md` | Scanner | Always overwritten |
| `scenar.config.ts` | User | Never touched |

### Validation against Stigmer

- Scanned `client-apps/web`: discovered **118 components**, skipped 5
- Scanned `@stigmer/react` SDK: discovered **106 components**, skipped 4
- Cross-project import paths correctly computed (`../../client-apps/web/src/...`)
- Replaced 1,100+ lines of hand-written `preview-configs.ts` in Stigmer's docs site

## Benefits

- **Zero-config component discovery**: one command finds all React components in a project
- **Protocol-agnostic mocking**: MSW works with REST, GraphQL, gRPC-web, Connect, tRPC — any HTTP-based protocol
- **Monorepo support**: scan app A, output into project B with correct relative imports
- **Safe re-scanning**: user customizations are never lost
- **Diagnostic reporting**: `report.md` tells users exactly what was found and what was skipped

## Impact

- **Scenar users**: dramatically lower barrier to creating demos from existing React apps
- **Stigmer**: deleted 1,465 lines of hand-written preview infrastructure, replaced with auto-generated views
- **README**: updated to position preview generation as the natural next step after manual Quick Start

## Related Work

- Stigmer's `@stigmer/react/demo` module (`DemoTransport`, `createDemoClient`, `fixtures`, `samples`) remains in use by existing scenarios but is architecturally superseded by `PreviewProvider` + MSW for new work
- Future work: migrate existing Stigmer scenarios from `createDemoClient` to `PreviewProvider`, potentially extract `samples` into a standalone test data package

---

**Status**: Production Ready
**Published**: `@scenar/preview@0.1.2` on npm
**Timeline**: Single session
