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

## Dependency Resolution Strategy

The gSnake project uses a **dual-mode dependency resolution system** to support both standalone submodule builds and integrated root repository development.

### Git Branch Dependencies (Standalone Mode)

When submodules are built independently (outside the root repository), they use git branch dependencies to access code from other submodules.

#### Rust (Cargo.toml)
```toml
[dependencies]
gsnake-core = { git = "https://github.com/nntin/gsnake", branch = "main", package = "gsnake-core" }
```

#### JavaScript/TypeScript (package.json)
```json
{
  "dependencies": {
    "gsnake-core": "git+https://github.com/nntin/gsnake.git#main:gsnake-core/engine/bindings/wasm/pkg"
  }
}
```

### Branch Strategy

All git dependencies target the **`main` branch**. This ensures:
- Submodules always use the latest stable version
- Consistent behavior across all standalone builds
- Simplified dependency management (no need to update commit SHAs or tags)

### Auto-Detection Mechanism (Root Repository Mode)

When submodules are built within the root repository, they automatically detect the local environment and override git dependencies with local path dependencies.

**Detection Logic:**
- Check if `../.git` exists (indicates root repository)
- Check if sibling submodules exist (e.g., `../gsnake-core/.git`)
- If both conditions are met: use local paths
- Otherwise: use git dependencies

**Rust Implementation:**
- Build scripts create `.cargo/config.toml` with `[patch]` section
- Patches redirect git dependencies to local paths

**JavaScript Implementation:**
- Preinstall scripts dynamically modify `package.json`
- Replace git URLs with `file:` paths before installation

### Versioning Strategy

- **Root Repository**: Uses semantic versioning in `package.json` for releases
- **Submodules**: Track the `main` branch of dependencies (no fixed versions in standalone mode)
- **Local Development**: Always uses the exact local code (no version constraints)

### Examples

#### Rust Git Dependency (gsnake-levels/Cargo.toml)
```toml
[package]
name = "gsnake-levels"
version = "0.1.0"

[dependencies]
gsnake-core = { git = "https://github.com/nntin/gsnake", branch = "main", package = "gsnake-core" }
```

#### JavaScript Git Dependency (gsnake-web/package.json)
```json
{
  "name": "gsnake-web",
  "version": "1.0.0",
  "dependencies": {
    "gsnake-core": "git+https://github.com/nntin/gsnake.git#main:gsnake-core/engine/bindings/wasm/pkg"
  }
}
```
