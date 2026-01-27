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

## Troubleshooting

### Git Dependency Not Found

**Symptom:** Build fails with error like `failed to load source for dependency` or `repository not found`.

**Causes and Solutions:**
- **Incorrect branch name**: Verify the branch name is `main` (not `master` or other)
- **Repository access**: Ensure you have access to `https://github.com/nntin/gsnake`
- **Network issues**: Check your internet connection and GitHub access
- **Submodule not published**: Verify the dependency code exists in the main branch

**Verification:**
```bash
# Test repository access
git ls-remote https://github.com/nntin/gsnake refs/heads/main

# Verify submodule path exists
git clone --depth 1 --branch main https://github.com/nntin/gsnake /tmp/test-repo
ls /tmp/test-repo/gsnake-core
```

### Local Override Not Working

**Symptom:** Root repository build still uses git dependencies instead of local paths.

**Causes and Solutions:**
- **Detection script not running**: Verify preinstall/build scripts are executing
- **Missing sibling directories**: Check that `../gsnake-core/.git` exists
- **Cache issues**: Clear build caches and reinstall dependencies

**Verification for Rust:**
```bash
cd gsnake-levels
# Check if .cargo/config.toml exists with [patch] section
cat .cargo/config.toml
# Should show: [patch."https://github.com/nntin/gsnake"]
```

**Verification for JavaScript:**
```bash
cd gsnake-web
# Check package.json after npm install
cat package.json | grep gsnake-core
# Should show: "file:../gsnake-core/..." if in root repo
```

### WASM Build Fails

**Symptom:** `wasm-pack build` fails or WASM target not found.

**Causes and Solutions:**
- **Missing Rust target**: Install the WASM target
  ```bash
  rustup target add wasm32-unknown-unknown
  ```
- **Missing wasm-pack**: Install wasm-pack
  ```bash
  cargo install wasm-pack
  ```
- **Outdated toolchain**: Update Rust toolchain
  ```bash
  rustup update stable
  ```

### Forcing Standalone Mode

To test standalone builds while in the root repository (ignoring local overrides):

**Rust (Cargo):**
```bash
# Temporarily rename .git to disable detection
mv ../.git ../.git.disabled
cargo clean
cargo build
mv ../.git.disabled ../.git
```

**JavaScript (npm):**
```bash
# Set environment variable to skip detection
FORCE_GIT_DEPS=1 npm install
# Or temporarily rename .git
mv ../.git ../.git.disabled
rm -rf node_modules package-lock.json
npm install
mv ../.git.disabled ../.git
```

### Verifying Dependency Mode

**Check which mode is active:**

**Rust:**
```bash
cd gsnake-levels
cargo tree | grep gsnake-core
# Git mode: shows "https://github.com/nntin/gsnake?branch=main#..."
# Local mode: shows "path+file:///path/to/gsnake-core"
```

**JavaScript:**
```bash
cd gsnake-web
npm ls gsnake-core
# Git mode: shows "git+https://github.com/..."
# Local mode: shows "file:../gsnake-core/..."
```
