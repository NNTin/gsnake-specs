# Technical Plan: Rust Core & CLI Integration

### Architectural Approach

1. **Core Logic Library:** A pure Rust crate (`gsnake-core`) will implement all game mechanics (gravity, collisions, movement). This ensures 100% logic parity between the web and CLI versions.
1. **Single Source of Truth:** The TypeScript `GameEngine` will be replaced by a WebAssembly (WASM) module compiled from the Rust core. The web frontend will become a "dumb" rendering layer.
1. **Push-Based Communication:** The engine will use a callback-driven architecture. The host environment (Svelte or CLI) registers a listener that the Rust core invokes whenever a state change occurs, passing a serialized `Frame`.
1. **WASM & Native Binaries:** `wasm-bindgen` will be used for the web target, while a native CLI target will use `crossterm` for raw terminal input and `ratatui` for TUI rendering.
1. **Terminal Constraints:** The CLI will perform a window size check upon initialization. If the terminal is smaller than the active level's grid dimensions, it will display a "Terminal too small" warning.
1. **Error Handling:** Failures (e.g., malformed JSON, initialization errors) will be handled via Rust's `Result` type. Errors will be bubbled up to the host environment with full traceback information for debugging.
1. **Host-Injected Data:** Level data will be loaded by the host (from `levels.json`) and injected into the Rust core as serialized JSON, allowing the core to remain filesystem-agnostic.

### Data Model

```rust
// Core state and configuration models
pub struct Position { pub x: i32, pub y: i32 }

pub enum CellType { Empty, SnakeHead, SnakeBody, Food, Obstacle, Exit }
pub enum GameStatus { Playing, GameOver, LevelComplete, AllComplete }
pub enum Direction { North, South, East, West }

pub struct GameState {
    pub status: GameStatus,
    pub current_level: u32,
    pub moves: u32,
    pub food_collected: u32,
    pub total_food: u32,
}

pub struct Snake {
    pub segments: Vec<Position>,
    pub direction: Option<Direction>,
}

// The "Frame" emitted to the host for rendering
pub struct Frame {
    pub grid: Vec<Vec<CellType>>,
    pub state: GameState,
}
```

### Component Architecture

1. **gsnake-core (Rust Library):**
   - **State Manager:** Maintains the internal snake segments, current level geometry, and game counters.
   - **Physics Engine:** Implements the turn-based gravity and collision detection logic.
   - **Event Dispatcher:** Manages host callbacks and triggers updates after every valid input.
1. **gsnake-wasm (Web Bridge):**
   - Provides JS-friendly bindings for the core library.
   - Serializes Rust structures into JS objects using `serde-wasm-bindgen`.
1. **gsnake-cli (Native App):**
   - **Input Handler:** Uses `crossterm` to capture real-time keyboard events and map them to `Direction` commands.
   - **Renderer:** Uses `ratatui` to draw the 2D grid and UI status bar in the terminal based on the `Frame` emitted by the core.
1. **Frontend (Svelte/TypeScript):**
   - Responsible for initial WASM loading and level data retrieval.
   - Subscribes to the WASM bridge to receive `Frame` updates and updates Svelte stores for visual reactivity.
