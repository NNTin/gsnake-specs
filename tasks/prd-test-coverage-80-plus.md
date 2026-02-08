# PRD: Test Coverage 80+ Across All Packages

## 1. Introduction/Overview

This feature raises automated test coverage to `80%+` for each major package in the gSnake codebase: `gsnake-web`, `gsnake-editor`, `gsnake-core`, and `gsnake-levels`.

The problem it solves is inconsistent quality confidence across packages, where uncovered behavior can hide regressions and bugs. This PRD defines a single-milestone plan to reach `80%+` coverage per package, enforce blocking CI gates immediately, include E2E quality checks, and allow source-code changes when tests expose defects or testability problems.

## 2. Goals

- Reach and sustain `>=80%` **line coverage** in each package:
- `gsnake-web`
- `gsnake-editor`
- `gsnake-core`
- `gsnake-levels`
- Make coverage gates blocking on pull requests and main-branch CI from day one.
- Expand automated E2E verification for critical gameplay and editor workflows.
- Fix production bugs discovered while writing/expanding tests.
- Keep tests deterministic and reliable in CI (low flake rate, stable execution).

## 3. User Stories

### US-001: Define package-level coverage contract

**Description:** As a maintainer, I want a single explicit coverage rule per package so that CI results are unambiguous.

**Acceptance Criteria:**

- [ ] Coverage metric for the `80+` goal is defined as **line coverage** for each package.
- [ ] Coverage command per package is documented and executable locally.
- [ ] CI checks report each package’s coverage percentage independently.
- [ ] Typecheck/lint passes.

### US-002: Raise `gsnake-web` non-UI logic coverage

**Description:** As a developer, I want robust tests around web engine/store logic so that gameplay state handling is safe to refactor.

**Acceptance Criteria:**

- [ ] Add or expand tests for `WasmGameEngine`, keyboard input handling, completion tracking, and stores behavior.
- [ ] Error paths (initialization, invalid states, contract error mapping) are covered.
- [ ] `gsnake-web` line coverage reaches `>=80%`.
- [ ] Typecheck/lint passes.

### US-003: Raise `gsnake-web` UI component behavior coverage

**Description:** As a player, I want UI gameplay flows to remain stable so that level loading, progression, and overlays work after changes.

**Acceptance Criteria:**

- [ ] Add component-level tests for app/game containers, level selection/loading states, and modal/banners.
- [ ] Include tests for user-visible error and completion states.
- [ ] UI assertions validate rendered output and interactions, not just function calls.
- [ ] `gsnake-web` line coverage remains `>=80%`.
- [ ] Typecheck/lint passes.
- [ ] Verify in browser using dev-browser skill.

### US-004: Raise `gsnake-editor` service/model coverage

**Description:** As a developer, I want editor server and model logic fully exercised so that payload handling and APIs are safe.

**Acceptance Criteria:**

- [ ] Expand tests for server routes, validation paths, and error handling.
- [ ] Add tests for level schema/type conversions and ID handling paths.
- [ ] `gsnake-editor` line coverage reaches `>=80%`.
- [ ] Typecheck/lint passes.

### US-005: Raise `gsnake-editor` UI workflow coverage

**Description:** As a level designer, I want key editor workflows to be protected by tests so that editing, saving, and loading do not regress.

**Acceptance Criteria:**

- [ ] Add component/integration tests for `EditorLayout`, `GridCanvas`, and save/load workflows.
- [ ] Cover undo/redo and entity placement edge cases.
- [ ] UI assertions verify actual state transitions in rendered components.
- [ ] `gsnake-editor` line coverage remains `>=80%`.
- [ ] Typecheck/lint passes.
- [ ] Verify in browser using dev-browser skill.

### US-006: Raise `gsnake-core` engine/bindings coverage

**Description:** As a gameplay engineer, I want engine and bindings behavior covered so that game rules and interfaces stay correct.

**Acceptance Criteria:**

- [ ] Expand tests across engine edge cases, serialization contracts, and gravity/stone mechanics branches.
- [ ] Add targeted tests for wasm/cli binding behavior where currently uncovered.
- [ ] Any bug discovered via tests is fixed with a regression test.
- [ ] `gsnake-core` line coverage reaches `>=80%`.
- [ ] Clippy/tests pass.

### US-007: Raise `gsnake-levels` tooling coverage

