# PRD: Fix CI Pipeline Failures for Standalone Submodule Builds

## Introduction

The gSnake project uses a dual-mode dependency resolution system where submodules can be built both standalone (using git dependencies) and within the root repository (using local paths). Currently, all submodule CI workflows are failing when tested with nektos/act due to incorrect checkout handling. This PRD addresses the root cause: Act copying the entire root repository when workflows expect only the submodule content.

**Current Status:** 9 of 14 CI jobs failing (gsnake-web, gsnake-editor, gsnake-levels, gsnake-specs)

## Goals

- Fix all 9 failing CI jobs in submodule workflows
- Ensure CI workflows properly simulate standalone mode (git dependencies only)
- Maintain compatibility with both GitHub Actions and nektos/act
- Verify the dual-mode dependency resolution system works correctly
- Document the proper patterns for standalone submodule CI workflows

## User Stories

### US-001: Fix checkout action for Act compatibility
**Description:** As a developer running Act tests, I need the checkout action to copy only the submodule content so that the workflow runs in true standalone mode.

**Acceptance Criteria:**
- [ ] Update all submodule workflow checkout steps to handle Act's behavior
- [ ] When run via Act from root repo, only submodule files are in working directory
- [ ] Verify `../.git` does NOT exist (standalone mode detection)
- [ ] No nested submodule directories (e.g., no `gsnake-web/gsnake-web/`)
- [ ] All workflows use consistent checkout pattern

### US-002: Fix gsnake-web CI workflows
**Description:** As a developer, I need gsnake-web build/typecheck/test jobs to pass in standalone mode so I can validate changes locally before pushing.

**Acceptance Criteria:**
- [ ] `scripts/detect-local-deps.js` can be found and executed
- [ ] Script correctly detects standalone mode (no `../.git`)
- [ ] Git dependency on gsnake-core WASM is used (not local path)
- [ ] All three jobs pass: build, typecheck, test
- [ ] Verify with: `act -W gsnake-web/.github/workflows/ci.yml -j build`

### US-003: Fix gsnake-editor CI workflows
**Description:** As a developer, I need gsnake-editor build/typecheck/test jobs to pass in standalone mode so I can validate editor changes.

**Acceptance Criteria:**
- [ ] `scripts/detect-local-deps.js` can be found and executed
- [ ] Script correctly detects standalone mode
- [ ] No dependencies on gsnake-web local files (uses HTTP endpoint)
- [ ] All three jobs pass: build, typecheck, test
- [ ] Verify with: `act -W gsnake-editor/.github/workflows/ci.yml -j build`

### US-004: Fix gsnake-levels CI workflows
**Description:** As a developer, I need gsnake-levels build/test jobs to pass in standalone mode so I can validate level management changes.

**Acceptance Criteria:**
- [ ] `Cargo.toml` is found in the working directory
- [ ] Git dependency on gsnake-core is used (not local path patch)
- [ ] Cargo successfully fetches gsnake-core from git
- [ ] Both jobs pass: build, test
- [ ] Verify with: `act -W gsnake-levels/.github/workflows/ci.yml -j build`

### US-005: Fix gsnake-specs validation workflow
**Description:** As a developer, I need gsnake-specs validation to pass so I can verify documentation structure is complete.

**Acceptance Criteria:**
- [ ] All expected directories are found: deployment, e2e-testing, findings, etc.
- [ ] Directory structure validation passes
- [ ] Markdown file count is accurate
- [ ] Validation job passes
- [ ] Verify with: `act -W gsnake-specs/.github/workflows/ci.yml -j validate`

### US-006: Remove standalone detection scripts from CI
**Description:** As a CI engineer, I need to ensure CI always uses git dependencies (standalone mode) without running detection scripts that expect root repo context.

**Acceptance Criteria:**
- [ ] Remove "Configure dependencies for standalone mode" steps from all submodule workflows
- [ ] CI workflows do not call `detect-local-deps.js` or similar scripts
- [ ] Package.json preinstall hooks do not run or are skipped in CI
- [ ] Git dependencies are used directly without any local path detection
- [ ] Document why detection is skipped in CI (always standalone)

### US-007: Document Act testing patterns
**Description:** As a developer, I need clear documentation on how to properly test submodule CI workflows with Act so I can validate changes before pushing.

**Acceptance Criteria:**
- [ ] Update each submodule's CI workflow with comments explaining Act compatibility
- [ ] Add troubleshooting section to repo-architecture.md for Act testing
- [ ] Document the checkout pattern and why it differs from GitHub Actions
- [ ] Provide example commands for testing each submodule
- [ ] Add section to scripts/test/CLAUDE.md explaining the checkout fix

### US-008: Verify all scenarios
**Description:** As a developer, I need to verify that both standalone and root repository modes work correctly after the fixes.

**Acceptance Criteria:**
- [ ] All 14 CI jobs pass when run via Act (5 root + 9 submodule)
- [ ] Standalone builds work: clone individual submodule and build
- [ ] Root repository builds work: local paths used for development
- [ ] Detection scripts correctly identify mode in both scenarios
- [ ] Run full test suite: `scripts/test/test_act.sh`
- [ ] Update result.txt with passing status for all jobs

## Functional Requirements

### FR-1: Checkout Action Pattern
All submodule CI workflows must use this checkout pattern that works with Act:

```yaml
- name: Checkout submodule
  uses: actions/checkout@v4
  with:
    path: ${{ github.event.repository.name }}
```

When this runs in Act from the root repository, a post-checkout step moves only the submodule files to the working directory.

