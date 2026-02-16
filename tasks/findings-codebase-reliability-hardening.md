# gSnake Codebase Critique & Findings (Open Items)

**Original Date:** 2026-02-15\
**Revalidated:** 2026-02-16\
**Scope:** Entire gSnake repository (excluding `gsnake-specs`)

## High Priority Open Issues

### 1. Code Duplication in Contract Validation

**Locations:**

- `contracts/level-definition.schema.json` (canonical schema)
- `gsnake-web/packages/gsnake-web-app/contracts/levelDefinitionGuard.ts` (manual TS guard)
- `e2e/level-definition.ts` (separate E2E guard)

**Impact:** Contract changes still require multiple implementations to stay aligned; drift risk remains (mitigated but not eliminated by schema-drift tests).

**Recommendations:**

1. Generate TypeScript guards from the canonical schema (instead of maintaining hand-written guards in multiple places)
1. Reuse one shared runtime guard between app and E2E
1. Keep schema-drift tests as backstop

### 2. Excessive `unwrap()` / `expect()` Usage in Rust Tooling

**Locations:**

- `gsnake-levels/src/*`
- `gsnake-core/engine/bindings/cli/src/*`

**Current status:** `rg` still finds many `unwrap()/expect()` sites across tooling/test code paths.

**Impact:**

- Tooling can panic instead of returning actionable errors
- Batch workflows are harder to debug and recover

**Recommendations:**

1. Replace runtime-path `unwrap()`/`expect()` with `?` and contextual errors
1. Aggregate validation/reporting errors where possible
1. Keep panic-style assertions only in tests where appropriate

______________________________________________________________________

## Medium Priority Open Issues

### 3. Solver Performance Bottlenecks

**Locations:**

- `gsnake-levels/src/playback_generator.rs`
- `gsnake-levels/src/bin/solve_level.rs`

**Issues:**

- Playback generation spawns a separate `cargo run --bin solve_level` process per level
- BFS solver clones full engine state per branch
- No cross-level process reuse/state caching

**Recommendations:**

1. Batch solving in a long-lived solver process
1. Profile clone-heavy hotspots before optimization
1. Add parallelism after baseline profiling

### 4. WASM Module Load Time

**Location:** `gsnake-web/packages/gsnake-web-app/engine/WasmGameEngine.ts`

**Issue:** Initialization waits on WASM module startup before game is ready, which can delay first interactive render.

**Recommendations:**

1. Evaluate lazy/background initialization patterns
1. Surface explicit loading state/progress in UI

### 5. Build-Script Coupling to Filesystem State

**Location:** `gsnake-levels/build.rs`

**Issue:** Build script rewrites `.cargo/config.toml` based on detected repo layout.

**Impact:** Hidden environment coupling; mode switching can be brittle.

**Recommendations:**

1. Document this behavior prominently
1. Prefer explicit Cargo/workspace override mechanisms where feasible

### 6. Gravity/Stone Mechanics Complexity

**Location:** `gsnake-core/engine/core/src/gravity.rs`

**Issue:** Logic remains complex and order-dependent despite improved comments.

**Recommendations:**

1. Add broader property-based tests for gravity/collision combinations
1. Consider separating gravity step resolution from collision rules for readability/testability

### 7. Dependency Maintenance Process

**Issue:** Update policy/automation is still not clearly documented for all parts of the stack (including n8n workflows/runtime).

**Recommendations:**

1. Document update cadence and ownership
1. Add routine security audit checks (`npm audit`, `cargo audit`) in automation where practical

______________________________________________________________________

## Low Priority Open Issues

### 8. CORS Configuration (Dev Server)

**Location:** `gsnake-editor/server.ts`

**Issue:** Allows all `localhost:*` origins.

**Recommendations:**

1. Restrict to explicit dev ports
1. Document local-only trust assumptions clearly

### 9. Environment Variable Access in n8n Code Nodes

**Issue:** Workflow setup/documentation still includes `$env`-based secret access patterns.

**Recommendations:**

1. Prefer n8n credential storage where possible
1. Audit code nodes for env usage

### 10. CI Server Startup and Process Management Robustness

**Location:** `.github/workflows/ci.yml`

**Issue:** E2E setup still relies on `sleep + curl retry` startup checks and aggressive `pkill/fuser` cleanup.

**Recommendations:**

1. Add explicit health endpoints/readiness checks
1. Replace broad process cleanup with tighter lifecycle management
