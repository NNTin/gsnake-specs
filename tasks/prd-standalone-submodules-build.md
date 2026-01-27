# PRD: Standalone Submodules Build

## Introduction

Today, several gSnake submodules (gsnake-web, gsnake-levels, gsnake-editor, gsnake-specs) cannot build or test when checked out alone because they reference sibling repos using local path dependencies (e.g., `../gsnake-core`). This PRD defines a migration to git branch-based dependencies so each submodule can build/test independently while preserving seamless local development in the root repo.

**Two Working Modes:**
1. **Standalone submodule**: Clone a single submodule → build and test work independently
2. **Root repo with submodules**: Clone parent repo with submodules → all work together with auto-detected local paths

## Goals

- Each submodule (gsnake-core, gsnake-web, gsnake-levels, gsnake-editor, gsnake-specs) can build/test without any sibling repos present
- All cross-repo dependencies resolve via git branch URLs by default (targeting `main` branch)
- Root repo auto-detects local submodules and uses them without manual configuration
- Each submodule has its own CI workflow that validates standalone builds
- All CI workflows are compatible with nektos/act for local testing
- Root repo versioning (semver in package.json) remains unchanged
- Documentation in `gsnake-specs/tasks/repo-architecture.md` reflects the new dependency model

## User Stories

### US-001: Define git branch dependency strategy and update documentation
**Description:** As a maintainer, I want a clear, documented strategy for git branch-based dependencies so all contributors understand how submodule dependencies work in both standalone and root repo contexts.

**Acceptance Criteria:**
- [ ] Create new section in `gsnake-specs/tasks/repo-architecture.md` titled "Dependency Resolution Strategy"
- [ ] Document git URL format for each submodule dependency (e.g., `git+https://github.com/nntin/gsnake?branch=main#subdirectory=gsnake-core`)
- [ ] Document branch strategy: all git dependencies target `main` branch
- [ ] Explain auto-detection mechanism: build scripts check for `../<submodule>/.git` to detect root repo context
- [ ] Document how root repo versioning (semver in package.json) differs from submodule branch tracking
- [ ] Include examples for Rust (Cargo.toml) and JavaScript (package.json) git dependencies
- [ ] Add troubleshooting section for common dependency resolution issues
- [ ] Update architecture diagram to show both git and local path resolution modes

### US-002: Migrate gsnake-core to git-resolvable dependency
**Description:** As a developer, I want gsnake-core to be consumable via git URL so downstream submodules (levels, web) can depend on it independently.

**Acceptance Criteria:**
- [ ] Add `.cargo/config.toml` to gsnake-core (if needed for workspace overrides)
- [ ] Verify gsnake-core can be referenced via git URL: `git+https://github.com/nntin/gsnake?branch=main#subdirectory=gsnake-core`
- [ ] `cargo build` succeeds in gsnake-core standalone (clone only that submodule)
- [ ] `cargo test` succeeds in gsnake-core standalone
- [ ] `cargo doc` succeeds in gsnake-core standalone
- [ ] Add gsnake-core/.github/workflows/ci.yml with jobs: build, test, doc
- [ ] CI workflow runs on: push to main, pull requests, workflow_dispatch
- [ ] CI workflow is compatible with nektos/act (no GitHub-specific features that fail locally)
- [ ] CI workflow validates WASM build target (wasm32-unknown-unknown)
- [ ] All tests pass in CI
- [ ] Document gsnake-core standalone build instructions in its README.md

### US-003: Migrate gsnake-levels to git dependency on gsnake-core
**Description:** As a developer, I want gsnake-levels to resolve gsnake-core via git URL so it builds independently without sibling repos.

**Acceptance Criteria:**
- [ ] Update gsnake-levels/Cargo.toml: replace `path = "../gsnake-core"` with git dependency:
  ```toml
  [dependencies]
  gsnake-core = { git = "https://github.com/nntin/gsnake", branch = "main", package = "gsnake-core" }
  ```
- [ ] Add build script (e.g., `build.sh` or `build.rs`) that detects root repo context
- [ ] Auto-detection logic: check if `../.git` exists AND `../gsnake-core/.git` exists
- [ ] If root repo detected: override with local path via `.cargo/config.toml` or environment variable
- [ ] Replace any `--manifest-path ../gsnake-core` CLI usage with `--package gsnake-core` or remove
- [ ] `cargo build` succeeds in gsnake-levels standalone (without sibling repos)
- [ ] `cargo test` succeeds in gsnake-levels standalone
- [ ] `cargo build` succeeds in gsnake-levels within root repo (using local gsnake-core)
- [ ] `cargo test` succeeds in gsnake-levels within root repo
- [ ] Add gsnake-levels/.github/workflows/ci.yml with standalone build/test validation
- [ ] CI workflow compatible with nektos/act
- [ ] Document override mechanism in gsnake-levels/README.md

