# PRD: CI Test Workflows Across gSnake Repos

## Introduction

We need GitHub Actions workflows in each gSnake repo to run tests, add missing tests where they do not exist, fix any resulting failures, and ensure all tests can be run locally with `nektos/act`. This covers the root `gsnake` repo and the submodules `gsnake-levels`, `gsnake-specs`, and `gsnake-web` (ignore `gsnake-python`). The root repo contains `gsnake-core` and `gsnake-editor` as internal projects, and their tests must be run by the root workflow.

## Goals

- Provide a per-repo GitHub Actions workflow that runs that repo’s tests on `ubuntu-latest`.
- Add missing test suites for repos that currently lack tests (`gsnake-levels`, `gsnake-editor`, `gsnake-specs`).
- Ensure all workflows run successfully locally via `act` without extra secrets or manual setup.
- Fix test failures uncovered by the new workflows to reach consistent “green” CI.

## User Stories

### US-001: Per-repo test workflows
**Description:** As a contributor, I want each repo to have a test workflow so that CI validates changes at the repo boundary.

**Acceptance Criteria:**
- [ ] Each git repo (`gsnake`, `gsnake-levels`, `gsnake-web`, `gsnake-specs`) includes a `.github/workflows/test.yml` that runs on `push` and `pull_request`.
- [ ] All workflows use only `ubuntu-latest`.
- [ ] All workflows use only public actions and do not require secrets.
- [ ] Each workflow can be run locally using `act` without additional configuration.
- [ ] Root `gsnake` workflow includes test steps for `gsnake-core` and `gsnake-editor` (both live in the root repo).

### US-002: gsnake-core test coverage
**Description:** As a maintainer, I want Rust unit and contract tests to run in CI so core logic remains stable.

**Acceptance Criteria:**
- [ ] Root workflow runs `cargo test` in `gsnake-core` (workspace root at `gsnake-core/Cargo.toml`).
- [ ] Root workflow runs `cargo test contract_tests` in `gsnake-core/engine/core`.
- [ ] Root workflow runs `cargo test generate_contract_fixtures` in `gsnake-core/engine/core` to ensure fixtures are up to date.
- [ ] Root workflow installs Rust toolchain using `dtolnay/rust-toolchain@stable`.
- [ ] Rust tests pass without requiring network access beyond crates.io.
- [ ] Any failing tests are fixed until the workflow is green.

### US-003: gsnake-levels tests
**Description:** As a maintainer, I want tests for the `gsnake-levels` CLI and core logic so level parsing and verification remain correct.

**Acceptance Criteria:**
- [ ] Add Rust unit tests for `resolve_playback_path` and `load_playback_directions`.
- [ ] Add an integration test that verifies `verify_level` succeeds for `levels/easy/level_001.json` and `playbacks/easy/level_001.json`.
- [ ] Tests live in `gsnake-levels/tests/` and use repo fixtures from `levels/` and `playbacks/`.
- [ ] Tests do not mutate files under `levels/` or `playbacks/`.
- [ ] `gsnake-levels/.github/workflows/test.yml` runs `cargo test`.
- [ ] The workflow installs Rust toolchain with `dtolnay/rust-toolchain@stable`.
- [ ] The workflow checks out `gsnake-levels` into `gsnake-levels/` and `NNTin/gSnake` into `gsnake-root/` (both inside the workspace).
- [ ] The workflow creates a workspace-root symlink `gsnake-core` → `gsnake-root/gsnake-core`.
- [ ] The workflow runs `cargo test` with `working-directory: gsnake-levels` so `../gsnake-core` resolves to the symlink.
- [ ] Tests pass locally and in CI.

### US-004: gsnake-web tests
**Description:** As a maintainer, I want web app unit tests to run in CI so contract and UI logic stays correct.

**Acceptance Criteria:**
- [ ] `gsnake-web/.github/workflows/test.yml` runs `npm ci` and `npm test`.
- [ ] Workflow installs Node 20 and runs commands with `working-directory: gsnake-web`.
- [ ] Workflow installs Rust toolchain with `dtolnay/rust-toolchain@stable` and installs `wasm-pack`.
- [ ] The workflow checks out `gsnake-web` into `gsnake-web/` and `NNTin/gSnake` into `gsnake-root/` (both inside the workspace).
- [ ] The workflow creates workspace-root symlinks:
  - [ ] `gsnake-core` → `gsnake-root/gsnake-core`
  - [ ] `scripts` → `gsnake-root/scripts`
- [ ] The workflow runs `python3 ../scripts/build_wasm.py` from `working-directory: gsnake-web` to create `../gsnake-core/engine/bindings/wasm/pkg` before `npm ci`.
- [ ] Tests pass locally and in CI.

### US-005: gsnake-editor tests
**Description:** As a maintainer, I want tests for the level editor so UI and server logic are verified.

**Acceptance Criteria:**
- [ ] Add `vitest`, `@testing-library/svelte`, `@testing-library/jest-dom`, and `jsdom` as dev dependencies in `gsnake-editor`.
- [ ] Add `test` script to `gsnake-editor/package.json` that runs `vitest run`.
- [ ] Add `vitest.config.ts` in `gsnake-editor` with default `environment: 'jsdom'`.
- [ ] Add `// @vitest-environment node` pragma to server tests that must run in Node.
- [ ] Add a UI test that renders `LandingPage.svelte` (or `GridSizeModal.svelte`) and asserts on a visible label or button.
- [ ] Refactor `server.ts` to export the Express `app` so it can be tested without starting a real server.
- [ ] Add a server test using `supertest` that:
  - [ ] `POST /api/test-level` with JSON returns `{ success: true }`.
  - [ ] `GET /api/test-level` returns the stored payload.
  - [ ] `GET /api/test-level` returns 404 when nothing is stored.
