## Objective

Restructure the repository into a submodule-ready layout by moving the Rust workspace and bindings under `gsnake-core/`, and relocating shared data into the core engine.

## Scope

**Included:**

- Move Rust workspace root into `gsnake-core/`
- Relocate engine crates under `gsnake-core/engine/` (`core/`, `bindings/wasm`, `bindings/cli`, `bindings/py`)
- Move `data/levels.json` into `gsnake-core/engine/core/data/`
- Remove root-level `gsnake-wasm/` and `gsnake-cli/` directories
- Update paths/references that break due to the move

**Excluded:**

- Submodule conversion (separate ticket)
- CI workflow updates (separate ticket)
- WASM linking or scripts (separate tickets)

## Spec References

- spec:epics/folder-structure/tech-plan.md - Architectural Approach, Section "gsnake-core (Rust Workspace)"
- spec:epics/folder-structure/tech-plan.md - Folder and File Layout (Target)

## Implementation Details

Relocate the Rust workspace and bindings under `gsnake-core/` to match the target layout in the tech plan. Remove the temporary root-level crates so the canonical location is `gsnake-core/engine/bindings/*`.

## Acceptance Criteria

- [ ] Rust workspace root lives at `gsnake-core/Cargo.toml`
- [ ] `gsnake-core/engine/core/` contains the core engine crate
- [ ] `gsnake-core/engine/bindings/wasm`, `gsnake-core/engine/bindings/cli`, and `gsnake-core/engine/bindings/py` exist and build relative to the workspace
- [ ] `gsnake-core/engine/core/data/levels.json` is the only levels file
- [ ] Root-level `gsnake-wasm/` and `gsnake-cli/` directories are removed
- [ ] Build or import paths updated to reflect new locations

## Dependencies

None.