### US-004: Migrate gsnake-web to git dependency on gsnake-core WASM
**Description:** As a developer, I want gsnake-web to resolve gsnake-core WASM artifacts via git dependency so it builds independently.

**Acceptance Criteria:**
- [ ] Update gsnake-web/package.json: replace `"gsnake-core": "file:../gsnake-core/engine/bindings/wasm/pkg"` with git dependency:
  ```json
  "gsnake-core": "git+https://github.com/nntin/gsnake.git#main:gsnake-core/engine/bindings/wasm/pkg"
  ```
- [ ] Create gsnake-web/scripts/detect-local-deps.js that checks for `../../gsnake-core/.git`
- [ ] Auto-detection: if root repo detected, use `file:../gsnake-core/engine/bindings/wasm/pkg` instead
- [ ] Integrate detection script into gsnake-web build process (preinstall or custom install script)
- [ ] Update gsnake-web scripts to handle WASM build:
  - If standalone: expect prebuilt WASM from git dependency
  - If root repo: run `python3 ../../scripts/build_wasm.py` before build
- [ ] `npm install` succeeds in gsnake-web standalone
- [ ] `npm run build` succeeds in gsnake-web standalone
- [ ] `npm run check` (typecheck/lint) succeeds in gsnake-web standalone
- [ ] `npm test` succeeds in gsnake-web standalone
- [ ] `npm run build` succeeds in gsnake-web within root repo (using local WASM)
- [ ] Contract test fixtures are either:
  - Committed to gsnake-web for standalone mode, OR
  - Generated via git submodule or documented as external dependency
- [ ] Add gsnake-web/.github/workflows/ci.yml with: install, build, typecheck, test
- [ ] CI workflow compatible with nektos/act (cache npm properly for local runs)
- [ ] Document standalone build requirements in gsnake-web/README.md (e.g., prebuilt WASM)

### US-005: Migrate gsnake-editor to standalone build (optional gsnake-web)
**Description:** As a developer, I want gsnake-editor to build and test without requiring gsnake-web as a sibling repo.

**Acceptance Criteria:**
- [ ] gsnake-editor test/integration mode configuration uses environment variable (e.g., `GSNAKE_WEB_URL`)
- [ ] Default value: `http://localhost:3000` (assumes gsnake-web running separately)
- [ ] Document in gsnake-editor/README.md: "To test editor, either run gsnake-web locally or set GSNAKE_WEB_URL"
- [ ] Remove any hardcoded sibling path references (e.g., `../gsnake-web`)
- [ ] `npm install` succeeds in gsnake-editor standalone
- [ ] `npm run build` succeeds in gsnake-editor standalone
- [ ] `npm run check` (typecheck/lint) succeeds in gsnake-editor standalone
- [ ] `npm test` (unit tests) succeeds in gsnake-editor standalone
- [ ] Integration tests are skipped if GSNAKE_WEB_URL is unreachable (with warning message)
- [ ] Add gsnake-editor/.github/workflows/ci.yml with: install, build, typecheck, test (unit only)
- [ ] CI workflow compatible with nektos/act
- [ ] Document how to run full integration tests in gsnake-editor/README.md

### US-006: Migrate gsnake-specs to standalone build
**Description:** As a maintainer, I want gsnake-specs to build documentation independently without relying on parent repo paths.

**Acceptance Criteria:**
- [ ] Verify all references in gsnake-specs docs are either:
  - Relative paths within gsnake-specs, OR
  - External git URLs (e.g., GitHub links to other submodules), OR
  - Explicitly documented as optional/requires root repo
- [ ] `mkdocs build` succeeds in gsnake-specs standalone
- [ ] `mkdocs serve` succeeds in gsnake-specs standalone
- [ ] Any Python dependencies listed in requirements.txt or pyproject.toml
- [ ] Add gsnake-specs/.github/workflows/ci.yml with: install, mkdocs build, link check
- [ ] CI workflow compatible with nektos/act
- [ ] Document build instructions in gsnake-specs/README.md

### US-007: Create root repo override mechanism with auto-detection
**Description:** As a developer working in the root repo, I want build scripts to automatically use local submodules without manual configuration.

**Acceptance Criteria:**
- [ ] Create `scripts/detect-repo-context.sh` that exports environment variables indicating root repo context
- [ ] Detection logic:
  ```bash
  if [ -d "../.git" ] && [ -d "../gsnake-core/.git" ]; then
    export GSNAKE_ROOT_REPO=true
    export GSNAKE_CORE_PATH=../gsnake-core
    # ... other submodules
  fi
  ```
