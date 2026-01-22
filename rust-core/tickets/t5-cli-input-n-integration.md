## Objective

Finalize the native gSnake experience by implementing input handling, level loading, and the main interactive game loop.

## Scope

**Included:**

- Implement real-time keyboard listener using `crossterm` (WASD, R, Q)
- Implement level loading from `data/levels.json` using `serde_json`
- Wire the input handler to the `gsnake-core` engine
- Implement the automatic level progression logic
- Implement error handling with tracebacks for environmental failures (missing levels, etc.)
- Finalize the `main` loop in `gsnake-cli`

**Excluded:**

- Web frontend changes

## Spec References

- spec:epics/rust-core/core-flows.md - Flow 2: Native Terminal Gameplay
- spec:epics/rust-core/tech-plan.md - Tech Plan: Architectural Approach

## Key Deliverables

1.  **Main Loop:** `gsnake-cli/src/main.rs` coordinating input, logic, and rendering.
2.  **Data Loader:** Level loading logic for the native environment.

## Acceptance Criteria

- ✅ Game is fully playable in the terminal with WASD controls
- ✅ Pressing 'R' restarts the level and 'Q' exits the process
- ✅ Levels transition automatically or via keypress upon completion
- ✅ CLI successfully reads levels from the project's JSON data
- ✅ System provides clear error tracebacks upon initialization or data failures

## Dependencies

- T2: Core Game Logic Implementation
- T4: CLI Renderer & TUI Layout
