# PRD: Fix CI Pipeline Errors

## Introduction

The gSnake project has 7 failing CI workflow jobs when tested with nektos/act. These failures prevent reliable continuous integration and must be resolved to ensure code quality and deployment reliability. The project uses a multi-repository architecture with workflows in both the root repository and submodule repositories (gsnake-core, gsnake-web, gsnake-editor, gsnake-levels, gsnake-specs).

## Goals

- Fix all 7 failing CI workflow jobs identified in scripts/test/result.txt
- Ensure all workflows pass when tested locally with nektos/act
- Maintain compatibility with GitHub Actions
- Prioritize TypeScript errors, then Cargo.toml path issues, then E2E test failures
- Achieve 100% passing test suite (20 jobs total)

## User Stories

### US-001: Fix TypeScript type errors in gsnake-web detect-local-deps.js
**Description:** As a developer, I need the gsnake-web typecheck job to pass so that TypeScript validation succeeds in CI.

**Current Error:**
- Job: `gsnake-web/.github/workflows/ci.yml::typecheck`
- Error: Parameters 'targetDependency' and 'label' in scripts/detect-local-deps.js have implicit 'any' type
- File: gsnake-web/scripts/detect-local-deps.js:77

**Acceptance Criteria:**
- [ ] Add explicit type annotations to parameters in detect-local-deps.js
- [ ] Verify typecheck passes locally: `act -W gsnake-web/.github/workflows/ci.yml -j typecheck --container-architecture linux/amd64`
- [ ] Verify no new TypeScript errors introduced
- [ ] Update scripts/test/result.txt shows SUCCESS for gsnake-web/.github/workflows/ci.yml::typecheck

### US-002: Fix TypeScript type errors in root deploy workflow validation
**Description:** As a developer, I need the root deploy workflow validation to pass so that deployment preparation succeeds.

**Current Error:**
- Job: `.github/workflows/deploy.yml::validate`
- Error: Same TypeScript errors in gsnake-web/scripts/detect-local-deps.js
- Blocks deployment workflow

**Acceptance Criteria:**
- [ ] Same fix as US-001 resolves this issue (shared file)
- [ ] Verify validate job passes: `act -W .github/workflows/deploy.yml -j validate --container-architecture linux/amd64`
- [ ] Update scripts/test/result.txt shows SUCCESS for .github/workflows/deploy.yml::validate

### US-003: Fix deployment workflow after validation passes
**Description:** As a developer, I need the deploy job to succeed so that the application can be deployed.

**Current Error:**
- Job: `.github/workflows/deploy.yml::deploy`
- Likely blocked by validate job failure or has separate issues
- Need to investigate after validate passes

**Acceptance Criteria:**
- [ ] Investigate deploy job failure after validate is fixed
- [ ] Fix any issues preventing deployment
- [ ] Verify deploy job passes: `act -W .github/workflows/deploy.yml -j deploy --container-architecture linux/amd64`
- [ ] Update scripts/test/result.txt shows SUCCESS for .github/workflows/deploy.yml::deploy

### US-004: Fix gsnake-levels test workflow Cargo.toml path issue
**Description:** As a developer, I need the gsnake-levels test job to find Cargo.toml so that Rust tests can run.

**Current Error:**
- Job: `gsnake-levels/.github/workflows/test.yml::test`
- Error: `could not find Cargo.toml in /home/nntin/git/gSnake/gsnake-levels or any parent directory`
- Issue: Workflow working directory likely incorrect

**Acceptance Criteria:**
- [ ] Investigate gsnake-levels/.github/workflows/test.yml working directory configuration
- [ ] Fix path resolution (may need to adjust checkout path or working directory)
- [ ] Ensure test.yml workflow correctly handles submodule structure
- [ ] Verify test job passes: `act -W gsnake-levels/.github/workflows/test.yml -j test --container-architecture linux/amd64`
- [ ] Update scripts/test/result.txt shows SUCCESS for gsnake-levels/.github/workflows/test.yml::test

### US-005: Fix gsnake-web test workflow
**Description:** As a developer, I need the gsnake-web test workflow to pass so that web application tests validate correctly.

**Current Error:**
- Job: `gsnake-web/.github/workflows/test.yml::test`
- Error details unknown (need to check logs)
- May be related to path issues or dependency resolution

**Acceptance Criteria:**
- [ ] Check gsnake-web/.github/workflows/test.yml failure logs in scripts/test/tmp_act_results/
- [ ] Identify root cause of test failure
- [ ] Fix identified issues (likely path or dependency related)
- [ ] Verify test job passes: `act -W gsnake-web/.github/workflows/test.yml -j test --container-architecture linux/amd64`
- [ ] Update scripts/test/result.txt shows SUCCESS for gsnake-web/.github/workflows/test.yml::test

### US-006: Fix gsnake-specs test workflow
**Description:** As a developer, I need the gsnake-specs test workflow to pass so that documentation validation succeeds.

**Current Error:**
- Job: `gsnake-specs/.github/workflows/test.yml::test`
- Error details unknown (need to check logs)
- May be related to markdown validation or structure checks

**Acceptance Criteria:**
- [ ] Check gsnake-specs/.github/workflows/test.yml failure logs in scripts/test/tmp_act_results/
- [ ] Identify root cause of test failure
- [ ] Fix identified issues
- [ ] Verify test job passes: `act -W gsnake-specs/.github/workflows/test.yml -j test --container-architecture linux/amd64`
- [ ] Update scripts/test/result.txt shows SUCCESS for gsnake-specs/.github/workflows/test.yml::test

### US-007: Fix E2E test timeout/failure in root CI workflow
**Description:** As a developer, I need the E2E tests to complete successfully so that integration testing validates the full application.

