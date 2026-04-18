# npm Publish Infrastructure and CI Pipeline

**Date**: April 18, 2026

## Summary

Set up the complete npm publishing infrastructure and CI pipeline for all four Scenar packages (`@scenar/core`, `@scenar/sdk`, `@scenar/react`, `scenar` CLI). This enables automated releases via GitHub Actions triggered by version tags, matching the Stigmer release pattern.

## Problem Statement

The Scenar monorepo had four publishable packages ready for consumption, but no mechanism to actually publish them to npm or validate PRs with CI.

### Pain Points

- Packages could not be installed from npm — only usable via local workspace references
- No CI to catch regressions on pull requests
- No release automation — publishing would require manual steps
- Missing LICENSE file despite Apache-2.0 declarations in package.json
- Package metadata incomplete (no `repository`, `publishConfig`, or `keywords`)

## Solution

Adapted Stigmer's proven tag-based release pattern for the Scenar pnpm monorepo:
- A publish script that builds all packages, pins workspace dependencies, and publishes from `dist/` in dependency-graph order
- Two GitHub Actions workflows: CI (build/test/typecheck on PRs) and Release (publish to npm on `v*` tags)
- Complete package metadata for npm discoverability

## Implementation Details

### npm Organization

- Created `@scenar` npm org on npmjs.com (owner: `whysosuresh`)
- All four package names confirmed available and unclaimed

### Package Metadata

Added to all four `package.json` files:
- `repository` field with per-package `directory` path
- `publishConfig: { "access": "public" }` for scoped packages
- `keywords` for npm search discoverability

### Publish Script (`scripts/publish-libs.mjs`)

Adapted from Stigmer's `scripts/publish-libs.mjs` with these differences:
- Uses `pnpm -r build` / `pnpm -r clean` instead of npm
- Package list: `packages/core` → `packages/sdk` → `packages/react` → `packages/cli`
- Workspace dep prefix: `@scenar/` instead of `@stigmer/`
- Handles `workspace:*` protocol pinning

Features:
- `--version` stamps version into every `dist/package.json`
- `--dry-run` for safe testing
- `--skip-build` for re-publishing pre-built artifacts
- Pre-release tag detection (`0.1.0-rc.1` → npm `next` tag)
- Temporary `.npmrc` creation from `NPM_TOKEN` env var

### CI Workflow (`.github/workflows/ci.yaml`)

- Triggers on PRs and pushes to `main`
- Steps: checkout → pnpm setup → Node 22 → install → build → test → typecheck

### Release Workflow (`.github/workflows/release.npm.yaml`)

- Triggers on `v*` tags or manual `workflow_dispatch` with version input
- Two-job pipeline: `determine-version` → `publish`
- Uses `NPM_TOKEN` secret for registry authentication

### Other Files

- `LICENSE`: Apache-2.0 full text (copyright: Scenar Contributors)
- `.gitignore`: Added `.npmrc` entry (safety net for temporary CI auth file)

## Benefits

- **Automated releases**: `git tag v0.1.0 && git push --tags` publishes all four packages
- **CI validation**: Every PR gets build + test + typecheck
- **npm discoverability**: Proper metadata, keywords, and repository links
- **Supply chain**: Lockstep versioning pins workspace deps to exact versions at publish time
- **Dry-run support**: Test the full publish flow without touching the registry

## Impact

- Unblocks Stigmer's T06 (Rewire Demos to Scenar Imports) — Stigmer can now `npm install @scenar/*` once the first release is tagged
- Any external consumer can install Scenar packages from npm
- Contributors get CI feedback on pull requests

## Related Work

- Stigmer `release.npm-libs.yaml` workflow — pattern source
- Stigmer `scripts/publish-libs.mjs` — script adapted from this
- Sub-project `20260417.03.sp.proto-simplify-and-cli` — packages being published

---

**Status**: ✅ Production Ready
**Timeline**: Single session
