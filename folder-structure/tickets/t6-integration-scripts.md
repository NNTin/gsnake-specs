## Objective

Add root orchestration scripts to build, link, and test the modular structure consistently for local dev and CI.

## Scope

**Included:**

- Implement `scripts/build_core.py`, `scripts/build_wasm.py`, `scripts/link_web.py`, and `scripts/test_integration.py`
- Ensure scripts orchestrate the same steps described in Core Flows
- Make scripts callable from the parent repo for integration runs

**Excluded:**

- GitHub Actions workflow changes
- Tooling changes inside submodules unrelated to orchestration

## Spec References

- spec:epics/folder-structure/tech-plan.md - Root (Integration Layer), Orchestration
- spec:epics/folder-structure/core-flows.md - Best-Practice Suggestions, Flows 1-4

## Implementation Details

Provide a small set of root scripts to run build/link/test flows consistently across local development and CI, including the WASM rebuild step and integration test execution.

## Acceptance Criteria

- [ ] `scripts/build_core.py` builds the Rust workspace under `gsnake-core/`
- [ ] `scripts/build_wasm.py` runs `wasm-pack build` for `engine/bindings/wasm`
- [ ] `scripts/link_web.py` ensures the `file:` dependency is set for `gsnake-wasm`
- [ ] `scripts/test_integration.py` runs integration tests from the root
- [ ] Scripts can be executed from the parent repo without manual `cd`

## Dependencies

- Depends on: t2-duplicate-rust-config
- Depends on: t3-wasm-file-linking-script
- Depends on: t4-uniform-level-access-api
