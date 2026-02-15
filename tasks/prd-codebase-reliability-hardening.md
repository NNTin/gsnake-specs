# PRD: gSnake Codebase Reliability Hardening

## 1. Introduction/Overview

`gsnake-specs/tasks/FINDINGS.md` (dated February 15, 2026) identified critical reliability and maintainability risks in the gSnake codebase: engine panic paths, insufficient input validation, query-parameter bypass behavior, duplicated contract validation logic, and test coverage gaps across web, editor, and core-engine contract behavior.

This PRD defines a single reliability-hardening feature set that turns those findings into implementable requirements. The intent is to remove crash-on-bad-input behavior, make validation consistent across modules, and raise confidence through targeted tests.

## 2. Goals

- Eliminate known crash paths caused by unchecked assumptions in production code.
- Reject invalid grid and query inputs deterministically with explicit fallback/error behavior.
- Reduce schema/validation drift by centralizing LevelDefinition validation logic.
- Expand automated tests to cover missing component, contract, and edge-case scenarios.
- Replace panic-oriented tooling paths with actionable aggregated errors where feasible.

## 3. User Stories

### US-001: Remove critical engine panic path

**Description:** As a player, I want malformed snake state to return a structured engine error instead of panicking so gameplay failures are handled safely.

**Acceptance Criteria:**

- [ ] `gsnake-core/engine/core/src/engine.rs` no longer uses `unwrap()` for head-segment access.
- [ ] Empty-segment snake state returns an explicit `EngineError` variant with a clear message.
- [ ] Existing valid gameplay behavior is unchanged for non-malformed states.
- [ ] Rust tests cover the malformed state and assert error (not panic).
- [ ] Typecheck/lint passes.

### US-002: Enforce grid dimension validation

**Description:** As a level author and player, I want invalid grid sizes rejected so engine allocation and frame processing remain safe.

**Acceptance Criteria:**

- [ ] Engine validates `gridSize` before frame/buffer allocation.
- [ ] Grid size `<= 0` is rejected with explicit error.
- [ ] Grid size above a defined safe max frame threshold is rejected with explicit error.
- [ ] Tests cover boundary values: negative, zero, min valid, max valid, and max+1.
- [ ] Typecheck/lint passes.

### US-003: Block `?level=NaN` and invalid level query input

**Description:** As a web player, I want invalid `level` query parameters sanitized so the game always starts in a consistent state.

**Acceptance Criteria:**

- [ ] `WasmGameEngine` treats non-integer values (`NaN`, empty, decimal, text) as invalid.
- [ ] Out-of-range levels fall back to level 1 (or configured default) consistently.
- [ ] Unit tests cover valid and invalid query parameter permutations.
- [ ] No inconsistent-level startup state is reachable through URL input.
- [ ] Typecheck/lint passes.

### US-004: Centralize LevelDefinition validation contract

**Description:** As a maintainer, I want one canonical validation contract so schema changes do not require manual sync across Rust, editor server, and tests.

**Acceptance Criteria:**

- [ ] A canonical LevelDefinition schema/validator artifact is defined and documented.
- [ ] Editor server and web-facing validation logic consume the canonical validator or generated guards.
- [ ] E2E tests stop duplicating ad-hoc `isLevelDefinition` logic and use shared validation utilities.
- [ ] Contract drift checks exist in CI or test suite to catch schema mismatch.
- [ ] Typecheck/lint passes.

### US-005: Replace fragile manual editor validation structure

**Description:** As an editor maintainer, I want composable schema-driven validation so adding/changing fields does not require edits in many disconnected code paths.

**Acceptance Criteria:**

- [ ] Manual rule chains in `gsnake-editor/server.ts` are replaced or wrapped by a schema-driven validator module.
- [ ] Validation errors return structured, field-specific messages.
- [ ] Server tests include edge cases from findings: `gridSize=0`, negative coordinates, invalid field combinations.
- [ ] New fields can be added by extending a single validation source.
- [ ] Typecheck/lint passes.

### US-006: Close high-priority web test coverage gaps

**Description:** As a maintainer, I want missing component and input-handler tests added so regressions are caught before merge.

**Acceptance Criteria:**

- [ ] Dedicated tests exist for `Cell.svelte`, `Header.svelte`, and `RestartButton.svelte`.
- [ ] `KeyboardHandler` tests cover rapid key presses and modifier-key scenarios.
- [ ] Coverage reports include these files in exercised paths.
- [ ] Typecheck/lint passes.
- [ ] Verify in browser using dev-browser skill.

### US-007: Expand contract matrix coverage

**Description:** As a core-engine maintainer, I want full cell-type and stone interaction coverage so contract behavior remains stable across releases.

**Acceptance Criteria:**

