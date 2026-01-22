## Objective

Expose a uniform levels API through bindings so both core and WASM consumers access level data through the same interface.

## Scope

**Included:**

- Add or update bindings to expose level data via a shared API in core and WASM
- Embed `levels.json` into the WASM package during build
- Update any consumers that read `levels.json` directly to use the bindings API

**Excluded:**

- Changes to level content or game rules
- New UI features unrelated to level loading

## Spec References

- spec:epics/folder-structure/tech-plan.md - Data Model, "Shared State (WASM/Core)"
- spec:epics/folder-structure/epic-brief.md - Requirements, "Shared Level Access"

## Implementation Details

Define a single levels API exposed by the engine bindings and ensure the WASM package embeds `levels.json` so web clients access levels through the same interface as other platforms.

## Acceptance Criteria

- [ ] Core bindings expose level data via a dedicated API
- [ ] WASM bindings expose level data via the same API surface
- [ ] WASM build embeds `levels.json` in the package output
- [ ] Web UI reads levels through WASM bindings only
- [ ] No direct reads of `levels.json` remain in UI code

## Dependencies

- Depends on: t1-restructure-repo-layout
