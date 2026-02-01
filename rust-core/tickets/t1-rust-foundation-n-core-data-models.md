## Objective

Establish the Rust project foundation and define the core data structures that will serve as the single source of truth for both the Web (WASM) and CLI versions of gSnake.

## Scope

**Included:**

- Initialize a Rust workspace with three crates: `gsnake-core` (library), `gsnake-wasm` (cdylib), and `gsnake-cli` (binary)
- Define core enums and structs in `gsnake-core` (`Position`, `CellType`, `Direction`, `GameStatus`, `GameState`, `Frame`)
- Implement `serde` serialization/deserialization for all core models
- Set up project-wide linting and formatting rules (clippy, rustfmt)

**Excluded:**

- Game logic implementation
- WASM bindings
- CLI TUI code

## Spec References

- spec:epics/rust-core/tech-plan.md - Tech Plan: Data Model
- spec:epics/rust-core/tech-plan.md - Tech Plan: Architectural Approach

## Key Deliverables

1. **Workspace Configuration:** `Cargo.toml` at the root managing the workspace.
1. **Core Models:** `gsnake-core/src/lib.rs` containing all shared data structures.
1. **Serialization:** `Cargo.toml` in `gsnake-core` with `serde` features enabled.

## Acceptance Criteria

- ✅ `cargo check` passes for the entire workspace
- ✅ Core models successfully serialize/deserialize to/from JSON
- ✅ Workspace structure allows `gsnake-wasm` and `gsnake-cli` to depend on `gsnake-core`

## Dependencies

None - this is the foundation ticket