**Description:** As a content pipeline maintainer, I want generation/sync/verification logic covered so metadata and playbacks remain correct.

**Acceptance Criteria:**

- [ ] Expand tests for generation, metadata sync, playback verification, and failure paths.
- [ ] Add tests for command/path edge cases that were previously untested.
- [ ] Any bug discovered via tests is fixed with a regression test.
- [ ] `gsnake-levels` line coverage reaches `>=80%`.
- [ ] Clippy/tests pass.

### US-008: Expand E2E quality gate coverage

**Description:** As a release owner, I want E2E scenarios to validate core user journeys so line coverage increases do not miss integration failures.

**Acceptance Criteria:**

- [ ] E2E suite covers critical flows: play level, level completion progression, editor create/save/load, and contract-sensitive flows.
- [ ] E2E is required in CI and blocks merges on failure.
- [ ] E2E tests are deterministic under CI environment setup.
- [ ] Typecheck/lint passes.
- [ ] Verify in browser using dev-browser skill.

### US-009: Enforce blocking CI thresholds immediately

**Description:** As a maintainer, I want immediate blocking thresholds so that coverage cannot regress below target.

**Acceptance Criteria:**

- [ ] CI fails if any package drops below `80%` line coverage.
- [ ] CI fails if any required E2E suite fails.
- [ ] Coverage artifacts are uploaded for debugging failed jobs.
- [ ] Threshold configuration is version-controlled and visible in repo.

## 4. Functional Requirements

- FR-1: The system must enforce `>=80%` **line coverage** for each package (`gsnake-web`, `gsnake-editor`, `gsnake-core`, `gsnake-levels`) independently.
- FR-2: Coverage enforcement must be blocking for PR and main CI from initial rollout (no grace period).
- FR-3: `gsnake-web` coverage must be measured through `vitest` coverage output.
- FR-4: `gsnake-editor` coverage must be measured through `vitest` coverage output.
- FR-5: `gsnake-core` coverage must be measured through `cargo llvm-cov`.
- FR-6: `gsnake-levels` coverage must be measured through `cargo llvm-cov`.
- FR-7: Coverage jobs must upload artifacts for failed/successful runs to aid diagnosis.
- FR-8: The E2E suite must be a required CI gate and must pass before merge.
- FR-9: E2E scenarios must include gameplay and editor end-to-end paths.
- FR-10: If new tests reveal source bugs, production code must be corrected in the same change set with regression tests.
- FR-11: Test fixtures and mocks must be deterministic and CI-safe.
- FR-12: Existing contract compatibility checks must continue to pass.
- FR-13: New/updated tests must run via package-native commands documented in each package.
- FR-14: Any threshold change must be explicit in committed configuration files.

## 5. Non-Goals (Out of Scope)

- Achieving `100%` coverage.
- Introducing mutation testing in this milestone.
- Rewriting architecture solely for coverage gains where no defect/testability value exists.
- Replacing existing test frameworks with new frameworks.
- Expanding non-critical UI polish unrelated to coverage/testability.

## 6. Design Considerations

- Prefer behavior-driven tests (observable outcomes) over implementation-coupled tests.
- Reuse existing fixtures and contract test assets where possible.
- Keep UI tests focused on critical workflows and state transitions.

## 7. Technical Considerations

- The repo uses multiple packages/submodules with separate workflows; enforcement must be consistent across root and package CI.
- Rust and Node coverage tooling differ; thresholds must be comparable by line coverage semantics.
- E2E setup includes wasm/editor/web startup dependencies; tests must avoid timing flakiness.
- Broad refactors are allowed when they materially improve testability or fix discovered bugs.

## 8. Success Metrics

- Each package reports `>=80%` line coverage in CI:
- `gsnake-web >=80`
- `gsnake-editor >=80`
- `gsnake-core >=80`
- `gsnake-levels >=80`
- E2E required suite passes on CI for PRs and merges.
- No open high-severity bugs introduced by coverage-related refactors.
- Coverage regressions are blocked automatically by CI threshold checks.

## 9. Open Questions

- Should branch coverage also have a mandatory floor (for example `>=70%`) in this milestone, or remain informational?
- Should E2E coverage be tracked as scenario pass-rate only, or instrumented for UI code coverage contribution in a future phase?
- Should threshold exceptions ever be allowed for generated/binding files, and if yes, under what review policy?
