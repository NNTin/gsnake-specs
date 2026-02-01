# PRD: Pre-Commit Hooks for gSnake Submodules

## Introduction

Add opt-in, repository-visible pre-commit hooks for each gSnake submodule so maintainers can quickly catch formatting, linting, typecheck, and build issues before committing. Hooks must be stored under `.github/hooks` and activated via `git config core.hooksPath .github/hooks`. Hooks must not run unit or e2e tests.

## Goals

- Provide a consistent, fast pre-commit safety net across submodules.
- Ensure hooks are visible in-repo and easy for maintainers to activate.
- Run format, lint, typecheck, and build steps without running tests.
- Keep hooks explicit about what they do and how to enable them.

## User Stories

### US-001: Enable shared hooks per repo

**Description:** As a maintainer, I want to enable shared hooks from `.github/hooks` so I can run consistent checks before committing.

**Acceptance Criteria:**

- [ ] `.github/hooks/README.md` explains how to enable hooks with `git config core.hooksPath .github/hooks`
- [ ] README includes verification and disable steps
- [ ] Typecheck/lint passes

### US-002: Pre-commit hook for gsnake-levels

**Description:** As a maintainer, I want a pre-commit hook for `gsnake-levels` that runs format, lint, typecheck, and build without tests.

**Acceptance Criteria:**

- [ ] Hook runs `cargo fmt --all -- --check`
- [ ] Hook runs `cargo clippy --all-targets -- -D warnings`
- [ ] Hook runs `cargo check`
- [ ] Hook runs `cargo build`
- [ ] Hook does not run `cargo test`
- [ ] Typecheck/lint passes

### US-003: Pre-commit hook for gsnake-web

**Description:** As a maintainer, I want a pre-commit hook for `gsnake-web` that runs format, lint, typecheck, and build without tests.

**Acceptance Criteria:**

- [ ] Hook runs formatter (e.g., Prettier) and linter (e.g., ESLint)
- [ ] Hook runs `npm run check`
- [ ] Hook runs a build command that does not trigger tests (e.g., `npm run build:hook`)
- [ ] Hook does not run `npm test`, `vitest`, or Playwright
- [ ] Typecheck/lint passes

### US-004: Pre-commit hook for gsnake-specs

**Description:** As a maintainer, I want a pre-commit hook for `gsnake-specs` that formats and lints Markdown using Python tooling.

**Acceptance Criteria:**

- [ ] Python tooling exists (`pyproject.toml`) for `mdformat` and `markdownlint` (or equivalent)
- [ ] Hook runs formatter and linter on Markdown files
- [ ] Hook includes a no-op typecheck/build step if not applicable
- [ ] Hook does not run tests
- [ ] Typecheck/lint passes

### US-005: Document hook scope and exclusions

**Description:** As a maintainer, I want clarity on which checks are run and that tests are excluded.

**Acceptance Criteria:**

- [ ] Hook scripts include brief comments stating they avoid tests
- [ ] Documentation lists the exact commands run for each submodule
- [ ] Typecheck/lint passes

## Functional Requirements

- FR-1: Each submodule provides a pre-commit hook stored under `.github/hooks` in its repo.
- FR-2: Hooks must run format, lint, typecheck, and build steps only.
- FR-3: Hooks must not run unit tests or e2e tests.
- FR-4: Hooks must be opt-in via `git config core.hooksPath .github/hooks`.
- FR-5: Hooks should be fast enough for local iteration and fail clearly on errors.

## Non-Goals (Out of Scope)

- No CI enforcement changes in this scope.
- No unit or e2e tests in pre-commit hooks.
- No changes to `gsnake-python` hooks in this iteration (submodule not initialized).

## Design Considerations

- Keep hooks as simple shell scripts with clear output and early exits on failure.
- Provide a single README in `.github/hooks` documenting enablement and commands.

## Technical Considerations

- `gsnake-web` currently runs tests via `prebuild`; add a hook-safe build script to avoid test execution.
- `gsnake-specs` will use Python-based Markdown tooling; standardize via `pyproject.toml`.
- Hooks should be compatible with macOS/Linux; avoid non-portable shell features.

## Success Metrics

- Maintainers can enable hooks in under 1 minute.
- Hook runtime under 2 minutes on a typical dev machine.
- At least one formatting or lint error is caught in pre-commit during initial rollout.

## Open Questions

- Do we want a shared root-level doc listing per-submodule commands in addition to per-repo `.github/hooks/README.md`?
- Should we provide a helper script to set `core.hooksPath` automatically?