**Current Error:**
- Job: `.github/workflows/ci.yml::e2e-test`
- Known issue: E2E tests run too long (mentioned in recent commits)
- May timeout or fail due to performance issues

**Acceptance Criteria:**
- [ ] Check .github/workflows/ci.yml e2e-test failure logs in scripts/test/tmp_act_results/
- [ ] Identify why E2E tests fail or timeout
- [ ] Fix timeout issues (adjust timeouts, optimize tests, or fix test logic)
- [ ] Consider test_act output improvements mentioned in commit faddb4b
- [ ] Verify e2e-test job passes: `act -W .github/workflows/ci.yml -j e2e-test --container-architecture linux/amd64`
- [ ] Update scripts/test/result.txt shows SUCCESS for .github/workflows/ci.yml::e2e-test

### US-008: Verify full test suite passes
**Description:** As a developer, I need all 20 CI jobs to pass so that the entire test suite is green.

**Acceptance Criteria:**
- [ ] Run full test suite (note: takes 30+ minutes, coordinate with user first)
- [ ] Verify scripts/test/result.txt shows: "**Total: 20 passed, 0 failed**"
- [ ] Document any known limitations or warnings in nektos/act vs GitHub Actions
- [ ] Ensure all submodule workflows and root workflows pass

## Functional Requirements

- FR-1: TypeScript files in gsnake-web/scripts/ must pass strict type checking
- FR-2: All Cargo.toml files must be found by their respective workflows
- FR-3: Test workflows must correctly resolve paths for submodule repositories
- FR-4: E2E tests must complete within reasonable time limits
- FR-5: All workflows must be compatible with nektos/act local testing
- FR-6: scripts/test/result.txt must accurately reflect test status
- FR-7: Fixes must not break existing passing jobs (13 currently passing)

## Non-Goals

- Improving test execution speed (except where it causes failures)
- Refactoring workflow structure or organization
- Adding new tests or test coverage
- Optimizing CI caching strategies
- Upgrading workflow actions or dependencies
- Changing the test_act.sh script behavior

## Technical Considerations

### Testing with nektos/act
- All fixes must be verified with: `act -W <workflow-path> -j <job-name> --container-architecture linux/amd64`
- Results are stored in scripts/test/tmp_act_results/ with .log and .status files
- Full suite can be run with scripts/test/test_act.sh (but takes 30+ minutes)

### Repository Structure
- Root repository at /home/nntin/git/gSnake
- Submodules: gsnake-core, gsnake-web, gsnake-editor, gsnake-levels, gsnake-specs
- Dual-mode dependency resolution (root repo mode vs standalone mode)
- Workflows use FORCE_GIT_DEPS=true for standalone testing

### Known Context
- Recent commits mention E2E test timeout issues (faddb4b, fbd9a4e)
- TypeScript errors are in detect-local-deps.js parameter type annotations
- Cargo.toml path issues suggest working directory mismatches
- All fixes will be tested between Ralph agent iterations

## Success Metrics

- 0 failed jobs in scripts/test/result.txt (down from current 7)
- 20 passing jobs in scripts/test/result.txt (up from current 13)
- 100% pass rate for CI workflows when tested with nektos/act
- All fixes validated locally before committing

## Open Questions

- Are there differences between act and GitHub Actions that cause false failures?
- Should test.yml workflows be deprecated in favor of ci.yml workflows?
- What is the expected runtime for E2E tests (to set appropriate timeouts)?
- Are there any workflows that should be marked as allowed to fail?

## Workflow Reference

### Current Status (from scripts/test/result.txt)
```
Failed (7):
- .github/workflows/deploy.yml::validate
- .github/workflows/deploy.yml::deploy
- .github/workflows/ci.yml::e2e-test
- gsnake-levels/.github/workflows/test.yml::test
- gsnake-specs/.github/workflows/test.yml::test
- gsnake-web/.github/workflows/test.yml::test
- gsnake-web/.github/workflows/ci.yml::typecheck

Passing (13):
- .github/workflows/ci.yml::build
- .github/workflows/ci.yml::test
- .github/workflows/ci.yml::wasm
- .github/workflows/ci.yml::gsnake-editor-test
- gsnake-editor/.github/workflows/ci.yml::build
- gsnake-editor/.github/workflows/ci.yml::typecheck
- gsnake-editor/.github/workflows/ci.yml::test
- gsnake-levels/.github/workflows/ci.yml::build
- gsnake-levels/.github/workflows/ci.yml::test
- gsnake-specs/.github/workflows/ci.yml::markdown-lint
- gsnake-specs/.github/workflows/ci.yml::link-check
- gsnake-web/.github/workflows/ci.yml::build
- gsnake-web/.github/workflows/ci.yml::test
```

## Implementation Order

1. **Phase 1 - TypeScript Errors** (US-001, US-002)
   - Fix detect-local-deps.js type annotations
   - Verify both gsnake-web typecheck and deploy validate pass

2. **Phase 2 - Cargo Path Issues** (US-004)
   - Fix gsnake-levels test.yml Cargo.toml path resolution
   - Verify cargo test command works correctly

3. **Phase 3 - Unknown Test Failures** (US-005, US-006)
   - Investigate and fix gsnake-web test.yml
   - Investigate and fix gsnake-specs test.yml

4. **Phase 4 - E2E Tests** (US-007)
   - Fix E2E test timeouts or failures
   - Optimize if needed to prevent long runtimes

5. **Phase 5 - Deploy Job** (US-003)
   - Fix deploy job (likely unblocked after validate passes)
   - Verify full deployment workflow

6. **Phase 6 - Final Validation** (US-008)
   - Run full test suite
   - Verify all 20 jobs pass
