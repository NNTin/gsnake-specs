# Epic Brief: Folder Structure Restructuring

## Summary

The current folder structure of the `gSnake` project is fragmented, with files for different components scattered across the root. This Epic aims to reorganize the project into a modular, Git-submodule compatible structure. The project will be divided into self-contained units: `gsnake-core` (Rust engine plus bindings for WASM, CLI, and Python), `gsnake-web` (Svelte, moved into a git submodule), and an empty `gsnake-python` submodule boundary (no implementation yet). The parent repository will focus on integration, project specifications (Epics), and global CI/CD pipelines, while components handle their own logic and data (e.g., moving `data/levels.json` into `gsnake-core`). The WASM package name is `gsnake-wasm`; local file-based linking is required for development.

## Context & Problem

The `gSnake` project consists of several distinct components with diverging deployment needs:

- **Web Frontend:** A Svelte-based web application.
- **Core Backend + Bindings:** The Rust game engine (`gsnake-core`), which will now include `data/levels.json`, plus bindings for WASM, CLI (`ratatui`), and Python.
- **WASM Bridge:** Rust bridge for the Web frontend (package name `gsnake-wasm`).
- **Python Support:** Bindings live in `gsnake-core/engine/bindings/py` (using `pyproject.toml`).
- **Python App Boundary:** `gsnake-python` is an empty submodule boundary only in the current scope (no implementation yet).

Currently, these components are co-located, leading to:

- **Deployment Complexity:** Mixed dependencies make publishing NPM and PyPI packages brittle.
- **Version Management:** Challenges in maintaining independent versioning for each module.
- **CI/CD Inefficiency:** Difficulty in executing platform-specific GitHub Actions pipelines independently.
- **Structural Debt:** The lack of clear boundaries prevents the transition to Git submodules, which is required for scaling the project and its CI/CD infrastructure.

## Requirements

- **Versioning:** Each submodule repository adheres to Semantic Versioning (SemVer). Versions are incremented and released manually by a human.

- **Development Linking:** During development, modules are linked locally (required):

  - WASM: `wasm-pack build` then use a `file:` dependency on `gsnake-core/engine/bindings/wasm/pkg` (the `pkg/` directory is generated and not committed). A root script rebuilds the WASM package before dev/test.
  - Python: `pip install -e gsnake-core/engine/bindings/py` using `pyproject.toml`.

- **Happy Path Focus:** Initial implementation will prioritize the happy path for integration, with stability improvements addressed iteratively.

- **Integration Build:** The parent repository can build everything together for integration tests.

- **Root Rust Config:** The root retains Rust configuration required for building and end-to-end testing, and must support running Rust commands from the parent directory.

- **Shared Level Access:** Levels are accessed through bindings that expose the same API in core and WASM, so consumers do not need to read `levels.json` directly.

- **Web Submodule Boundary:** All Svelte web files move into a git submodule named `gsnake-web`, and no web app source remains at the repository root.

- **Web App Migration:** The existing Svelte app is moved into a fresh `gsnake-web` repository (new history) and added back to the parent as a submodule; root references, scripts, and docs are updated to point at the submodule path. The `gsnake-web` remote is `git@github.com:NNTin/gsnake-web.git`.

## CI/CD Suggestions (Best Practices)

- Use Python as the orchestration layer in CI for integration steps.
- Use separate workflows for integration tests and web deployment; keep integration gated on `main` and PRs.
- Check out submodules explicitly in CI (pinned SHAs) and cache Rust/Node dependencies per module.
- Pin Rust toolchain versions in CI to match root configuration and reduce drift.
- Deploy the Svelte web app from the parent workflow after integration tests pass.

**Recommendation:** Start with a single parent workflow that checks out submodules, runs integration builds/tests (including Playwright), and deploys the Svelte web app on `main`. Split into module-specific workflows after submodule isolation is stable.

## Success Criteria

- `gsnake-core` can be built and tested in complete isolation from the parent repository.

- `gsnake-web` can be built and tested in complete isolation, consuming the core engine via local links.

- `gsnake-web` exists as a git submodule pointer in the parent repository.

- All Svelte-related configs live exclusively under `gsnake-web`, while Playwright e2e tests remain in the parent repo.

- The parent `.github/workflows` runs integration tests and the Svelte web deployment with submodules checked out.

- The existing Playwright test suite passes at the root level, confirming the engine and web client are communicating correctly.

- The folder structure is ready for conversion to Git submodules after migration and proper isolation.