- [ ] Update root repo scripts (build_wasm.py, generate_ts_types.py) to respect local paths when GSNAKE_ROOT_REPO=true
- [ ] Update root repo package.json scripts to source detection script before builds
- [ ] Verify all root repo workflows pass with submodules checked out:
  - `npm --prefix gsnake-web ci`
  - `npm --prefix gsnake-web run build`
  - `npm --prefix gsnake-web run check`
  - `npm --prefix gsnake-web test`
  - `cd gsnake-levels && cargo build`
  - `cd gsnake-levels && cargo test`
- [ ] Root repo CI (`.github/workflows/deploy.yml`) continues to work unchanged
- [ ] Document override mechanism in root repo README.md
- [ ] Document in `gsnake-specs/tasks/repo-architecture.md` under new "Local Development Overrides" section

### US-008: Add per-submodule CI workflows compatible with nektos/act
**Description:** As a developer, I want each submodule to have CI that validates standalone builds and works with nektos/act for local testing.

**Acceptance Criteria:**
- [ ] All submodule CI workflows (from US-002 through US-006) are created
- [ ] Each workflow file is named `.github/workflows/ci.yml` for consistency
- [ ] All workflows use standard GitHub Actions that work with nektos/act:
  - `actions/checkout@v4`
  - `actions/setup-node@v4`
  - `dtolnay/rust-toolchain@stable`
  - Avoid: GitHub-specific features like `github-script`, environment files (use GITHUB_OUTPUT)
- [ ] Each workflow has these triggers:
  ```yaml
  on:
    push:
      branches: [main]
    pull_request:
    workflow_dispatch:
  ```
- [ ] Test each workflow locally using nektos/act:
  ```bash
  cd gsnake-core && act -j build && act -j test
  cd gsnake-web && act -j build && act -j typecheck
  # ... etc for each submodule
  ```
- [ ] All local act runs succeed (or document known limitations)
- [ ] All workflows run successfully on GitHub Actions when pushed
- [ ] Document nektos/act usage in root repo README.md: "Testing CI Locally"

### US-009: Update repo-architecture.md with new dependency model
**Description:** As a maintainer, I want the architecture documentation to accurately reflect the git-based dependency model.

**Acceptance Criteria:**
- [ ] Update `gsnake-specs/tasks/repo-architecture.md` to show two dependency modes:
  - Standalone mode (git URLs)
  - Root repo mode (local paths via auto-detection)
- [ ] Update Mermaid diagram to show conditional dependency resolution
- [ ] Add new sections:
  - "Git Branch Dependencies"
  - "Local Development Overrides"
  - "Building Standalone Submodules"
  - "CI/CD Strategy"
- [ ] Include examples of git dependency declarations for Rust and JavaScript
- [ ] Link to each submodule's README.md for detailed build instructions
- [ ] Mark old pain points as resolved: "WEB, LEVELS, EDITOR can now build alone"
- [ ] Document versioning: root repo uses semver, submodules track `main` branch
- [ ] Update "Future goal" section to reflect completion of standalone builds

## Functional Requirements

1. **FR-1:** All submodules must resolve cross-repo dependencies via git branch URLs (targeting `main`) by default
2. **FR-2:** Each submodule must provide build/test commands that succeed with only that repo checked out
3. **FR-3:** Build scripts must auto-detect root repo context (presence of `../.git` and sibling submodules)
4. **FR-4:** When root repo is detected, build scripts must override git dependencies with local paths automatically
5. **FR-5:** Override mechanism must not require manual edits to committed dependency files
6. **FR-6:** Each submodule must have `.github/workflows/ci.yml` validating standalone builds
7. **FR-7:** All CI workflows must be compatible with nektos/act for local testing
8. **FR-8:** Root repo CI must continue to work unchanged with submodules checked out
9. **FR-9:** `gsnake-specs/tasks/repo-architecture.md` must document both standalone and root repo build modes
10. **FR-10:** Each submodule must have a README.md with standalone build instructions

## Non-Goals (Out of Scope)

- Publishing packages to registries (crates.io, npm, PyPI)
- Rewriting core architecture or changing existing APIs
- Introducing monorepo tooling (Nx, Turborepo, etc.)
- Changing root repo versioning strategy (keep existing semver in package.json)
- Supporting Windows-specific path resolution (focus on Unix-like systems first)

## Design Considerations

### Dependency Resolution Strategy

**Git Dependencies (Standalone Mode):**
- Rust: `git = "https://github.com/nntin/gsnake", branch = "main", package = "gsnake-core"`
- JavaScript: `"git+https://github.com/nntin/gsnake.git#main:gsnake-core/engine/bindings/wasm/pkg"`
- Always target `main` branch for simplicity and latest code

