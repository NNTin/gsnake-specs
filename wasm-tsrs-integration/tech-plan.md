# Tech Plan: wasm-bindgen + ts-rs Integration

## Architectural Approach

- Split domain modeling from execution: `gsnake-core/engine/core` owns Rust domain logic and ts-rs derives; `gsnake-core/engine/bindings/wasm` remains a thin wasm-bindgen execution layer with strict typed serialization.
- Authoritative rendering from Rust: UI consumes `Frame.grid` produced by core; Svelte stops recomputing grid and avoids desync between food/obstacles and runtime state.
- Shared type safety: ts-rs generates `gsnake-web/types/models.ts`, and TS imports these types directly; avoid hand-written duplicates for domain models, enums, and payloads.
- Runtime boundaries: wasm-bindgen exports a strict API; all serialized state uses serde (and thus ts-rs) with string enums only.
- Iteration velocity: keep wasm layer small to reduce JS glue changes; prefer direct serialization of domain structs to reduce manual mapping logic.
- Common demeanor (contract posture): stable shapes, typed error payloads, and predictable event order; UI trusts the core as source of truth and never re-derives gameplay state.

## Data Model

- `LevelDefinition` (static): `id`, `name`, `grid_size`, `snake` (initial segments), `obstacles`, `food`, `exit`, `snake_direction` (required). Serialized via serde; ts-rs derives for TS usage.
- `LevelState` (runtime): mutable entities derived from `LevelDefinition` at load time: `snake`, `food`, `moves`, `status`, etc. Not exposed directly to UI; used by engine to compute `Frame`.
- `Frame` (authoritative render payload): `grid: Vec<Vec<CellType>>` and `state: GameState`. Serialized via serde; ts-rs derives. `snake` is removed or treated as debug-only and not relied on by the UI.
- `GameState`: retains `status`, `current_level`, `moves`, `food_collected`, `total_food`; serialized for UI status, modals, and counters.
- `CellType`, `GameStatus`, and `Direction` serialize as string enums matching ts-rs output; numeric or object enum representations are invalid.
- `ContractError`: typed error payload with `kind`, `message`, and optional `context` returned from wasm boundary.
- `ProcessMoveResult`: `{ frame: Frame }` on success; `ContractError` on failure.

```mermaid
erDiagram
  LevelDefinition ||--|| LevelState : derives
  LevelState ||--|| Frame : generates
  Frame ||--|| GameState : contains
  LevelDefinition ||--o{ Position : defines
  LevelDefinition ||--o{ Position : obstacles
  LevelDefinition ||--o{ Position : food
  LevelDefinition ||--|| Position : exit
```

## Component Architecture

- Interface/Contract (UI \<-> Core via wasm-bindgen):
  - Input: `LevelDefinition` and `Direction` as strict serialized types.
  - Output: `Frame` payloads; `GameState` is read from `Frame.state`.
  - Required order: init -> level load -> initial frame emission -> input loop; every accepted input yields a new `Frame`.
  - Error semantics: wasm boundary returns `ContractError` payloads; UI shows a failure state and logs `kind`.
  - Source of truth: UI renders from `Frame.grid` only; derived or cached gameplay state in UI is advisory.
- Core (`gsnake-core/engine/core`):
  - `models.rs`: all domain structs/enums with serde + ts-rs derives.
  - `engine.rs`: pure logic, consumes `LevelDefinition`, produces `LevelState` and `Frame`.
- wasm wrapper (`gsnake-core/engine/bindings/wasm`):
  - `WasmGameEngine` exposes `new(level)`, `processMove(direction)`, `getFrame()`, `getLevels()`, and emits an initial frame after construction or when `onFrame` is registered.
  - Uses `serde_wasm_bindgen` to accept/return strict serialized payloads; no string parsing or enum normalization.
  - `processMove` returns `Frame` on success; failures return `ContractError`.
- Web (`gsnake-web`):
  - `types/` imports generated `models.ts` from ts-rs.
  - `engine/WasmGameEngine.ts` only forwards payloads and dispatches events; no manual status/enum mapping or normalization; includes a single retry on init failure.
  - `components/GameGrid.svelte` renders from `Frame.grid` and uses grid size dynamically from the frame.
- Tests:
  - Rust: unit tests for `engine.rs` and serialization of `LevelDefinition`, `Frame`, `GameState`.
  - Web: integration tests ensure `Frame.grid` drives UI correctly and that ts-rs types align with runtime payloads (string enums only).
  - Contract: both Rust and web tests validate string enum serialization and payload shape.

```mermaid
erDiagram
  UIClient ||--o{ LevelDefinition : provides
  UIClient ||--o{ Direction : sends
  WasmBinding ||--o{ Frame : emits
  WasmBinding ||--|| LevelDefinition : consumes
  WasmBinding ||--|| Direction : receives
  WasmBinding ||--|| GameCore : wraps
  GameCore ||--|| Frame : produces
```
