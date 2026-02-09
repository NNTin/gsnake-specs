# PRD: Act-Aware Coverage Pipeline Reliability

## 1. Introduction/Overview

The local CI loop that runs via `scripts/test/test_act.sh` currently reports repeated coverage failures even when coverage execution and thresholds pass. The latest run completed on **February 9, 2026** with `19 passed, 6 failed`, and all failures were coverage jobs:

- `gsnake-levels/.github/workflows/ci.yml::coverage`
- `gsnake-web/.github/workflows/ci.yml::coverage`
- `gsnake-editor/.github/workflows/ci.yml::coverage`
- `.github/workflows/ci.yml::core-coverage`
- `.github/workflows/ci.yml::gsnake-editor-coverage`
- `.github/workflows/ci.yml::gsnake-web-coverage`

Log evidence in `scripts/test/tmp_act_results/` shows coverage/test commands completing successfully, then failing only at artifact upload with:

- `::error::Unable to get the ACTIONS_RUNTIME_TOKEN env variable`

This is an `act` runtime mismatch for `actions/upload-artifact`, not necessarily a real coverage failure. The pipeline must be fixed so local loops produce actionable results, while preserving strict coverage quality gates. Coverage thresholds must not be reduced. If real coverage is below threshold, tests must be added or truly untestable files must be explicitly excluded with justification.

Ralph loop progress history in `scripts/ralph/progress.txt` shows this same `19 passed, 6 failed` pattern repeating across multiple iterations on **February 9, 2026** (`US-002` through `US-013`) even after substantial test additions and local coverage command successes. This indicates the primary blocker is pipeline/runtime handling, not missing coverage work.

## 2. Goals

- Eliminate false-negative coverage failures in local `act` pipeline runs.
- Preserve all current coverage thresholds and blocking behavior for real coverage regressions.
- Make `scripts/test/result.txt` and `tmp_act_results` clearly distinguish runtime/tooling limitations from real quality failures.
- Keep GitHub Actions behavior correct, including artifact upload where supported.
- Ensure loop automation (Ralph loops) converges instead of repeatedly retrying non-actionable failures.
- Preserve the existing package-level coverage-command contract as the source of truth for coverage behavior.

## 3. User Stories

### US-001: Classify `act` artifact-token failures correctly

**Description:** As a maintainer, I want local coverage jobs that only fail on `ACTIONS_RUNTIME_TOKEN` upload errors to be marked as non-blocking runtime limitations so I can focus on real failures.

**Acceptance Criteria:**

- [ ] Local run output detects the signature `Unable to get the ACTIONS_RUNTIME_TOKEN env variable`.
- [ ] A job is classified as runtime-limited only when coverage execution has already succeeded in that same log.
- [ ] Runtime-limited jobs are not reported as full coverage failures in `result.txt`.
- [ ] `result.txt` includes a clear reason label for the classification.
- [ ] Typecheck/lint passes.

### US-002: Keep real coverage failures blocking

**Description:** As a maintainer, I want genuine coverage failures (threshold misses, test failures, command failures) to remain blocking so quality gates stay strict.

**Acceptance Criteria:**

- [ ] Any threshold failure remains `FAILED`.
- [ ] Any test failure during coverage run remains `FAILED`.
- [ ] Any coverage command non-zero exit before artifact upload remains `FAILED`.
- [ ] Failure summaries include actionable root-cause hints (threshold miss, test failure, infra/runtime issue).
- [ ] Typecheck/lint passes.

### US-003: Preserve and enforce current threshold contract

**Description:** As a release owner, I want all existing coverage thresholds preserved so we do not weaken quality standards.

**Acceptance Criteria:**

- [ ] `gsnake-web` thresholds remain at least: `lines 80`, `statements 80`, `functions 30`, `branches 45`.
- [ ] `gsnake-editor` thresholds remain at least: `lines 80`, `statements 60`, `functions 75`, `branches 40`.
- [ ] `gsnake-core` and `gsnake-levels` remain `--fail-under-lines 80`.
- [ ] No threshold values are reduced in workflows, scripts, or test configs.
- [ ] Typecheck/lint passes.

### US-004: Address true low-coverage cases without lowering thresholds

**Description:** As a developer, I want a clear remediation path for real coverage deficits so pipelines pass for valid reasons.

**Acceptance Criteria:**

- [ ] When coverage is below threshold, new tests are added for uncovered behavior first.
- [ ] If code is truly untestable, coverage exclusion is narrowly scoped and documented with rationale.
- [ ] Exclusions do not hide testable business logic.
- [ ] After remediation, coverage jobs pass without threshold reduction.
- [ ] Typecheck/lint passes.

### US-005: Improve `tmp_act_results` diagnostics for loop automation

**Description:** As an automation agent operator, I want `tmp_act_results` to provide machine-actionable diagnostics so loops can decide next actions correctly.

**Acceptance Criteria:**