- [ ] Root workflow runs `npm --prefix gsnake-editor test`.

### US-006: gsnake-specs tests
**Description:** As a maintainer, I want validation of spec content so docs don’t regress.

**Acceptance Criteria:**
- [ ] Add `gsnake-specs/package.json` with scripts:
  - [ ] `lint:md` → `markdownlint-cli2 \"**/*.md\" \"!**/node_modules/**\"`
  - [ ] `check:links` → `markdown-link-check -c .mlc.json \"**/*.md\"`
- [ ] Add `markdownlint-cli2` and `markdown-link-check` as dev dependencies in `gsnake-specs`.
- [ ] Add `.mlc.json` in `gsnake-specs` that ignores external HTTP/HTTPS links to avoid CI flakiness.
- [ ] `gsnake-specs/.github/workflows/test.yml` runs `npm ci`, `npm run lint:md`, and `npm run check:links`.
- [ ] Tests pass locally and in CI.

### US-007: gsnake root E2E tests
**Description:** As a maintainer, I want end-to-end tests to run in CI to ensure the product still works as a whole.

**Acceptance Criteria:**
- [ ] `gsnake/.github/workflows/test.yml` runs `npm ci`, `npx playwright install --with-deps chromium`, and `npm run test:e2e`.
- [ ] The workflow uses `actions/checkout` with `submodules: recursive` so `gsnake-web` and `gsnake-levels` are available.
- [ ] The workflow uses `actions/setup-node@v4` with `node-version: 20`.
- [ ] The workflow installs Rust toolchain with `dtolnay/rust-toolchain@stable` and installs `wasm-pack` so `scripts/build_fixture_wasm.sh` can run.
- [ ] Playwright installs successfully in GitHub Actions and with `act`.

### US-008: act compatibility
**Description:** As a contributor, I want to run CI locally with `act` to debug without pushing to GitHub.

**Acceptance Criteria:**
- [ ] All workflows avoid actions that require GitHub-hosted tokens or privileged services.
- [ ] All workflows pass when run locally using:
  - [ ] `act -P ubuntu-latest=catthehacker/ubuntu:act-latest -W .github/workflows/test.yml`
- [ ] No workflow step assumes `GITHUB_TOKEN` permissions beyond default read.

## Functional Requirements

- FR-1: Each git repo includes `.github/workflows/test.yml` that runs on `push` and `pull_request`.
- FR-2: All workflows run on `ubuntu-latest` only.
- FR-3: Root workflow runs `cargo test` in `gsnake-core` and `npm --prefix gsnake-editor test`.
- FR-4: `gsnake-levels` workflow runs `cargo test` after adding new unit tests and sets up a workspace-root `gsnake-core` symlink to `gsnake-root/gsnake-core`.
- FR-5: `gsnake-web` workflow runs `npm ci` and `npm test` with `working-directory: gsnake-web`, sets up workspace-root `gsnake-core` and `scripts` symlinks to `gsnake-root/*`, and runs `python3 ../scripts/build_wasm.py` before `npm ci`.
- FR-6: `gsnake-specs` workflow runs `npm ci`, `npm run lint:md`, and `npm run check:links`.
- FR-7: Root workflow runs Playwright tests using `npm run test:e2e`.
- FR-8: All workflows must be runnable via `act` without extra secrets.

## Non-Goals

- Adding new product features unrelated to test coverage.
- Adding cross-platform (Windows/macOS) CI.
- Full performance benchmarking.

## Design Considerations (Optional)

- Keep workflows minimal and consistent across repos for ease of maintenance.
- Use cache only if it doesn’t break `act` runs.

## Technical Considerations (Optional)

- Playwright needs `npx playwright install --with-deps chromium` on CI and `act`.
- Rust tests use `dtolnay/rust-toolchain@stable` with `wasm32-unknown-unknown` target when building wasm.
- `gsnake-web` depends on a local `file:` package at `../gsnake-core/engine/bindings/wasm/pkg`.
- `gsnake-web` unit tests read fixtures from `../gsnake-core/engine/core/tests/fixtures`.
- Python 3 is required for `scripts/build_wasm.py` when `gsnake-web` tests need fresh wasm artifacts.
- `gsnake-levels` and `gsnake-web` workflows must check out `NNTin/gSnake` into `gsnake-root/` and create workspace-root symlinks (`gsnake-core`, `scripts`).

## Test Command Options + Recommendation

**Options for running tests per repo**

- **gsnake (root, E2E):** `npm ci` → `npx playwright install --with-deps chromium` → `npm run test:e2e`
- **gsnake-core (in root workflow):** `cargo test` → `cargo test contract_tests` → `cargo test generate_contract_fixtures`
- **gsnake-levels (Rust CLI):** `cargo test` (after adding unit tests)
- **gsnake-web (Svelte/Vitest):** `npm ci` → `npm test` (optionally `npm run check` for TS validation)
- **gsnake-editor (Svelte app + server):** `npm ci` → `npm test` (after adding Vitest + Svelte Testing Library)
- **gsnake-specs (Markdown docs):** `npm ci` → `npm run lint:md` → `npm run check:links`

**Recommended default set**

- Use the commands above as the default for each repo’s CI workflow.
- Keep `gsnake` root E2E tests in a dedicated workflow to avoid slowing down unit-test workflows.
- Add a small but real test suite to `gsnake-levels`, `gsnake-editor`, and `gsnake-specs` so CI covers meaningful behavior.

## Success Metrics

- All test workflows are green on `main`.
- `act` can run each repo’s workflow successfully without manual secrets.
- New tests exist in repos that previously had none.

## Open Questions

None. All tooling, commands, and expectations are specified above.
