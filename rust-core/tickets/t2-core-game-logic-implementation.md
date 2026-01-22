## Objective

Port the game's movement, gravity, and collision logic from TypeScript to the Rust `gsnake-core` library, ensuring 100% logic parity.

## Scope

**Included:**

- Implement `GameEngine` struct in `gsnake-core`
- Port `process_move` logic (head calculation, eating food, tail popping)
- Port `apply_gravity` logic (iterative falling and collision checks)
- Implement `Frame` generation (computing the 2D grid from current state)
- Comprehensive unit tests for all game rules and edge cases

**Excluded:**

- External event listeners/callbacks (handled in T3)
- Real-time input handling

## Spec References

- spec:epics/rust-core/tech-plan.md - Tech Plan: Architectural Approach
- spec:epics/rust-core/core-flows.md - All Shared Flows

## Key Deliverables

1.  **Engine Logic:** `gsnake-core/src/engine.rs` containing the core game loop.
2.  **Test Suite:** Unit tests verifying gravity behavior, boundary collisions, and win/loss conditions.

## Acceptance Criteria

- ✅ All unit tests in `gsnake-core` pass
- ✅ Gravity logic behaves identically to the original TypeScript implementation
- ✅ Engine correctly generates a `Frame` representing the current game state

## Dependencies

- T1: Rust Foundation & Core Data Models
