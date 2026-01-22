## Objective

Create the visual interface for the native gSnake CLI using the `ratatui` library.

## Scope

**Included:**

- Initialize `gsnake-cli` crate and configure `ratatui` and `crossterm`
- Implement a 2D grid renderer that maps `CellType` to terminal characters/colors
- Create a status bar showing current level, moves, and food collected
- Implement text-based overlays for "Game Over" and "Level Complete" messages
- Implement a terminal size validation check to ensure the grid fits in the window
- Ensure the terminal UI maintains a fixed grid size as per requirements

**Excluded:**

- Real-time input handling (WASD)
- Game loop integration

## Spec References

- spec:epics/rust-core/tech-plan.md - Tech Plan: Component Architecture (CLI)
- spec:epics/rust-core/core-flows.md - Flow 2: Native Terminal Gameplay

## Key Deliverables

1.  **TUI Modules:** `gsnake-cli/src/ui.rs` or equivalent for rendering logic.
2.  **Visual Components:** Grid, status bar, and overlay widgets.

## Acceptance Criteria

- ✅ CLI displays a recognizable gSnake grid and status bar
- ✅ Overlays render correctly within the terminal bounds
- ✅ UI correctly represents the `Frame` emitted by the core engine
- ✅ CLI displays a "Terminal too small" warning when the window is insufficient

## Dependencies

- T1: Rust Foundation & Core Data Models
