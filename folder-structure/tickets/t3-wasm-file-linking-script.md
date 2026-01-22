## Objective

Standardize WASM development linking by using a `file:` dependency to the local WASM package and a root script to rebuild WASM before dev/test.

## Scope

**Included:**

- Update `gsnake-web/package.json` to use `file:../gsnake-core/engine/bindings/wasm/pkg` for `gsnake-wasm`
- Add a root script that rebuilds the WASM package before dev/test
- Update any documentation references in the repo to remove `npm link` in favor of `file:`

**Excluded:**

- Publishing to npm
- CI workflow updates

## Spec References

- spec:epics/folder-structure/epic-brief.md - Requirements, "Development Linking"
- spec:epics/folder-structure/tech-plan.md - Architectural Approach, gsnake-web integration
- spec:epics/folder-structure/core-flows.md - Flow 2 (Web Frontend Consumption)

## Implementation Details

Replace `npm link` guidance with a `file:` dependency and provide a root script that runs `wasm-pack build` so local dev and CI workflows share the same rebuild step.

## Acceptance Criteria

- [ ] `gsnake-web/package.json` uses `file:../gsnake-core/engine/bindings/wasm/pkg` for `gsnake-wasm`
- [ ] Root script rebuilds WASM package before dev/test
- [ ] Documentation in the repo no longer instructs `npm link` for WASM
- [ ] `gsnake-web` installs and runs using the local WASM package via `file:`

## Dependencies

- Depends on: t1-restructure-repo-layout