- [ ] Web contract tests include FloatingFood, FallingFood, Stone, and Spike variants.
- [ ] Core contract tests expand the stone interaction matrix and document expected outcomes.
- [ ] Test fixtures cover mixed-cell combinations and invalid variants.
- [ ] Typecheck/lint passes.

### US-008: Replace panic-oriented tooling flows with error propagation

**Description:** As a CLI/tools user, I want batch commands to report all detected issues instead of crashing on first failure.

**Acceptance Criteria:**

- [ ] High-impact `unwrap()`/`expect()` sites in `gsnake-levels` and CLI playback paths are replaced with `Result` propagation.
- [ ] Validation commands aggregate and report multiple errors in one run where practical.
- [ ] Error messages include actionable context (file/level/field).
- [ ] Existing success-path behavior remains unchanged.
- [ ] Typecheck/lint passes.

### US-009: Document fragile engine behavior and field semantics

**Description:** As a contributor, I want physics-order and serialization semantics documented so changes do not introduce subtle regressions.

**Acceptance Criteria:**

- [ ] Gravity/collision order dependency rationale is documented adjacent to relevant engine code.
- [ ] Level serialization semantics (`totalFood`, optional fields) are documented in a single authoritative location.
- [ ] Wasm frame-emission pattern requirements are documented and linked from implementation.
- [ ] Documentation references are added to tests or developer docs for discoverability.
- [ ] Typecheck/lint passes.

## 4. Functional Requirements

- FR-1: Engine state access must not panic on missing snake segments.
- FR-2: Engine must return explicit validation error for malformed snake state.
- FR-3: Engine must validate `gridSize` before memory/frame allocation.
- FR-4: Engine must reject non-positive grid sizes.
- FR-5: Engine must reject grid sizes above safe frame-cap threshold.
- FR-6: Web query parsing must reject non-integer `level` values.
- FR-7: Web query parsing must clamp/fallback out-of-range `level` values.
- FR-8: Startup state must be deterministic regardless of invalid query params.
- FR-9: LevelDefinition validation must have one canonical schema source.
- FR-10: Editor/server validation must depend on canonical schema-derived checks.
- FR-11: E2E contract checks must reuse shared validators, not duplicate local logic.
- FR-12: Server validation responses must include field-specific error context.
- FR-13: Tests must cover `gridSize=0`, negative coordinates, and invalid field combos.
- FR-14: Web unit tests must cover Cell/Header/RestartButton components.
- FR-15: Keyboard handler tests must cover rapid input and modifier keys.
- FR-16: Contract fixtures must include FloatingFood/FallingFood/Stone/Spike cases.
- FR-17: Core contract tests must include expanded stone interaction matrix.
- FR-18: High-impact tooling panic paths must be replaced with structured error propagation.
- FR-19: Tooling validation commands must support multi-error reporting where feasible.
- FR-20: Gravity-order and serialization semantics must be documented and discoverable.

## 5. Non-Goals (Out of Scope)

- Parallel solver/caching redesign and broader solver performance refactor.
- WASM load-time optimization and worker-based startup re-architecture.
- CI process supervision redesign beyond tests required for this feature set.
- Production security hardening beyond reliability findings tied directly to this PRD.
- Full repository-wide elimination of every `unwrap()` occurrence in a single milestone.

## 6. Design Considerations

- Preserve existing layered architecture boundaries (core engine, bindings, web adapter, UI).
- Prefer generated validators/type-guards over copy-pasted contract checks.
- Keep error surfaces human-readable and machine-parseable for tooling workflows.

## 7. Technical Considerations

- Rust and TypeScript validation representations must remain semantically aligned.
- Contract canonicalization should minimize duplicate schemas while remaining practical for build/test workflows.
- Existing CI should enforce drift detection and regression coverage for new validator contracts.
- Refactors should avoid behavior changes to valid gameplay and existing level data.

## 8. Success Metrics

- Zero known critical panic paths from FINDINGS remain in engine runtime path.
- Invalid `level` query parameters no longer create inconsistent game state.
- Editor validation rules are maintained through one schema-driven pathway.
- Newly identified missing tests are present and green in CI.
- Contract regression coverage includes all previously missing variants.
- Tooling commands return actionable errors instead of first-failure panics for targeted paths.

## 9. Open Questions

- What should be the canonical source of truth for schema generation: Rust-first, JSON Schema-first, or TypeScript-first?
- For grid-size caps, what exact max bound should be treated as stable API contract versus internal safety guard?
- Should error aggregation in `gsnake-levels` be introduced incrementally per command or enforced repo-wide in one pass?
- Is this PRD executed as one epic with multiple tickets, or split into phased PRDs aligned to the Findings action plan weeks?
