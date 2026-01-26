# Repo Architecture (Interdependencies)

```mermaidgraph TD
  %% Submodules / repos (each in its own subgraph)
  subgraph ROOT_REPO["gsnake (root repo)"]
    CORE["gsnake-core (root folder)"]
    SCRIPTS["scripts/ (root folder)"]
    WASM_PKG["gsnake-core/engine/bindings/wasm/pkg (build output)"]
    FIXTURES["gsnake-core/engine/core/tests/fixtures"]
    LEVELS_REPO
    WEB_REPO
    SPECS_REPO
    PY_REPO
    EDITOR_REPO
  end

  subgraph LEVELS_REPO["gsnake-levels (submodule)"]
    LEVELS["gsnake-levels"]
  end

  subgraph WEB_REPO["gsnake-web (submodule)"]
    WEB["gsnake-web"]
  end

  subgraph SPECS_REPO["gsnake-specs (submodule)"]
    SPECS["gsnake-specs"]
  end

  subgraph PY_REPO["gsnake-python (submodule, empty/ignored)"]
    PY["gsnake-python"]
  end

  subgraph EDITOR_REPO["gsnake-editor (submodule)"]
    EDITOR["gsnake-editor"]
  end

  %% Core relationships
  CORE --> WASM_PKG

  %% gsnake-web dependencies
  WEB --> CORE
  WEB --> SCRIPTS
  SCRIPTS -->|generate_ts_types.py| WEB
  SCRIPTS -->|build_wasm.py| WASM_PKG
  SCRIPTS -->|build_core.py| CORE
  SCRIPTS -->|"generate_ts_types.py (cargo export_ts)"| CORE
  WASM_PKG -->|file: dependency| WEB
  FIXTURES -->|contract fixtures| WEB
  CORE -->|export_ts writes types| WEB

  %% gsnake-levels dependency
  LEVELS -->|path dep ../gsnake-core| CORE
  LEVELS -->|runs gsnake-cli via manifest| CORE

  %% gsnake-editor (submodule)
  EDITOR -->|test-mode relies on| WEB

  %% Specs
  SPECS -->|docs only| CORE

  %% Current pain
  WEB -.->|cannot build alone| CORE
  LEVELS -.->|cannot build alone| CORE
  EDITOR -.->|cannot build alone| CORE

  %% Future goal
  GOAL[Goal: publish/build dependencies externally]
  CORE --> GOAL
  WEB --> GOAL
  LEVELS --> GOAL
  EDITOR --> GOAL

  %% Subgraph styling
  style ROOT_REPO fill:#e6f2ff,stroke:#1e88e5,stroke-width:2px
  style LEVELS_REPO fill:#f3e5f5,stroke:#8e24aa,stroke-width:2px
  style WEB_REPO fill:#e8f5e9,stroke:#43a047,stroke-width:2px
  style SPECS_REPO fill:#fff3e0,stroke:#fb8c00,stroke-width:2px
  style PY_REPO fill:#fce4ec,stroke:#d81b60,stroke-width:2px
  style EDITOR_REPO fill:#ede7f6,stroke:#5e35b1,stroke-width:2px
```
