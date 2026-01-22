## Objective

Ensure `gsnake-core` can build and test in isolation by duplicating Rust toolchain and lint configs inside the submodule boundary.

## Scope

**Included:**

- Add `rust-toolchain.toml` under `gsnake-core/`
- Duplicate any root Rust lint/config files required for standalone builds
- Verify Rust commands run successfully from inside `gsnake-core/`

**Excluded:**

- Toolchain upgrades or lint rule changes
- CI workflow changes

## Spec References

- spec:epics/folder-structure/tech-plan.md - Architectural Approach, gsnake-core constraint
- spec:epics/folder-structure/tech-plan.md - Folder and File Layout (Target)
- spec:epics/folder-structure/epic-brief.md - Success Criteria

## Implementation Details

Mirror root Rust configuration files inside `gsnake-core/` so the submodule can be built and tested independently of the parent repository.

## Acceptance Criteria

- [ ] `gsnake-core/rust-toolchain.toml` exists and matches the root toolchain
- [ ] Any required lint configs are present under `gsnake-core/`
- [ ] Running `cargo build` from `gsnake-core/` succeeds
- [ ] Running `cargo test` from `gsnake-core/` succeeds

## Dependencies

- Depends on: t1-restructure-repo-layout
