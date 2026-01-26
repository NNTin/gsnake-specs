# PRD: Standalone Submodules Build

## Introduction

Today, several gSnake submodules (gsnake-web, gsnake-levels, gsnake-editor, gsnake-specs) cannot build or test when checked out alone because they reference sibling repos in the parent workspace. This PRD defines a migration to git-based dependencies so each submodule can build/test independently while still allowing the root repo (`github.com/nntin/gsnake`) to work with local submodules.

## Goals

- Each submodule (gsnake-web, gsnake-levels, gsnake-editor, gsnake-specs) can build/test without any sibling repos present.
- All cross-repo dependencies resolve via git URLs by default.
- Root repo continues to work with checked-out submodules without breaking local workflows.
- Documentation in `gsnake-specs/tasks/repo-architecture.md` reflects the new dependency model.

## User Stories

### US-001: Git dependency strategy defined per repo
**Description:** As a maintainer, I want explicit git-based dependency rules for each submodule so I can build/test with only that repo checked out.

**Acceptance Criteria:**
- [ ] Document dependency rules for each submodule: gsnake-web, gsnake-levels, gsnake-editor, gsnake-specs.
- [ ] Each rule specifies the git URL and expected tag/branch strategy.
- [ ] Each rule specifies how to override to local paths when in the root repo.
- [ ] Documentation lives in a new section within `gsnake-specs/tasks/repo-architecture.md` or a linked task file.

### US-002: gsnake-web uses git dependency for wasm/core artifacts
**Description:** As a developer, I want gsnake-web to resolve core/wasm dependencies via git so it builds independently.

**Acceptance Criteria:**
- [ ] gsnake-web uses git dependencies (not `file:`) to fetch gsnake-core/wasm artifacts.
- [ ] Build and test scripts run without any sibling repos present.
- [ ] If an override mechanism exists for local dev, it is documented and optional.
- [ ] Typecheck/test commands run without requiring parent repo.

### US-003: gsnake-levels uses git dependency for gsnake-core
**Description:** As a developer, I want gsnake-levels to use gsnake-core via git so it builds independently.

**Acceptance Criteria:**
- [ ] `gsnake-levels` resolves `gsnake-core` dependency via git URL (not `../gsnake-core`).
- [ ] Any CLI invocation that used `--manifest-path ../gsnake-core` is replaced or made git-resolvable.
- [ ] `cargo test` runs without any sibling repos present.

### US-004: gsnake-editor builds/tests without gsnake-web sibling
**Description:** As a developer, I want gsnake-editor to run and test without requiring gsnake-web as a sibling repo.

**Acceptance Criteria:**
- [ ] gsnake-editor’s integration/test-mode path uses a git-resolvable dependency or optional config (e.g., env var) instead of hardcoding `http://localhost:3000` without a fallback.
- [ ] Document how to run the editor standalone, including any required external service.
- [ ] `npm run build` and `npm run check` run without sibling repos.

### US-005: gsnake-specs builds mkdocs without sibling repos
**Description:** As a maintainer, I want gsnake-specs to build documentation independently.

**Acceptance Criteria:**
- [ ] MkDocs build works in gsnake-specs without relying on parent repo paths.
- [ ] Any references to sibling repo paths are replaced with git URLs or made optional.
- [ ] Build instructions are added/updated in gsnake-specs docs.

### US-006: Root repo supports local overrides for submodule dev
**Description:** As a developer, I want the root repo to keep working with local submodules while defaulting to git dependencies.

**Acceptance Criteria:**
- [ ] Document an override mechanism for local development in the root repo (e.g., scripts or environment flags).
- [ ] Override does not require modifying committed dependency declarations.
- [ ] Root workflow continues to pass with submodules checked out.

## Functional Requirements

1. FR-1: All submodules must resolve cross-repo dependencies via git URLs by default.
2. FR-2: Each submodule must provide build/test commands that succeed with only that repo checked out.
3. FR-3: Root repo must provide a documented override path to use local submodule checkouts without editing dependency files.
4. FR-4: `gsnake-specs/tasks/repo-architecture.md` must be updated to reflect the git-based dependency model and overrides.
5. FR-5: Dependency strategy must include a versioning/tagging policy (e.g., tags, branches) for git dependencies.

## Non-Goals (Out of Scope)

- Publishing packages to registries (crates.io, npm, PyPI).
- Rewriting core architecture or changing existing APIs.
- Introducing monorepo tooling.

## Design Considerations

- Prefer minimal friction for contributors: default `git` dependencies should “just work.”
- Provide a clean local override workflow for developers working in the root repo.

## Technical Considerations

- Git dependencies must target `github.com/nntin/gsnake` for shared code, or submodule repo URLs from `.gitmodules`.
- Rust git dependencies should pin a tag or commit for reproducibility.
- JS git dependencies should use git URLs and pinned refs; avoid `file:` paths.
- For mkdocs, ensure any repo references are externalized or removed.

## Success Metrics

- Each submodule builds/tests successfully when cloned alone (verified locally or in CI).
- Root repo builds/tests successfully with submodules checked out.
- No remaining `../gsnake-*` path dependencies in submodules (except optional local overrides).

## Open Questions

1. Which tagging scheme will be used for git dependencies (semantic version tags vs. branch pins)?
2. Should local overrides use env vars, scripts, or per-repo config files?
3. Do we need per-repo CI to validate standalone builds, or only root CI?
