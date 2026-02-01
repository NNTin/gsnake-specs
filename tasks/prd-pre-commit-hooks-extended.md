# PRD: Pre-Commit Hooks for gsnake-editor and Parent gSnake Project

## Introduction

Extend pre-commit hook coverage to include `gsnake-editor` submodule and the parent gSnake project. Hooks follow the same pattern as existing submodules: stored under `.github/hooks`, activated via `git config core.hooksPath .github/hooks`, and run format/lint/typecheck/build without tests. Additionally, provide a helper script to enable hooks across all submodules.

## Goals

- Add pre-commit hook for `gsnake-editor` following established patterns
- Add pre-commit hook for parent gSnake project focusing on gsnake-core integration
- Create helper script to enable hooks in all submodules at once
- Maintain consistency with existing hook implementations
- Keep hooks fast and focused on quick quality checks

## User Stories

### US-001: Pre-commit hook for gsnake-editor

**Description:** As a maintainer, I want a pre-commit hook for `gsnake-editor` that runs format, lint, typecheck, and build without tests.

**Acceptance Criteria:**

- [ ] `.github/hooks/README.md` exists in gsnake-editor explaining how to enable hooks
- [ ] Hook runs formatter (Prettier or equivalent)
- [ ] Hook runs linter (ESLint or equivalent)
- [ ] Hook runs `npm run check` for typecheck
- [ ] Hook runs `npm run build` without triggering tests
- [ ] Hook does not run `npm test`, `vitest`, or test-related commands
- [ ] Format/lint scripts added to package.json if not present
- [ ] Typecheck/lint passes

### US-002: Pre-commit hook for parent gSnake project

**Description:** As a maintainer, I want a pre-commit hook for the parent gSnake project that validates gsnake-core workspace and root-level integration files.

**Acceptance Criteria:**

- [ ] `.github/hooks/pre-commit` exists in parent repo
- [ ] Hook runs `cargo fmt --all -- --check` in gsnake-core
- [ ] Hook runs `cargo clippy --all-targets -- -D warnings` in gsnake-core
- [ ] Hook runs `cargo check` in gsnake-core
- [ ] Hook runs `cargo build` in gsnake-core
- [ ] Hook does NOT run `cargo test`
- [ ] Hook does NOT run `npm run test:e2e` (Playwright tests)
- [ ] Hook does NOT recurse into submodules (gsnake-web, gsnake-levels, etc.)
- [ ] Typecheck/lint passes

### US-003: Shared hooks README for gsnake-editor

**Description:** As a maintainer, I want clear documentation in gsnake-editor explaining how to enable and verify hooks.

**Acceptance Criteria:**

- [ ] `.github/hooks/README.md` created in gsnake-editor
- [ ] README explains `git config core.hooksPath .github/hooks` command
- [ ] README includes verification step (`git config --get core.hooksPath`)
- [ ] README includes disable step (`git config --unset core.hooksPath`)
- [ ] README lists exact commands run by the hook
- [ ] Typecheck/lint passes

### US-004: Helper script to enable hooks in all repos

**Description:** As a maintainer, I want a helper script that enables hooks in the parent repo and all initialized submodules with one command.

**Acceptance Criteria:**

- [ ] Script created at `.github/hooks/enable-all-hooks.sh` in parent repo
- [ ] Script sets `core.hooksPath` in parent repo
- [ ] Script iterates through initialized submodules and sets `core.hooksPath` in each
- [ ] Script skips uninitialized submodules gracefully
- [ ] Script provides clear output showing which repos had hooks enabled
- [ ] Script is executable (`chmod +x`)
- [ ] Usage documented in parent repo's `.github/hooks/README.md`
- [ ] Typecheck/lint passes

### US-005: Update parent README to document helper script

**Description:** As a maintainer, I want the parent repo's hooks README to document the helper script and per-repo enablement.

**Acceptance Criteria:**

- [ ] Parent repo's `.github/hooks/README.md` updated
- [ ] Documents running `./enable-all-hooks.sh` for one-command enablement
- [ ] Documents manual per-repo enablement as alternative
- [ ] Lists which repos have hooks available
- [ ] Explains that submodule hooks are independent (can enable selectively)
- [ ] Typecheck/lint passes

### US-006: Document hook exclusions

**Description:** As a maintainer, I want clarity that hooks exclude tests and don't recurse into submodules.

**Acceptance Criteria:**

- [ ] Hook scripts include comments stating they avoid tests
- [ ] Parent repo hook comments explicitly state "does not check submodules"
- [ ] Each README lists exact commands run
- [ ] Clear explanation that tests are run in CI, not pre-commit
- [ ] Typecheck/lint passes

## Functional Requirements

- FR-1: `gsnake-editor` provides a pre-commit hook under `.github/hooks` running format/lint/typecheck/build.
- FR-2: Parent gSnake project provides a pre-commit hook checking only gsnake-core (Rust workspace).
- FR-3: Parent hook must NOT recurse into submodules (gsnake-web, gsnake-levels, etc.).
- FR-4: Helper script enables hooks in parent + all initialized submodules.
- FR-5: All hooks remain opt-in via `git config core.hooksPath .github/hooks`.
- FR-6: No unit, integration, or e2e tests run in any pre-commit hook.

## Non-Goals (Out of Scope)

- No changes to existing submodule hooks (gsnake-web, gsnake-levels, gsnake-specs)
- No CI enforcement changes
- No automatic hook installation (remains opt-in)
- No gsnake-python hook in this iteration (submodule not initialized)

## Design Considerations

- Helper script should use `git submodule foreach` or similar to iterate submodules
- Script should check if `.github/hooks` exists before trying to enable hooks
- Keep parent hook focused on gsnake-core only; submodules have their own hooks
- gsnake-editor may need format/lint scripts added to package.json (similar to gsnake-web)

## Technical Considerations

- `gsnake-editor` uses Svelte + TypeScript + Vite; likely needs Prettier and ESLint
- Parent repo contains gsnake-core (Rust workspace with members: core, wasm, cli)
- Parent repo has root package.json with Playwright e2e tests - exclude from hook
- Helper script should be POSIX-compatible (macOS/Linux)
- Submodules may be sparse (not all initialized); script must handle gracefully

## Success Metrics

- Maintainers can enable hooks in all repos in under 30 seconds using helper script
- Parent hook runtime under 1 minute on typical dev machine
- gsnake-editor hook runtime under 2 minutes on typical dev machine
- At least one formatting or lint error caught in gsnake-editor or parent during initial rollout

## Open Questions

- Should the helper script also provide a disable-all-hooks option?
- Should gsnake-editor use the same lint/format tooling as gsnake-web for consistency?
