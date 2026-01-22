# Technical Plan - Modular Repository Restructuring

This plan outlines the restructuring of the `gSnake` repository into a modular, submodule-ready architecture.

## Architectural Approach

The system transitions from a monolithic flat structure to a nested, workspace-oriented modular design. This enables each component to eventually function as an independent Git submodule with its own lifecycle and CI/CD pipelines.

### 1. gsnake-core (Rust Workspace)
- **Role**: The source of truth for game logic and data.
- **Structure**: A Rust workspace moved into `gsnake-core/`.
  - **engine/core**: Pure Rust game engine. Owns `data/levels.json`.
  - **engine/bindings/**: Sub-crates for platform-specific integration.
    - `wasm/`: Generates WASM bindings for web consumption (package name `gsnake-wasm`).
    - `cli/`: Terminal interface using `ratatui`.
    - `py/`: Python bindings using `pyproject.toml`.
- **Constraint**: `gsnake-core` holds the workspace-level `Cargo.toml`. Root Rust configs are duplicated into `gsnake-core` (e.g. `rust-toolchain.toml`, lint configs) so `gsnake-core` builds in isolation. Root helper scripts `cd` into `gsnake-core` for commands run from the parent directory.

### 2. gsnake-web (Svelte Frontend)
- **Role**: Standalone Svelte web application.
- **Integration**: Consumes the WASM package from `gsnake-core/engine/bindings/wasm/pkg` via a `file:` dependency after `wasm-pack build`. A root script rebuilds the WASM package before dev/test to keep the `file:` target up to date.
- **Constraint**: `gsnake-web` is a git submodule containing all Svelte files; no web app source remains at the repository root. Vite/Svelte configs live inside `gsnake-web`. Root Playwright config remains for integration tests.
- **Migration Note**: The existing Svelte app is moved into a fresh `gsnake-web` repository (new history) at `git@github.com:NNTin/gsnake-web.git` and reattached to the parent repo as a submodule during the restructure.

### 3. gsnake-python (Placeholder)
- **Role**: Placeholder boundary for future Python integration.
- **Structure**: Git submodule pointer to an empty repository (no implementation in current scope).

### 4. Root (Integration Layer)
- **Role**: Orchestration, global CI/CD, and cross-module specifications (`epics/`).
- **State**: Drastically reduced; implementation detail is delegated to sub-modules. Existing GitHub Actions remain at the root for integration testing and Svelte deployment during transition.
- **Orchestration**: Integration workflows use Python as the orchestration layer. Root `scripts/` provides repeatable build/link/test entrypoints for local dev and CI. Submodules are pinned by SHA in the parent repo, and integration runs are gated on those SHAs.
  - **Note**: The parent repo must initialize/update the `gsnake-web` submodule before running integration or web workflows.

## Data Model

The restructuring centralizes game data within the core engine to ensure consistency across all platforms (Web, CLI, Python).

### Entity: Game Level
- **Source**: `gsnake-core/engine/core/data/levels.json` (moved from root).
- **Structure**:
  - `id`: Unique identifier (Integer).
  - `name`: Display name (String).
  - `gridSize`: Boundary dimensions (`width`, `height`).
  - `snake`: Initial body coordinates (Array of Points).
  - `obstacles`: Static collision points (Array of Points).
  - `food`: Consumable items that also act as platforms (Array of Points).
  - `exit`: Target coordinate for level completion (Point).

### Entity: Shared State (WASM/Core)
- **Interface**: Data models defined in `gsnake-core/engine/core/src/engine.rs` are serialized via `serde` for cross-boundary communication (Rust <-> TS).
  - **Constraint**: Levels are exposed through bindings with a uniform API in core and WASM; the web UI reads level/state data through the WASM API only. `levels.json` is embedded in the WASM package during build and exposed through bindings.

## Component Architecture

The architecture enforces strict boundaries between engine logic, platform bindings, and user interfaces.

### 1. Game Engine (engine/core)
- **Responsibility**: Physics (gravity), collision detection, snake movement, and level state management.
- **Interface**: Exposed as a Rust library for consumption by bindings.

### 2. Platform Bindings (engine/bindings)
- **WASM Bridge**: Translates Rust structures to JavaScript-friendly objects using `wasm-bindgen`.
- **CLI Wrapper**: Manages terminal I/O and renders the game state using `ratatui`.
- **Python Bridge**: Provides bindings with `pyproject.toml` for Python scriptability.

### 3. Web UI (gsnake-web)
- **Responsibility**: Rendering the game grid, handling keyboard input, and managing high-level UI states (Modals, Overlays).
- **Integration Point**: `WasmGameEngine.ts` acts as the service layer between Svelte stores and the WASM binary.

### 4. CLI App (gsnake-cli)
**Responsibility**: Direct-to-terminal gameplay.
**Integration Point**: Implemented as `gsnake-core/engine/bindings/cli`, linking directly to the engine.

## Folder and File Layout (Target)
```
gSnake/
├── epics/
│   └── folder-structure/
│       ├── epic-brief.md
│       ├── core-flows.md
│       └── tech-plan.md
├── scripts/
│   ├── build_core.py
│   ├── build_wasm.py
│   ├── link_web.py
│   └── test_integration.py
├── playwright.config.ts          # Root integration tests
├── rust-toolchain.toml           # Root Rust config only
├── gsnake-core/
│   ├── rust-toolchain.toml       # Duplicated Rust config for standalone builds
│   ├── Cargo.toml                # Workspace root
│   └── engine/
│       ├── core/
│       │   ├── Cargo.toml
│       │   └── data/
│       │       └── levels.json
│       └── bindings/
│           ├── wasm/
│           │   ├── Cargo.toml
│           │   └── pkg/           # Generated (gitignored)
│           ├── cli/
│           │   └── Cargo.toml
│           └── py/
│               └── pyproject.toml
├── gsnake-web/                  # Git submodule
│   ├── package.json
│   ├── svelte.config.js
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── src/
└── gsnake-python/                 # Submodule pointer (empty repo)
```
