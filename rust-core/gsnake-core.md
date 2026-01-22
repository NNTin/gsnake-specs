# System Spec: gsnake-core

**Version:** 1.0
**Status:** Implemented
**Last Updated:** 2026-01-18

---

## Table of Contents
1. [Background](#1-background)
2. [Map Architecture](#2-map-architecture)
3. [Define Interfaces](#3-define-interfaces)
4. [Define State](#4-define-state)
5. [Trace Flows](#5-trace-flows)
6. [Observe System](#6-observe-system)
7. [Define Integration](#7-define-integration)
8. [Enforce Constraints](#8-enforce-constraints)
9. [Build and Development](#9-build-and-development)
10. [Performance Characteristics](#10-performance-characteristics)
11. [References](#11-references)

---

## 1. Background

### 1.1 Problem Statement

gSnake is a gravity-based puzzle game where the player navigates a snake through increasingly complex levels. The original implementation coupled game logic to the TypeScript/Svelte web environment, creating several limitations:

- **Platform Lock-in:** Game could only run in browsers, preventing native terminal gameplay or AI agent integration
- **Inconsistent Behavior:** No guarantees of identical game mechanics across potential future implementations
- **Browser Dependency:** Automated agents (e.g., LLMs) would require full browser environments for simple command-driven gameplay
- **Maintenance Burden:** Logic changes would need to be replicated across multiple platform-specific implementations

### 1.2 Solution Overview

**gsnake-core** is a cross-platform game engine written in Rust that serves as the single source of truth for all game logic. By compiling to both WebAssembly (WASM) and native binaries, it provides:

- **100% Logic Parity:** Identical game behavior across web, CLI, and future platforms
- **Platform Decoupling:** Pure game logic separated from rendering and I/O concerns
- **First-Class CLI Experience:** Native terminal gameplay without browser overhead
- **Extensibility:** Clear architecture for future command-driven interfaces (AI agents, APIs, etc.)

### 1.3 Key Principles

1. **Separation of Concerns:** Core engine is filesystem-agnostic and I/O-free; hosts inject data and handle rendering
2. **Push-Based Architecture:** Engine emits `Frame` objects to registered callbacks on state changes
3. **Type Safety:** Strong typing via Rust with serialization contracts for cross-language boundaries
4. **Single Source of Truth:** One codebase, multiple compilation targets (WASM, native binary)

---

## 2. Map Architecture

### 2.1 Workspace Structure

The gsnake-core project uses a Cargo workspace with three primary crates:

```
gsnake-core/
├── Cargo.toml                    # Workspace definition
├── engine/
│   ├── core/                     # Pure game logic library
│   │   ├── src/
│   │   │   ├── lib.rs           # Public API exports
│   │   │   ├── models.rs        # Data structures & types
│   │   │   ├── engine.rs        # GameEngine implementation
│   │   │   └── levels.rs        # Level parsing utilities
│   │   ├── data/
│   │   │   └── levels.json      # Level definitions
│   │   └── tests/
│   │       └── contract_tests.rs # Serialization contracts
│   │
│   └── bindings/
│       ├── wasm/                # WebAssembly bridge
│       │   └── src/lib.rs       # WASM bindings via wasm-bindgen
│       │
│       └── cli/                 # Terminal interface
│           └── src/
│               ├── main.rs      # Entry point & game loop
│               ├── ui.rs        # Ratatui rendering
│               ├── input.rs     # Keyboard input handling
│               └── levels.rs    # Level file loading
```

**Location:** `/home/nntin/git/gSnake/gsnake-core/`

### 2.2 Module Boundaries

#### Core Library (`gsnake-core`)
- **Responsibility:** Pure game logic, state management, physics simulation
- **Dependencies:** `serde`, `serde_json`, `ts-rs` (for TypeScript type generation)
- **Exports:** Data models, `GameEngine`, level parsing utilities
- **Constraints:** No I/O, no filesystem access, no rendering

**Source:** `/home/nntin/git/gSnake/gsnake-core/engine/core/`

#### WASM Binding (`gsnake-wasm`)
- **Responsibility:** JavaScript FFI bridge for web environments
- **Dependencies:** `wasm-bindgen`, `serde-wasm-bindgen`, `js-sys`, `web-sys`
- **Exports:** `WasmGameEngine` wrapper, `getLevels()` helper
- **Constraints:** Web-specific error handling, callback marshalling

**Source:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/src/lib.rs`

#### CLI Application (`gsnake-cli`)
- **Responsibility:** Native terminal gameplay interface
- **Dependencies:** `crossterm`, `ratatui`, `anyhow`
- **Capabilities:** Real-time keyboard input, TUI rendering, level file loading
- **Constraints:** Requires terminal dimensions ≥ grid size + UI overhead

**Source:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/`

### 2.3 Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         HOST ENVIRONMENT                         │
│  ┌─────────────────────┐              ┌─────────────────────┐  │
│  │   Web (TypeScript)  │              │   CLI (Terminal)    │  │
│  │                     │              │                     │  │
│  │  • Load levels.json │              │  • Load levels.json │  │
│  │  • Capture keyboard │              │  • Capture keyboard │  │
│  │  • Render SVG grid  │              │  • Render TUI grid  │  │
│  └──────────┬──────────┘              └──────────┬──────────┘  │
│             │                                    │              │
│             │ injectLevel(LevelDefinition)       │              │
│             │ processMove(Direction)             │              │
│             ▼                                    ▼              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              WASM Bridge / Native Engine                 │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │          gsnake-core (Rust)                        │  │  │
│  │  │                                                     │  │  │
│  │  │  ┌─────────────┐         ┌──────────────────┐     │  │  │
│  │  │  │ GameEngine  │────────▶│  LevelState      │     │  │  │
│  │  │  │             │         │  • Snake         │     │  │  │
│  │  │  │ • new()     │         │  • Obstacles     │     │  │  │
│  │  │  │ • process_  │         │  • Food          │     │  │  │
│  │  │  │   move()    │         │  • Exit          │     │  │  │
│  │  │  │ • generate_ │         └──────────────────┘     │  │  │
│  │  │  │   frame()   │                                  │  │  │
│  │  │  └─────────────┘                                  │  │  │
│  │  │         │                                          │  │  │
│  │  │         │ apply_gravity()                          │  │  │
│  │  │         │ check_collision()                        │  │  │
│  │  │         ▼                                          │  │  │
│  │  │  ┌─────────────┐                                  │  │  │
│  │  │  │   Frame     │ ───────── emit ────────┐         │  │  │
│  │  │  │             │                        │         │  │  │
│  │  │  │ • grid[][]  │                        │         │  │  │
│  │  │  │ • GameState │                        │         │  │  │
│  │  │  └─────────────┘                        │         │  │  │
│  │  └─────────────────────────────────────────┼─────────┘  │  │
│  └────────────────────────────────────────────┼────────────┘  │
│             │                                  │              │
│             │ onFrame(Frame)                   │              │
│             ▼                                  ▼              │
│  ┌─────────────────────┐              ┌─────────────────────┐│
│  │  UI Update (Svelte) │              │  TUI Render (Ratatui)││
│  └─────────────────────┘              └─────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 2.4 Contract Boundaries

All cross-language communication uses JSON serialization with strict contracts:

| Boundary | Input | Output | Contract File |
|----------|-------|--------|---------------|
| **Level Injection** | `LevelDefinition` (JSON) | `GameEngine` instance | `/home/nntin/git/gSnake/gsnake-core/engine/core/src/models.rs` (lines 122-159) |
| **Move Processing** | `Direction` enum | `Frame` struct | `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs` (lines 48-121) |
| **State Query** | None | `GameState` struct | `/home/nntin/git/gSnake/gsnake-core/engine/core/src/models.rs` (lines 61-83) |
| **Error Reporting** | N/A | `ContractError` | `/home/nntin/git/gSnake/gsnake-core/engine/core/src/models.rs` (lines 201-230) |

---

## 3. Define Interfaces

### 3.1 Core Engine API

The `GameEngine` struct provides the primary interface for game logic:

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs`

```rust
pub struct GameEngine {
    level_definition: LevelDefinition,
    level_state: LevelState,
    game_state: GameState,
    input_locked: bool,
}

impl GameEngine {
    // Constructor
    pub fn new(level: LevelDefinition) -> Self;

    // Primary gameplay method
    pub fn process_move(&mut self, direction: Direction) -> bool;

    // State generation for rendering
    pub fn generate_frame(&self) -> Frame;

    // Read-only accessors
    pub fn game_state(&self) -> &GameState;
    pub fn level_definition(&self) -> &LevelDefinition;
    pub fn level_state(&self) -> &LevelState;
}
```

**Key Methods:**

- **`new(level: LevelDefinition) -> Self`**
  Initializes a new game instance with the provided level configuration. Automatically derives runtime state from the immutable level definition.

  **Lines:** 15-28

- **`process_move(direction: Direction) -> bool`**
  Executes a single turn: validates input → moves snake → applies gravity → checks collisions → updates game status. Returns `false` if move is rejected (opposite direction, input locked, game not playing).

  **Lines:** 48-121
  **Critical Logic:** Lines 64-70 (opposite direction check), 96-112 (collision & exit detection)

- **`generate_frame() -> Frame`**
  Produces a snapshot of the current game state as a 2D grid of `CellType` enums plus metadata. Used by rendering layers to visualize the game.

  **Lines:** 124-166

### 3.2 WASM Bridge Interface

The WASM binding exposes a JavaScript-friendly API:

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/src/lib.rs`

```rust
#[wasm_bindgen]
pub struct WasmGameEngine {
    engine: GameEngine,
    on_frame_callback: Option<Function>,
}

#[wasm_bindgen]
impl WasmGameEngine {
    // Constructor from JS object
    #[wasm_bindgen(constructor)]
    pub fn new(level_json: JsValue) -> Result<WasmGameEngine, JsValue>;

    // Register callback for state changes
    #[wasm_bindgen(js_name = onFrame)]
    pub fn on_frame(&mut self, callback: Function);

    // Process move and emit frame to callback
    #[wasm_bindgen(js_name = processMove)]
    pub fn process_move(&mut self, direction: JsValue) -> Result<JsValue, JsValue>;

    // Query methods
    #[wasm_bindgen(js_name = getFrame)]
    pub fn get_frame(&self) -> Result<JsValue, JsValue>;

    #[wasm_bindgen(js_name = getGameState)]
    pub fn get_game_state(&self) -> Result<JsValue, JsValue>;

    #[wasm_bindgen(js_name = getLevel)]
    pub fn get_level(&self) -> Result<JsValue, JsValue>;
}

// Global helper function
#[wasm_bindgen(js_name = getLevels)]
pub fn get_levels() -> Result<JsValue, JsValue>;
```

**Error Handling:**
All methods return `Result<JsValue, JsValue>` where errors are serialized `ContractError` objects.

**Lines:** 21-128

### 3.3 Data Structures

#### 3.3.1 Position

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/models.rs` (lines 4-16)

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize, TS)]
pub struct Position {
    pub x: i32,
    pub y: i32,
}
```

**Contract:** Serializes as `{"x": i32, "y": i32}`

#### 3.3.2 CellType Enum

**Lines:** 18-27

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, TS)]
pub enum CellType {
    Empty,
    SnakeHead,
    SnakeBody,
    Food,
    Obstacle,
    Exit,
}
```

**Contract:** Serializes as string literals (`"Empty"`, `"SnakeHead"`, etc.)

#### 3.3.3 Direction Enum

**Lines:** 38-58

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, TS)]
pub enum Direction {
    North,
    South,
    East,
    West,
}

impl Direction {
    pub fn opposite(&self) -> Self;
}
```

**Contract:** Serializes as `"North"` | `"South"` | `"East"` | `"West"`

#### 3.3.4 GameState

**Lines:** 61-83

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
#[ts(rename_all = "camelCase")]
pub struct GameState {
    pub status: GameStatus,
    pub current_level: u32,
    pub moves: u32,
    pub food_collected: u32,
    pub total_food: u32,
}
```

**Contract:** Uses camelCase for JSON fields (`currentLevel`, `foodCollected`, `totalFood`)

#### 3.3.5 GameStatus Enum

**Lines:** 29-36

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, TS)]
pub enum GameStatus {
    Playing,
    GameOver,
    LevelComplete,
    AllComplete,
}
```

#### 3.3.6 Frame

**Lines:** 188-199

```rust
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize, TS)]
pub struct Frame {
    pub grid: Vec<Vec<CellType>>,
    pub state: GameState,
}
```

**Contract:**
```json
{
  "grid": [[CellType]], // 2D array indexed as grid[y][x]
  "state": GameState
}
```

#### 3.3.7 LevelDefinition

**Lines:** 122-159

```rust
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
#[ts(rename_all = "camelCase")]
pub struct LevelDefinition {
    pub id: u32,
    pub name: String,
    pub grid_size: GridSize,
    pub snake: Vec<Position>,
    pub obstacles: Vec<Position>,
    pub food: Vec<Position>,
    pub exit: Position,
    pub snake_direction: Direction,
}
```

**Contract:** Injected as JSON from host environments
**Example:** `/home/nntin/git/gSnake/gsnake-core/engine/core/data/levels.json`

#### 3.3.8 ContractError

**Lines:** 201-230

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
#[ts(rename_all = "camelCase")]
pub struct ContractError {
    pub kind: ContractErrorKind,
    pub message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub context: Option<BTreeMap<String, String>>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, TS)]
#[serde(rename_all = "camelCase")]
#[ts(rename_all = "camelCase")]
pub enum ContractErrorKind {
    InvalidInput,      // Malformed input data
    InputRejected,     // Valid but rejected by game logic
    SerializationFailed, // JSON serialization error
    InitializationFailed, // Engine initialization error
    InternalError,     // Unexpected internal error
}
```

**Usage:** Returned as `Err(JsValue)` in WASM methods, logged in CLI

---

## 4. Define State

### 4.1 State Machine

The game operates as a finite state machine with four primary states:

```
                    ┌─────────────┐
                    │   PLAYING   │◀──────┐
                    └──────┬──────┘       │
                           │              │
                    processMove(dir)      │
                           │              │
              ┌────────────┼────────────┐ │
              │            │            │ │
     collision?        all_food?    exit? │
              │            │            │ │
              ▼            ▼            │ │
        ┌──────────┐ ┌────────────┐   │ │
        │ GAMEOVER │ │ LEVEL      │   │ │
        │          │ │ COMPLETE   │   │ │
        └────┬─────┘ └─────┬──────┘   │ │
             │             │           │ │
        reset()       next_level()     │ │
             │             │           │ │
             └─────────────┴───────────┘ │
                                         │
                   if last_level():      │
                      ┌─────────────┐    │
                      │    ALL      │    │
                      │  COMPLETE   │    │
                      └──────┬──────┘    │
                             │           │
                        restart()        │
                             └───────────┘
```

**Implementation:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/models.rs` (lines 29-36)

### 4.2 Entity Structures

#### 4.2.1 Snake Entity

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/models.rs` (lines 86-105)

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize, TS)]
pub struct Snake {
    pub segments: Vec<Position>,  // Head at index 0
    pub direction: Option<Direction>,
}
```

**Invariants:**
- Head is always at `segments[0]`
- Segments are ordered from head to tail
- Direction is `Some` during gameplay, `None` on initialization

**Movement Logic:**
1. New head position calculated based on direction
2. New head inserted at `segments[0]`
3. If food eaten: tail remains (snake grows)
4. If no food: tail removed via `segments.pop()`

**Source:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs` (lines 76-94)

#### 4.2.2 LevelState

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/models.rs` (lines 162-185)

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct LevelState {
    pub grid_size: GridSize,
    pub snake: Snake,
    pub obstacles: Vec<Position>,
    pub food: Vec<Position>,
    pub exit: Position,
}
```

**Purpose:** Runtime mutable state derived from immutable `LevelDefinition`

**Mutations:**
- `food` vector shrinks as items are collected
- `snake.segments` grows/shrinks with gameplay
- All other fields remain constant during a level

### 4.3 Game Engine Internal State

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs` (lines 6-12)

```rust
pub struct GameEngine {
    level_definition: LevelDefinition,  // Immutable reference level
    level_state: LevelState,            // Mutable gameplay state
    game_state: GameState,              // Metadata (moves, status, etc.)
    input_locked: bool,                 // Prevents concurrent input
}
```

**State Transitions:**

| Field | Mutated By | Trigger |
|-------|-----------|---------|
| `level_state.snake` | `process_move()` | Valid directional input |
| `level_state.food` | `process_move()` | Snake head reaches food position |
| `game_state.moves` | `process_move()` | Every successful move |
| `game_state.food_collected` | `process_move()` | Food consumption |
| `game_state.status` | `process_move()` | Collision, exit reached |
| `input_locked` | `process_move()` | Set at start, cleared at end |

**Lines:** 62, 88, 98, 115, 109

### 4.4 Error States

Errors are categorized by `ContractErrorKind` and surfaced via `Result` types:

| Error Kind | Trigger | Example |
|------------|---------|---------|
| `InvalidInput` | Malformed JSON, invalid enum string | `Direction::from_str("Invalid")` |
| `InputRejected` | Game logic rejects move | Opposite direction, input locked |
| `SerializationFailed` | Serde serialization error | Circular references (impossible in current schema) |
| `InitializationFailed` | Cannot parse level data | Missing required field in `LevelDefinition` |
| `InternalError` | Unexpected panic or logic error | Callback invocation failure |

**WASM Error Handling:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/src/lib.rs` (lines 141-175)

**CLI Error Handling:** Uses `anyhow::Result` with context propagation
**Source:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/main.rs` (lines 22-27)

---

## 5. Trace Flows

### 5.1 Primary Flow: Player Move with Gravity

**Entry Point:** User presses directional key → Host calls `engine.process_move(direction)`

**Trace:**

```
1. process_move(Direction)                     [engine.rs:48]
   ├─ Input validation                         [engine.rs:52-54]
   │  └─ Reject if input_locked or !Playing
   │
   ├─ Lock input                                [engine.rs:62]
   │
   ├─ Opposite direction check                  [engine.rs:65-70]
   │  └─ Reject if direction.opposite() == current_direction
   │
   ├─ Update snake.direction                    [engine.rs:74]
   │
   ├─ Calculate new head position               [engine.rs:76-77]
   │  └─ get_new_head_position()                [engine.rs:270-277]
   │
   ├─ Check for food collision                  [engine.rs:80]
   │  └─ get_food_index()                       [engine.rs:262-267]
   │
   ├─ Insert new head at segments[0]            [engine.rs:83]
   │
   ├─ Handle food consumption or tail removal   [engine.rs:85-94]
   │  ├─ IF food_index.is_some():
   │  │  ├─ Remove food from level_state.food   [engine.rs:87]
   │  │  └─ Increment food_collected            [engine.rs:88]
   │  └─ ELSE: Pop tail                         [engine.rs:92]
   │
   ├─ Check immediate collision                 [engine.rs:97-99]
   │  └─ check_collision()                      [engine.rs:211-230]
   │     ├─ is_within_bounds()                  [engine.rs:233-238]
   │     ├─ is_obstacle()                       [engine.rs:240-246]
   │     └─ Self-collision check                [engine.rs:223-227]
   │
   ├─ Apply gravity                             [engine.rs:101]
   │  └─ apply_gravity()                        [engine.rs:169-182]
   │     └─ WHILE can_snake_fall():             [engine.rs:170]
   │        ├─ Move all segments down (y += 1)  [engine.rs:172-174]
   │        └─ Check collision after each fall  [engine.rs:177-180]
   │
   ├─ Check exit condition                      [engine.rs:104-111]
   │  └─ IF !GameOver AND head==exit AND food_collected==total_food:
   │     └─ Set status = LevelComplete
   │
   ├─ Increment move counter                    [engine.rs:115]
   │
   └─ Unlock input & return true                [engine.rs:118-120]
```

**Key Decision Points:**

- **Line 66:** Opposite direction detection (prevents 180° turns)
- **Line 85:** Food consumption vs. tail removal (growth mechanic)
- **Line 97:** Immediate collision after move
- **Line 170:** Continuous gravity fall loop
- **Line 106:** Exit validation with food requirement

### 5.2 Gravity Application Flow

**Purpose:** Simulate continuous downward movement until snake lands on a platform

**Trace:**

```
apply_gravity()                                [engine.rs:169]
├─ WHILE can_snake_fall():                     [engine.rs:170]
│  ├─ FOR each segment:                        [engine.rs:172]
│  │  └─ segment.y += 1                        [engine.rs:173]
│  │
│  ├─ Check collision(head)                    [engine.rs:177]
│  └─ IF collision: GameOver + break           [engine.rs:178-180]
│
└─ can_snake_fall()                            [engine.rs:185-208]
   ├─ FOR each segment:
   │  ├─ Calculate next_y = segment.y + 1      [engine.rs:187]
   │  ├─ IF next_y >= grid_height: RETURN false [engine.rs:190]
   │  ├─ IF is_obstacle(next_pos): RETURN false [engine.rs:197]
   │  └─ IF is_food(next_pos): RETURN false    [engine.rs:202]
   └─ RETURN true (can fall)
```

**Special Behavior:** Food acts as platforms (line 202)
**Collision Handling:** GameOver if collision occurs during fall (line 178)

### 5.3 Frame Generation Flow

**Purpose:** Convert internal game state into a renderable 2D grid

**Trace:**

```
generate_frame()                               [engine.rs:124]
├─ Initialize empty grid                       [engine.rs:126-130]
│  └─ grid = vec![vec![Empty; width]; height]
│
├─ Place obstacles                             [engine.rs:133-137]
│  └─ FOR each obstacle: grid[y][x] = Obstacle
│
├─ Place food                                  [engine.rs:140-144]
│  └─ FOR each food: grid[y][x] = Food
│
├─ Place exit                                  [engine.rs:147-150]
│  └─ grid[exit.y][exit.x] = Exit
│
├─ Place snake head                            [engine.rs:153-157]
│  └─ grid[head.y][head.x] = SnakeHead
│
├─ Place snake body                            [engine.rs:159-163]
│  └─ FOR segments[1..]: grid[y][x] = SnakeBody
│
└─ Return Frame { grid, game_state }           [engine.rs:165]
```

**Layer Order:** Obstacles → Food → Exit → Snake (head overwrites body at index 0)

### 5.4 Error Flow: Input Rejection

**Scenario:** User attempts opposite direction turn

**Trace:**

```
process_move(Direction::South)                 [engine.rs:48]
└─ IF current_direction == Direction::North:   [engine.rs:65-66]
   └─ is_opposite_direction() == true          [engine.rs:280-288]
      └─ Unlock input                          [engine.rs:67]
      └─ RETURN false                          [engine.rs:68]

WASM Layer:
processMove(direction: JsValue)                [wasm/lib.rs:55]
└─ IF !engine.process_move():                  [wasm/lib.rs:59]
   └─ RETURN Err(ContractError {               [wasm/lib.rs:61-64]
         kind: InputRejected,
         message: "Input rejected by engine"
      })
```

**User Impact:** Web UI ignores the input; CLI shows no state change

---

## 6. Observe System

### 6.1 Logging Strategy

**Philosophy:** Engine is logging-agnostic; observability is the host's responsibility

#### WASM Logging

**Browser Console:**
Panic messages are routed to `console.error` via `console_error_panic_hook`

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/src/lib.rs` (line 18)

```rust
#[wasm_bindgen(start)]
pub fn init_panic_hook() {
    console_error_panic_hook::set_once();
}
```

**Debug Helper:**
```rust
#[wasm_bindgen]
pub fn log(s: &str) {
    web_sys::console::log_1(&JsValue::from_str(s));
}
```

**Lines:** 131-134

#### CLI Logging

**Error Context:**
Uses `anyhow::Context` for error chain propagation

**Example:**
```rust
load_levels(&levels_path)
    .with_context(|| format!(
        "Failed to load levels. Path: {}",
        levels_path.display()
    ))?;
```

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/main.rs` (lines 23-27)

### 6.2 Error Reporting

#### Contract Errors

All WASM boundary errors are serialized as `ContractError` objects:

```json
{
  "kind": "invalidInput",
  "message": "Failed to deserialize Direction",
  "context": {
    "input": "InvalidString",
    "expected": "North|South|East|West"
  }
}
```

**Test Suite:** `/home/nntin/git/gSnake/gsnake-core/engine/core/tests/contract_tests.rs`

**Serialization Tests:**
- Enum string values (lines 14-103)
- CamelCase field names (lines 132-183)
- Error context serialization (lines 186-223)
- Round-trip integrity (lines 229-307)

#### Panic Handling

**Development:** Full stack traces in debug builds
**Production (WASM):** Panic messages logged to browser console with source mapping

### 6.3 Diagnostic Outputs

#### CLI Status Bar

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/ui.rs` (lines 63-80)

Displays:
- Current level number
- Move count
- Food collected / total food

**Rendered as:**
```
╔══════════════════════════════════════════╗
║  Level: 1 | Moves: 23 | Food: 3/5        ║
╚══════════════════════════════════════════╝
```

#### Frame Export (Debug)

The core library includes `ts-rs` for automatic TypeScript type generation:

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/bin/export_ts.rs`

Generates `.d.ts` files for TypeScript consumers to ensure type safety.

### 6.4 Testing & Validation

#### Unit Tests

**Core Logic Tests:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs` (lines 292-509)

Coverage:
- Basic movement and gravity (lines 320-337)
- Opposite direction blocking (lines 341-352)
- Food collection and growth (lines 354-368)
- Collision detection (lines 371-419)
- Gravity physics (lines 422-455)
- Exit and level completion (lines 458-487)

#### Contract Tests

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/tests/contract_tests.rs`

Ensures:
- Enum serialization consistency (lines 14-103)
- Struct field naming conventions (lines 110-183)
- Round-trip serialization integrity (lines 229-307)
- Fixture generation for integration tests (lines 314-351)

**Fixture Output:** `/home/nntin/git/gSnake/gsnake-core/engine/core/tests/fixtures/`

---

## 7. Define Integration

### 7.1 WASM ↔ Web Integration

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/src/lib.rs`

#### Initialization Flow

```
Web App Startup
├─ Load WASM module via import
├─ Call getLevels() to fetch embedded levels     [wasm/lib.rs:122-128]
├─ Instantiate WasmGameEngine(level_json)        [wasm/lib.rs:33-41]
└─ Register onFrame(callback)                    [wasm/lib.rs:46-49]
```

**TypeScript Usage:**
```typescript
import init, { WasmGameEngine, getLevels } from './wasm/gsnake_wasm.js';

await init();
const levels = getLevels();
const engine = new WasmGameEngine(levels[0]);

engine.onFrame((frame) => {
  // Update Svelte stores
  gameState.set(frame.state);
  gridData.set(frame.grid);
});

// Process input
engine.processMove({ North });
```

#### Data Serialization

**Direction:** Maps to JS strings
```rust
pub enum Direction { North, South, East, West }
```
↓
```javascript
"North" | "South" | "East" | "West"
```

**Frame:** Maps to JS object
```rust
pub struct Frame {
    pub grid: Vec<Vec<CellType>>,
    pub state: GameState,
}
```
↓
```javascript
{
  grid: CellType[][],  // 2D array
  state: {
    status: "Playing" | "GameOver" | "LevelComplete" | "AllComplete",
    currentLevel: number,
    moves: number,
    foodCollected: number,
    totalFood: number
  }
}
```

**Serialization Helper:** `serde-wasm-bindgen` (lines 6, 34, 56, 93)

#### Error Boundary

WASM methods return `Result<JsValue, JsValue>`:

```rust
pub fn process_move(&mut self, direction: JsValue) -> Result<JsValue, JsValue> {
    let direction: Direction = from_value(direction)
        .map_err(|e| contract_error(ContractErrorKind::InvalidInput, &e.to_string()))?;

    let processed = self.engine.process_move(direction);
    if !processed {
        return Err(contract_error(
            ContractErrorKind::InputRejected,
            "Input rejected by engine",
        ));
    }

    // ... emit frame and return
}
```

**Lines:** 54-80

**Error Object:**
```javascript
{
  kind: "inputRejected",
  message: "Input rejected by engine",
  context?: Record<string, string>
}
```

### 7.2 Rust ↔ CLI Integration

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/main.rs`

#### Game Loop Flow

```
CLI Startup
├─ Load levels.json from filesystem              [cli/main.rs:23-27]
│  └─ levels::load_levels(&path)                 [cli/levels.rs]
│
├─ Initialize terminal (crossterm)               [cli/main.rs:34-38]
│
├─ Create GameEngine with level[0]               [cli/main.rs:57]
│
└─ Main Loop:                                    [cli/main.rs:60-140]
   ├─ Render frame via ratatui                   [cli/main.rs:67-80]
   │  └─ UI::render(frame)                       [cli/ui.rs:33-60]
   │
   ├─ Poll input (100ms timeout)                 [cli/main.rs:83]
   │  └─ InputHandler::poll_action()             [cli/input.rs]
   │
   └─ Match action:                              [cli/main.rs:84-136]
      ├─ Quit: break loop
      ├─ Reset: engine = GameEngine::new(level)
      ├─ Continue: load next level or restart
      └─ Move(dir): engine.process_move(dir)
```

#### Input Mapping

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/input.rs`

| Keyboard | Action | Engine Call |
|----------|--------|-------------|
| W, ↑ | MoveNorth | `process_move(Direction::North)` |
| S, ↓ | MoveSouth | `process_move(Direction::South)` |
| A, ← | MoveWest | `process_move(Direction::West)` |
| D, → | MoveEast | `process_move(Direction::East)` |
| R | Reset | `GameEngine::new(current_level)` |
| Space | Continue | Load next level |
| Q, Esc | Quit | Exit process |

#### Rendering Pipeline

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/ui.rs`

```
Frame → UI::render()                            [ui.rs:33]
├─ Create layout (status, grid, footer)         [ui.rs:34-41]
│
├─ Render status bar                            [ui.rs:44]
│  └─ "Level: X | Moves: Y | Food: Z/W"         [ui.rs:64-67]
│
├─ Render grid                                  [ui.rs:47]
│  ├─ Convert CellType → (char, color)          [ui.rs:268-277]
│  └─ Build ratatui Lines                       [ui.rs:94-101]
│
├─ Render footer help text                      [ui.rs:50]
│
└─ Render overlays (if GameOver/Complete)       [ui.rs:53-59]
   ├─ GameOver → Red overlay                    [ui.rs:125-167]
   └─ LevelComplete → Green overlay             [ui.rs:170-219]
```

**Cell Rendering:**
```rust
CellType::SnakeHead  → "●●" (Green)
CellType::SnakeBody  → "██" (Light Green)
CellType::Food       → "◆◆" (Red)
CellType::Obstacle   → "▓▓" (Gray)
CellType::Exit       → "⚑⚑" (Yellow)
CellType::Empty      → "  " (Black)
```

**Lines:** 20-26

#### Terminal Size Validation

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/ui.rs` (lines 302-314)

```rust
pub fn validate_terminal_size(
    terminal_width: u16,
    terminal_height: u16,
    grid_width: usize,
    grid_height: usize,
) -> Result<(), String> {
    let required_width = (grid_width as u16 * 2) + 4;  // +4 for borders
    let required_height = grid_height as u16 + 6;      // +6 for UI chrome

    if terminal_width < required_width || terminal_height < required_height {
        Err(format!("Terminal too small. Required: {}x{}, Current: {}x{}",
                    required_width, required_height, terminal_width, terminal_height))
    } else {
        Ok(())
    }
}
```

**Integration:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/main.rs` (lines 68-79)

### 7.3 Level Data Injection

#### WASM Approach

Levels are embedded in the WASM binary at compile time:

```rust
const LEVELS_JSON: &str = include_str!("../../../core/data/levels.json");

#[wasm_bindgen(js_name = getLevels)]
pub fn get_levels() -> Result<JsValue, JsValue> {
    let levels = parse_levels_json(LEVELS_JSON)
        .map_err(|e| contract_error(ContractErrorKind::InitializationFailed, &e.to_string()))?;
    to_value(&levels)
        .map_err(|e| contract_error(ContractErrorKind::SerializationFailed, &e.to_string()))
}
```

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/src/lib.rs` (lines 13, 122-128)

#### CLI Approach

Levels are loaded from the filesystem at runtime:

```rust
pub fn load_levels(path: &Path) -> Result<Vec<LevelDefinition>, serde_json::Error> {
    let data = fs::read_to_string(path)?;
    gsnake_core::levels::parse_levels_json(&data)
}

pub fn levels_path() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .join("../../core/data/levels.json")
}
```

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/levels.rs`

**Error Handling:** Contextual error if file not found (main.rs lines 23-27)

### 7.4 Compilation Targets

#### WASM Target

**Build Command:**
```bash
cd gsnake-core/engine/bindings/wasm
wasm-pack build --target web --release
```

**Output:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/pkg/`
- `gsnake_wasm.js` - JavaScript glue code
- `gsnake_wasm_bg.wasm` - WebAssembly binary
- `gsnake_wasm.d.ts` - TypeScript definitions

**Optimizations:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/Cargo.toml` (lines 27-30)
```toml
[profile.release]
opt-level = "z"      # Optimize for size
lto = true           # Link-time optimization
codegen-units = 1    # Single codegen unit for better optimization
```

#### Native CLI Target

**Build Command:**
```bash
cd gsnake-core/engine/bindings/cli
cargo build --release
```

**Output:** `/home/nntin/git/gSnake/gsnake-core/target/release/gsnake-cli`

**Run:**
```bash
./target/release/gsnake-cli
```

**Requirements:**
- Must be run from workspace root (for levels.json access)
- Terminal must support Unicode characters
- Minimum terminal size: `(grid_width * 2 + 4) × (grid_height + 6)`

---

## 8. Enforce Constraints

### 8.1 Game Rule Invariants

| ID | Rule | Verified By | Data | Stress |
|----|------|-------------|------|--------|
| C-001 | NEVER allow opposite direction turns | Unit tests | Direction pairs | Rapid input sequences |
| C-002 | NEVER allow movement when input locked | Unit tests | Concurrent moves | High-frequency input |
| C-003 | NEVER allow snake to exceed grid bounds | Collision detection | Edge positions | All boundary cells |
| C-004 | NEVER process moves when game not Playing | State checks | All non-Playing states | State transition edges |

#### C-001: NEVER allow opposite direction turns

**Constraint:** Snake cannot reverse direction 180° in a single move (e.g., North→South while moving North)

- **Instead:** Input is silently rejected; snake continues in current direction
- **Exception:** None
- **Verified by:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs:280-288` (lines 341-352 in tests)
- **Implementation:** `engine.rs:65-70`
- **Test data:** All direction pairs with `direction.opposite()` checks
- **Stress:** Rapid alternating opposite inputs (North, South, North, South)

#### C-002: NEVER allow concurrent move processing

**Constraint:** Only one move can be processed at a time (enforced via `input_locked` flag)

- **Instead:** Return `false` if input arrives while previous move is processing
- **Exception:** None
- **Verified by:** `engine.rs:52-54`
- **Test data:** Synthetic concurrent calls (not currently tested)
- **Stress:** High-frequency input events exceeding frame rate

#### C-003: NEVER allow snake to exist outside grid bounds

**Constraint:** All snake segments must satisfy `0 <= x < grid_width && 0 <= y < grid_height`

- **Instead:** Collision detected → GameOver
- **Verified by:** `engine.rs:233-238` (`is_within_bounds`)
- **Test data:** Movements toward all four grid edges
- **Scale:** All boundary cells tested
- **Stress:** Gravity falls to bottom edge, movements at corners

#### C-004: NEVER process moves in non-Playing states

**Constraint:** `process_move()` only mutates state when `status == Playing`

- **Instead:** Return `false` immediately
- **Exception:** None
- **Verified by:** `engine.rs:52-54`
- **Test data:** Calls to `process_move()` with status = GameOver, LevelComplete, AllComplete
- **Stress:** Move attempts immediately after state transitions

### 8.2 Serialization Constraints

| ID | Rule | Verified By | Data |
|----|------|-------------|------|
| C-005 | NEVER serialize enums as numeric values | Contract tests | All enum types |
| C-006 | NEVER use snake_case in JSON output | Contract tests | All structs with rename |
| C-007 | NEVER omit required fields in contracts | Serde validation | LevelDefinition, Frame |

#### C-005: NEVER serialize enums as numeric values

**Constraint:** All enums (Direction, CellType, GameStatus) must serialize as string literals

- **Instead:** Use Serde's default string serialization
- **Exception:** None
- **Verified by:** `/home/nntin/git/gSnake/gsnake-core/engine/core/tests/contract_tests.rs:14-103`
- **Test data:** All enum variants serialized and asserted as strings
- **Example:** `Direction::North` → `"North"` (NOT `0`)

#### C-006: NEVER use snake_case in JavaScript-facing JSON

**Constraint:** All struct fields exposed to JS must use camelCase

- **Instead:** Apply `#[serde(rename_all = "camelCase")]` and `#[ts(rename_all = "camelCase")]`
- **Exception:** Internal Rust-only types
- **Verified by:** `contract_tests.rs:132-183`
- **Test data:** GameState serialization with `currentLevel`, `foodCollected`, `totalFood`

#### C-007: NEVER omit required contract fields

**Constraint:** LevelDefinition must have all fields; Frame must contain grid and state

- **Instead:** Serde will fail deserialization
- **Exception:** `ContractError.context` is optional (`skip_serializing_if = "Option::is_none"`)
- **Verified by:** Round-trip tests in `contract_tests.rs:229-307`
- **Test data:** Complete level fixtures with all required fields

### 8.3 Memory and Resource Constraints

| ID | Rule | Verified By |
|----|------|-------------|
| C-008 | NEVER perform I/O in core engine | Code review |
| C-009 | NEVER allocate unbounded collections | Grid size limits |
| C-010 | NEVER embed large assets in WASM without review | Build process |

#### C-008: NEVER perform I/O in core engine

**Constraint:** `gsnake-core` library must remain I/O-free

- **Instead:** Hosts (WASM, CLI, Python) inject data via constructors
- **Exception:** None
- **Verified by:** Dependency analysis (no `std::fs`, `std::net`, etc.)
- **Reason:** Cross-platform compatibility and testability

#### C-009: NEVER allocate unbounded collections

**Constraint:** All game collections bounded by level design (grid size, food count, snake length)

- **Instead:** Collections sized by `LevelDefinition` constants
- **Maximum sizes:**
  - Grid: `grid_size.width × grid_size.height` (typically ≤ 20×20 = 400 cells)
  - Snake: ≤ grid cell count (worst case: fills entire grid)
  - Food: Defined in level data (typically ≤ 10)
  - Obstacles: Defined in level data
- **Verified by:** Level data validation (no dynamic resize in gameplay)

#### C-010: NEVER embed large assets without size review

**Constraint:** WASM binary size must remain minimal for web loading

- **Current approach:** `levels.json` embedded via `include_str!`
- **Mitigation:** Use `opt-level = "z"`, `lto = true`, `wee_alloc`
- **Alternative for large assets:** Fetch at runtime instead of embedding
- **Verified by:** Build artifact size monitoring (not automated)

---

## 9. Build and Development

### 9.1 Development Environment Setup

**Prerequisites:**
- Rust toolchain (1.70+) with `wasm32-unknown-unknown` target
- Node.js 20+
- Python 3.8+ (for build scripts)
- wasm-pack (`cargo install wasm-pack`)

**Installation:**
```bash
# Clone repository
git clone https://github.com/NNTin/gSnake.git
cd gSnake

# Install Rust target
rustup target add wasm32-unknown-unknown

# Install Node dependencies (for web)
npm --prefix gsnake-web install

# Install Playwright for E2E tests
npx playwright install --with-deps chromium
```

### 9.2 Build Commands

#### WASM Build

**Script:** `/home/nntin/git/gSnake/scripts/build_wasm.py`

```bash
python scripts/build_wasm.py
```

**What it does:**
1. Changes to `gsnake-core/engine/bindings/wasm/`
2. Runs `wasm-pack build --target web --release`
3. Output: `pkg/` directory with `.wasm`, `.js`, and `.d.ts` files

**Manual build:**
```bash
cd gsnake-core/engine/bindings/wasm
wasm-pack build --target web --release
```

**Output location:** `gsnake-core/engine/bindings/wasm/pkg/`

**Optimization flags** (`Cargo.toml`):
```toml
[profile.release]
opt-level = "z"      # Optimize for size
lto = true           # Link-time optimization
codegen-units = 1    # Better optimization at cost of compile time
```

#### CLI Build

```bash
cd gsnake-core/engine/bindings/cli
cargo build --release
```

**Output:** `gsnake-core/target/release/gsnake-cli`

**Run:**
```bash
./gsnake-core/target/release/gsnake-cli
```

**Note:** Must run from workspace root for `levels.json` access

#### TypeScript Type Generation

**Script:** `/home/nntin/git/gSnake/scripts/generate_ts_types.py`

```bash
python scripts/generate_ts_types.py
```

**What it does:**
1. Runs `cargo run -p gsnake-core --bin export_ts`
2. Generates TypeScript type definitions from Rust structs
3. Output: `/home/nntin/git/gSnake/gsnake-web/types/models.ts`

**Implementation:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/bin/export_ts.rs`

**Exported types:**
- Position, GridSize, Direction, CellType, GameStatus
- LevelDefinition, GameState, Frame
- ContractError, ContractErrorKind

**Workflow:**
1. Rust types derive `#[derive(TS)]` trait
2. `export_ts.rs` calls `T::export()` for each type
3. Types written to single `models.ts` file
4. Web app imports types for compile-time safety

### 9.3 Testing

#### Unit Tests

**Core library tests:**
```bash
cd gsnake-core/engine/core
cargo test
```

**Coverage:**
- Basic movement and gravity
- Opposite direction blocking
- Food collection and growth
- Collision detection
- Exit and level completion

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs:292-509`

#### Contract Tests

**Generate test fixtures:**
```bash
cd gsnake-core/engine/core
cargo test generate_contract_fixtures
```

**Validates:**
- String enum serialization (not numeric)
- CamelCase JSON field names
- Round-trip serialization integrity
- Optional field omission (`ContractError.context`)

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/tests/contract_tests.rs`

**Fixture output:** `gsnake-core/engine/core/tests/fixtures/`

#### End-to-End Tests

**Playwright browser tests:**
```bash
npm run test:e2e
```

**What it tests:**
- Keyboard input simulation (arrow keys, WASD)
- DOM grid state assertions
- Movement mechanics
- Gravity physics (snake falls off platform)

**Configuration:** `/home/nntin/git/gSnake/package.json`
```json
{
  "scripts": {
    "test:e2e": "playwright test --reporter=list",
    "report": "allure open allure-report"
  }
}
```

**Reporting:**
- Allure reports with video recording
- Screenshots on failure
- Chromium browser with `--with-deps`

#### Integration Testing

**Script:** `/home/nntin/git/gSnake/scripts/test_integration.py`

**Orchestrates:**
1. Rebuild WASM binary
2. Install web dependencies
3. Run E2E tests
4. Generate reports

**Used by:** CI/CD and local development

### 9.4 CI/CD Pipeline

**Workflow:** `/home/nntin/git/gSnake/.github/workflows/deploy.yml`

**Triggers:**
- Push to `main` branch
- Version tags (e.g., `v0.2.1`)

**Steps:**
1. **Setup:**
   - Rust toolchain + `wasm32-unknown-unknown` target
   - Node.js 20
   - Playwright with Chromium

2. **Build:**
   - Generate TypeScript types (`scripts/generate_ts_types.py`)
   - Build WASM (`scripts/build_wasm.py`)
   - TypeScript type checking (`npm --prefix gsnake-web run check`)

3. **Test:**
   - Contract tests (`cargo test` in `engine/core`)
   - E2E tests (`scripts/test_integration.py`)

4. **Deploy (main branch):**
   - Deploy to `/gSnake/main/` on GitHub Pages
   - Inject base path via `VITE_BASE_PATH=/gSnake/main`

5. **Deploy (version tags):**
   - Deploy to `/gSnake/v{version}/`
   - Update `versions.json` registry in `gh-pages` branch
   - Inject base path via `VITE_BASE_PATH=/gSnake/v{version}`

**Deployment strategy:**
- Main branch: Latest development build
- Version tags: Stable releases with version registry
- Base path injection: Supports subdirectory hosting

### 9.5 WASM Memory Management

**Allocator:** `wee_alloc::WeeAlloc`

**Configuration:**
```rust
// /home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/src/lib.rs
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;
```

**Purpose:**
- Smaller binary size (~1-2KB allocator vs. default ~10KB)
- Optimized for WASM environments
- Trade-off: Potentially slower allocation, but negligible for game's usage pattern

**Memory layout:**
- Static data: Embedded `levels.json` via `include_str!` (compile-time)
- Stack: Game state structs (Position, Snake, LevelState)
- Heap: Collections (Vec for snake segments, food, obstacles)

**Memory constraints:**
- No explicit limits (relies on browser/WASM runtime limits)
- Bounded by level design (grid size, entity counts)
- No unbounded growth during gameplay

**Panic handling:**
```rust
#[wasm_bindgen(start)]
pub fn init_panic_hook() {
    console_error_panic_hook::set_once();
}
```

**Purpose:** Route panic messages to browser console with stack traces

### 9.6 Python Bindings (Placeholder)

**Status:** Minimal/Placeholder implementation

**Location:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/py/`

**Packaging:** `pyproject.toml` with `pip install -e .`

**Expected architecture:**
- PyO3 bindings to Rust core
- Similar API to WASM/CLI (uniform level access)
- Python-native error handling

**Current state:** No PyO3 implementation found in sources

**Future implementation path:**
1. Add PyO3 dependency
2. Expose `GameEngine` via Python wrapper class
3. Serialize `Frame` to Python dictionaries
4. Match WASM API surface for consistency

---

## 10. Performance Characteristics

### 10.1 Algorithm Complexity

| Operation | Time Complexity | Space Complexity | Notes |
|-----------|-----------------|------------------|-------|
| **process_move** | O(S + F + K×S) | O(1) | S=snake length, F=food count, K=gravity fall distance |
| **check_collision** | O(S) | O(1) | Self-collision loop over segments |
| **apply_gravity** | O(K × S) | O(1) | K iterations × S segment updates |
| **can_snake_fall** | O(S × (O + F)) | O(1) | Each segment checks obstacles and food |
| **generate_frame** | O(W × H + S + F + O) | O(W × H) | Grid allocation + entity placement |

**Key bottlenecks:**
- **Gravity loop:** Worst case = snake falls entire grid height (e.g., K=20)
- **Frame generation:** Full grid allocation every frame (e.g., 20×20 = 400 cells)

**Optimization notes:**
- Current implementation prioritizes correctness and readability
- No profiling or benchmarking infrastructure
- Game scale is small enough that performance is not a concern (grids typically ≤ 20×20)

### 10.2 Memory Usage

**Per-frame allocation:**
- `Frame.grid`: `Vec<Vec<CellType>>` = W × H × 1 byte ≈ 400 bytes for 20×20 grid
- Serialized to JSON: ~2-3KB depending on grid size

**Engine state:**
- `Snake.segments`: `Vec<Position>` = S × 8 bytes (max ~400 positions)
- `LevelState.food`: `Vec<Position>` = F × 8 bytes (typically ~80 bytes for 10 food)
- `LevelState.obstacles`: `Vec<Position>` = O × 8 bytes
- Total state: < 5KB for typical levels

**WASM bundle size:**
- Not measured in current codebase
- Optimization flags in place (`opt-level = "z"`, `lto = true`)
- `wee_alloc` reduces allocator overhead by ~8KB

### 10.3 Known Gaps

**No performance testing infrastructure:**
- No benchmarks (Criterion.rs or similar)
- No WASM bundle size tracking
- No memory profiling
- No frame generation profiling

**Potential future work:**
- Add `#[bench]` tests for critical paths
- Track WASM bundle size in CI
- Profile frame generation for larger grids
- Consider differential rendering (only changed cells)

---

## 11. References

### 11.1 File Paths

All paths are absolute from repository root:

| Component | Path |
|-----------|------|
| **Core Library** | `/home/nntin/git/gSnake/gsnake-core/engine/core/` |
| Models | `/home/nntin/git/gSnake/gsnake-core/engine/core/src/models.rs` |
| Engine Logic | `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs` |
| Level Parser | `/home/nntin/git/gSnake/gsnake-core/engine/core/src/levels.rs` |
| **WASM Binding** | `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/src/lib.rs` |
| **CLI Application** | `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/` |
| CLI Main | `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/main.rs` |
| CLI UI | `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/ui.rs` |
| CLI Input | `/home/nntin/git/gSnake/gsnake-core/engine/bindings/cli/src/input.rs` |
| **Test Suite** | `/home/nntin/git/gSnake/gsnake-core/engine/core/tests/contract_tests.rs` |
| **Level Data** | `/home/nntin/git/gSnake/gsnake-core/engine/core/data/levels.json` |
| **Workspace Config** | `/home/nntin/git/gSnake/gsnake-core/Cargo.toml` |

### 11.2 Key Algorithms

| Algorithm | File | Lines | Description |
|-----------|------|-------|-------------|
| **Gravity Application** | `engine.rs` | 169-182 | Continuous downward movement until platform detection |
| **Collision Detection** | `engine.rs` | 211-230 | Out-of-bounds, obstacle, and self-collision checks |
| **Platform Detection** | `engine.rs` | 185-208 | Checks if snake can fall (obstacles and food block) |
| **Move Processing** | `engine.rs` | 48-121 | Complete turn logic: validate → move → gravity → check state |
| **Frame Generation** | `engine.rs` | 124-166 | Convert internal state to 2D grid for rendering |
| **Opposite Direction** | `engine.rs` | 280-288 | Prevents 180° turns (e.g., North→South) |

### 11.3 Dependencies

#### Core Library
```toml
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
ts-rs = "7.0"  # TypeScript type generation
```

#### WASM Binding
```toml
wasm-bindgen = "0.2"
serde-wasm-bindgen = "0.6"
js-sys = "0.3"
web-sys = { version = "0.3", features = ["console"] }
wee_alloc = "0.4"  # Small allocator for WASM
console_error_panic_hook = "0.1"
```

#### CLI Application
```toml
ratatui = "0.28"    # Terminal UI framework
crossterm = "0.28"  # Cross-platform terminal control
anyhow = "1.0"      # Error handling
```

### 11.4 Related Documentation

- **Epic Brief:** `/home/nntin/git/gSnake/epics/rust-core/epic-brief.md`
- **Core Flows:** `/home/nntin/git/gSnake/epics/rust-core/core-flows.md`
- **Tech Plan:** `/home/nntin/git/gSnake/epics/rust-core/tech-plan.md`

### 11.5 External Resources

- [wasm-bindgen Documentation](https://rustwasm.github.io/wasm-bindgen/)
- [Ratatui Book](https://ratatui.rs/)
- [Crossterm API](https://docs.rs/crossterm/)
- [Serde Guide](https://serde.rs/)

---

**End of Specification**