- [ ] Per-job logs and status files remain available for post-run analysis.
- [ ] Each failed/non-success job can be mapped to one of: `real_failure`, `act_runtime_limitation`, `unknown`.
- [ ] The summary includes exact logfile paths for each non-success job.
- [ ] Unknown failures remain blocking by default.
- [ ] Typecheck/lint passes.

### US-006: Document expected local vs GitHub behavior

**Description:** As a contributor, I want documentation of coverage behavior differences between `act` and GitHub Actions so I can debug efficiently.

**Acceptance Criteria:**

- [ ] Docs explain that `actions/upload-artifact` may fail under `act` due to missing runtime token.
- [ ] Docs explain how to tell if coverage itself passed before upload failed.
- [ ] Docs include commands to rerun individual failed jobs.
- [ ] Docs define when a failure requires code/tests changes versus infra handling.
- [ ] Typecheck/lint passes.

### US-007: Keep package-level coverage commands as source of truth

**Description:** As a maintainer, I want coverage configuration centralized in package scripts/config files so root and submodule workflows cannot drift.

**Acceptance Criteria:**

- [ ] Coverage threshold changes are made in package configs/scripts first (`vitest.config.ts` or `scripts/coverage.sh`), not duplicated inline in workflow steps.
- [ ] Root and standalone/submodule coverage jobs call package-level coverage commands.
- [ ] `gsnake-core/scripts/coverage.sh` keeps package-scoped coverage gating (`--package gsnake-core`) to avoid threshold drift from unrelated targets.
- [ ] Typecheck/lint passes.

## 4. Functional Requirements

- FR-1: The pipeline tooling must detect `ACTIONS_RUNTIME_TOKEN` upload failures from coverage job logs.
- FR-2: The pipeline tooling must distinguish upload-step-only failures from coverage execution failures.
- FR-3: Local `act` summaries must mark upload-token-only failures as runtime limitations, not true coverage regressions.
- FR-4: True coverage failures (threshold/test/command) must remain blocking.
- FR-5: Current coverage thresholds must remain unchanged or stricter.
- FR-6: Threshold values must be explicitly validated in CI configuration and test config files.
- FR-7: If thresholds are not met, remediation must be adding tests or justified file exclusions.
- FR-8: File exclusions must be minimal, explicit, and documented with reason.
- FR-9: Unknown classification outcomes must default to blocking failure.
- FR-10: `result.txt` must include categorized status and root-cause summary per failed/non-success job.
- FR-11: `tmp_act_results` must preserve the raw logs needed to validate classification decisions.
- FR-12: GitHub-hosted CI must continue to run coverage and artifact upload behavior without local `act` workaround regressions.
- FR-13: Coverage behavior and thresholds must be owned by package-level commands/config (`vitest.config.ts`, `scripts/coverage.sh`) and not duplicated with conflicting inline workflow flags.
- FR-14: Root and submodule workflows must invoke package coverage commands so local and CI behavior stays aligned.
- FR-15: `gsnake-core` coverage gating must remain package-scoped (`cargo llvm-cov --package gsnake-core`) unless explicitly redesigned in a separate effort.

## 5. Non-Goals (Out of Scope)

- Lowering any existing coverage threshold.
- Disabling coverage jobs to make local runs green.
- Treating unknown failures as success.
- Replacing coverage tooling (`vitest`, `cargo llvm-cov`) in this milestone.
- Broad CI redesign unrelated to coverage-job false negatives.

## 6. Design Considerations

- Prefer explicit status categories in local summaries instead of binary success/failure when diagnosing `act` limitations.
- Keep runtime-limitation handling isolated to local/`act` behavior; avoid weakening GitHub CI semantics.
- Surface evidence in human-readable form (`result.txt`) and automation-friendly form (stable status labels).

## 7. Technical Considerations

- `act` frequently lacks `ACTIONS_RUNTIME_TOKEN` required by `actions/upload-artifact@v4`.
- Coverage logs currently show successful test execution and generated coverage reports before upload fails.
- The repository has both root and submodule workflows with coverage jobs; behavior must be consistent across all six affected jobs.
- The existing `scripts/test/test_act.sh` workflow aggregation and `tmp_act_results` artifacts are central integration points for loop automation.
- Progress evidence shows repeated loop checkpoints with unchanged `19 passed, 6 failed` outcomes while functional/test/typecheck jobs remained green, supporting an infra-classification issue rather than broad missing test coverage.

## 8. Success Metrics

- Local `act` runs no longer report false coverage failures when coverage and thresholds pass.
- Real coverage regressions still fail locally and in GitHub CI.
- Coverage threshold values remain unchanged across all packages.
- Ralph loop retries due solely to upload-token false failures are reduced to zero.
- Latest baseline (`19 passed, 6 failed` on February 9, 2026) improves so remaining failures are only actionable engineering failures.

## 9. Open Questions

- Should the primary fix be workflow-level (skip/non-blocking artifact upload under `act`) or result-classification-level in `test_act.sh`, or both?
- Do we want a third explicit summary status label (for example `RUNTIME_LIMITATION`) in addition to `SUCCESS` and `FAILED`?
- What approval policy should govern adding new coverage exclusions for untestable files?