### FR-2: Standalone Mode Enforcement in CI
CI workflows must NOT run detection scripts. Instead:
- Skip preinstall hooks: `npm install --ignore-scripts` or `NPM_CONFIG_IGNORE_SCRIPTS=true`
- OR: Remove the detection step entirely and rely on git dependencies in package.json
- Rust workflows already use git dependencies by default (no build.rs patch in CI)

### FR-3: Act-Specific Workarounds
Add a post-checkout step to handle Act's behavior:

```yaml
- name: Checkout submodule
  uses: actions/checkout@v4
  with:
    path: temp-checkout

- name: Prepare workspace for Act compatibility
  run: |
    # Move only submodule content to working directory
    if [ -d "temp-checkout/<submodule-name>" ]; then
      # Act copied entire root repo, extract submodule
      mv temp-checkout/<submodule-name>/* .
      mv temp-checkout/<submodule-name>/.* . 2>/dev/null || true
    else
      # GitHub Actions checked out submodule correctly
      mv temp-checkout/* .
      mv temp-checkout/.* . 2>/dev/null || true
    fi
    rm -rf temp-checkout
```

### FR-4: Dependency Resolution
- **gsnake-web**: Must use `git+https://github.com/nntin/gsnake.git#main:gsnake-core/engine/bindings/wasm/pkg`
- **gsnake-editor**: Must use `git+https://github.com/nntin/gsnake.git#main:gsnake-web` or HTTP endpoint
- **gsnake-levels**: Must use git dependency in Cargo.toml (no local patch in CI)

### FR-5: Working Directory Validation
All workflows must verify they're in the correct directory:

```yaml
- name: Verify workspace
  run: |
    echo "Current directory: $(pwd)"
    echo "Contents:"
    ls -la
    # Check for expected files
    [ -f "package.json" ] || [ -f "Cargo.toml" ] || exit 1
```

## Non-Goals (Out of Scope)

- **Not changing the root repository CI workflow** - Root repo CI (.github/workflows/ci.yml) already works correctly
- **Not modifying the dual-mode dependency system** - The architecture is sound, only CI workflow implementation needs fixes
- **Not publishing to package registries** - Still using git dependencies (npm/crates.io publishing is future work)
- **Not adding new CI jobs** - Only fixing existing failing jobs
- **Not changing detection script logic** - Scripts work correctly for local development, they just shouldn't run in CI

## Technical Considerations

### Root Cause Analysis

**Issue:** Act's checkout action behaves differently than GitHub Actions when run from a parent repository.

**Act behavior:**
```
actions/checkout@v4 with path: gsnake-web
→ Copies /home/nntin/git/gSnake/* to /home/nntin/git/gSnake/gsnake-web/
→ Creates: gsnake-web/gsnake-web/, gsnake-web/gsnake-core/, etc.
```

**Expected GitHub Actions behavior:**
```
actions/checkout@v4 with path: gsnake-web
→ Checks out gsnake-web repository to gsnake-web/ directory
→ Creates: gsnake-web/package.json, gsnake-web/src/, etc.
```

### Solution Approaches

**Option A: Post-checkout normalization** (Recommended)
- Add step after checkout to detect and fix nested structure
- Minimal changes to workflow logic
- Compatible with both Act and GitHub Actions
- Slightly more verbose workflows

**Option B: Custom action wrapper**
- Create a composite action that handles checkout + normalization
- Reusable across all submodules
- More complex to maintain
- Adds another abstraction layer

**Option C: Remove detection scripts from CI entirely** (Simplest)
- Skip preinstall hooks in npm install
- Rely solely on git dependencies
- Simplest solution but diverges from local dev flow
- Recommended for standalone mode enforcement

**Recommendation:** Use Option C for JS submodules (skip detection) and Option A for any workflows that need special handling.

### Act Compatibility

All workflows must be tested with:
```bash
act --container-architecture linux/amd64 -W <workflow-path> -j <job-name>
```

Known Act limitations:
- Cache actions may fail (acceptable)
- Checkout behavior differs (addressed by this PRD)
- Docker volume management issues (run jobs sequentially)

## Success Metrics

- **Primary:** All 14 CI jobs pass (5 root + 9 submodule)
- **Verification:** `scripts/test/test_act.sh` completes with 0 failures
- **Local dev:** Developers can still build locally with local path detection
- **Standalone:** Each submodule can be cloned and built independently
- **Documentation:** Clear instructions for testing with Act

## Implementation Order

1. **US-006** - Remove detection scripts from CI (simplest fix)
2. **US-002** - Fix gsnake-web workflows (test the solution)
3. **US-003** - Fix gsnake-editor workflows (apply same pattern)
4. **US-004** - Fix gsnake-levels workflows (Rust variant)
5. **US-005** - Fix gsnake-specs validation (pure structure check)
6. **US-001** - Consolidate checkout pattern if needed
7. **US-007** - Document the fixes and patterns
8. **US-008** - Final verification and validation

## Open Questions

1. Should we create a reusable composite action for checkout+normalization?
   - **Answer:** Not yet - wait to see if skipping detection is sufficient

2. Do we need to test in real GitHub Actions or is Act sufficient?
   - **Answer:** Both - Act for rapid iteration, real GHA for final validation

3. Should detection scripts be removed from package.json preinstall entirely?
   - **Answer:** No - keep for local dev, just skip in CI with `--ignore-scripts`

4. How do we ensure developers don't accidentally commit local path dependencies?
   - **Answer:** Add pre-commit hook to verify package.json has git dependencies

## References

- Root cause analysis: scripts/test/result.txt (failure logs)
- Architecture: gsnake-specs/tasks/repo-architecture.md (dual-mode system)
- Act documentation: https://github.com/nektos/act
- Testing guide: scripts/test/CLAUDE.md