**Local Path Overrides (Root Repo Mode):**
- Detect root repo context: check for `../.git` and `../<submodule>/.git`
- Rust: use `.cargo/config.toml` with `[patch]` or environment variable `CARGO_PATCH`
- JavaScript: dynamically rewrite `package.json` in preinstall script or use npm/yarn workspace features
- Overrides are automatic and transparent to developers

### CI Strategy

**Per-Submodule CI:**
- Validates standalone builds (clone only that submodule)
- Runs on every push to main and on pull requests
- Compatible with nektos/act for local validation before pushing
- Fast feedback loop for submodule-specific changes

**Root Repo CI:**
- Validates integration across all submodules
- Uses local submodule paths (existing behavior)
- Runs E2E tests, contract tests, full deployment
- Already exists in `.github/workflows/deploy.yml`

## Technical Considerations

### Git Subdirectory Dependencies

**Challenge:** Git doesn't natively support checking out subdirectories as dependencies

**Solutions:**
- **Rust:** Cargo supports `git` + `package` for workspace members in git repos
- **JavaScript:** npm/yarn support `#subdirectory=path` syntax in git URLs (npm 7+)
- **Alternative:** Publish intermediate build artifacts (WASM pkg) to separate git repo or use git subtree

### WASM Build Artifacts

**Challenge:** gsnake-web needs prebuilt WASM in standalone mode

**Options:**
1. **Commit built WASM to gsnake-core** (increases repo size but simplest for standalone)
2. **Separate git repo for WASM artifacts** (gsnake-core-wasm)
3. **Build WASM on-demand in gsnake-web CI** (slower but avoids committed artifacts)

**Recommendation:** Start with option 1 (commit WASM to gsnake-core), migrate to option 2 if repo size becomes an issue

### Contract Test Fixtures

**Challenge:** gsnake-web needs contract fixtures from gsnake-core

**Solution:**
- Commit fixtures to both repos (small JSON files, acceptable duplication)
- OR: Fetch fixtures via git URL during test setup
- Document in README: "Run contract tests in root repo for full validation"

### nektos/act Compatibility

**Known Limitations:**
- Some GitHub Actions features not supported (e.g., composite actions, some caching)
- Docker-in-Docker can be slow
- Service containers may require additional configuration

**Workarounds:**
- Use simple, standard actions (checkout, setup-node, rust-toolchain)
- Cache node_modules and cargo registry using act's `-bind` flag for local paths
- Document act-specific setup in root README.md

## Success Metrics

- ✅ Each submodule builds successfully when cloned alone (0 errors)
- ✅ Each submodule tests successfully when cloned alone (0 failures, except integration tests requiring external services)
- ✅ Root repo builds successfully with submodules checked out (existing behavior preserved)
- ✅ All submodule CI workflows pass on GitHub Actions
- ✅ All submodule CI workflows pass locally with nektos/act (or documented limitations)
- ✅ Zero remaining `../gsnake-*` path dependencies in committed dependency files
- ✅ All documentation updated to reflect new model
- ✅ Developer feedback: "Standalone builds are intuitive and work as expected"

## Migration Validation Plan

**For Each User Story (US-002 through US-006):**
1. Implement changes in feature branch
2. Test standalone build: clone ONLY that submodule, run build/test
3. Test root repo build: checkout all submodules, run build/test
4. Test CI locally: `cd <submodule> && act`
5. Push to GitHub, verify CI passes
6. Merge to main only after all tests pass
7. Document any issues or limitations in submodule README.md

**Full Integration Test (After All US Complete):**
1. Fresh clone of root repo with all submodules
2. Run full CI pipeline locally: `act -j validate` (from root repo CI)
3. Deploy to staging environment
4. Verify deployed application works correctly
5. Update documentation with final lessons learned

## Open Questions (Resolved)

1. ~~Which tagging scheme for git dependencies?~~
   **Resolved:** Use branch names (`main`) for simplicity and latest code

2. ~~Should local overrides use env vars, scripts, or config files?~~
   **Resolved:** Auto-detection scripts that check for root repo context

3. ~~Do we need per-repo CI?~~
   **Resolved:** Yes, add per-submodule CI compatible with nektos/act

## Implementation Timeline

**Phase 1: Foundation (US-001, US-002)**
- Document strategy
- Migrate gsnake-core (foundation for others)

**Phase 2: Rust Dependencies (US-003)**
- Migrate gsnake-levels (simpler than web)

**Phase 3: JavaScript Dependencies (US-004, US-005)**
- Migrate gsnake-web (most complex)
- Migrate gsnake-editor (depends on web)

**Phase 4: Documentation & Tooling (US-006, US-007, US-008, US-009)**
- Migrate gsnake-specs
- Create override mechanism
- Add CI workflows
- Update architecture docs

**Phase 5: Validation**
- Test all standalone builds
- Test root repo integration
- Verify CI with nektos/act
- Collect developer feedback
