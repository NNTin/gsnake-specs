# Core Flows: Rust Core & CLI Integration

### 1. Web: WASM-Powered Gameplay

This flow describes the user journey in the browser as the game transitions to the Rust-backed engine.

**Trigger:** User navigates to the web application.

**Steps:**

1. **Initialization:** The browser background-loads the WASM module and the `levels.json` data.
1. **Level Injection:** The host (TypeScript) injects the current level data into the Rust core.
1. **Interactive Loop:**
   - User presses an arrow key or WASD.
   - TypeScript passes the input to the WASM bridge.
   - Rust core processes the turn (move + gravity) and emits a new `Frame`.
   - The Svelte UI renders the 2D grid based on the emitted `Frame`.
1. **Result States:**
   - **Level Complete:** UI displays the completion modal; user clicks "Next" to trigger a new injection.
   - **Game Over:** UI displays the game over modal; user clicks "Restart" to re-inject the current level.

### 2. CLI: Native Terminal Gameplay

This flow describes the native experience for a user playing gSnake directly in their terminal.

**Trigger:** User runs the `gsnake` binary in their terminal.

**Steps:**

1. **Start-up:** The CLI binary starts automatically at Level 1, rendering a fixed-size grid and status bar.
1. **Gameplay:**
   - User uses **WASD** keys to move the snake.
   - The terminal updates the grid view instantly after every move and gravity fall.
   - User can press **R** at any time to reset the current level.
1. **Overlay Feedback:**
   - **Level Complete / Game Over:** A text-based overlay appears within the terminal grid boundaries (e.g., "GAME OVER - Press R to Restart or Q to Quit").
1. **Exit:** User presses **Q** to terminate the process and return to the shell.

### 3. Shared: Level Progression

This flow describes how users move through the game's levels across both platforms.

**Steps:**

1. **Completion:** Upon reaching the exit, the engine emits a `LevelComplete` status.
1. **Auto-Advance:** In the CLI, the next level loads automatically after a short delay or keypress. In Web, the user interacts with a modal to proceed.
1. **Data Consistency:** Both platforms utilize the same `levels.json` structure, ensuring the puzzles and mechanics remain identical regardless of the interface.

### 4. Exceptional Flows: Error Handling & Constraints

This flow describes how the system reacts to environmental failures.

**Flow: Terminal Too Small (CLI)**

1. **Trigger:** User runs `gsnake` in a small terminal window.
1. **Detection:** CLI checks terminal dimensions against Level 1 grid size.
1. **Feedback:** CLI displays a "Terminal too small" warning and waits for resize or exit.

**Flow: Environmental Error (Shared)**

1. **Trigger:** Missing `levels.json` or malformed data injection.
1. **Action:** The engine/host throws an error.
1. **Feedback:** The UI/CLI displays the error message and a traceback for debugging, then terminates or halts.
