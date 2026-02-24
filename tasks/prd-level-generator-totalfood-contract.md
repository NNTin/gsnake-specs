# PRD: Level Generator TotalFood Contract Alignment

## 1. Introduction/Overview

`gsnake-web` fails to start default gameplay when generated levels from `gsnake-levels` do not include `totalFood`, while web contract validation requires it. This feature aligns level generation, generated artifacts, and startup validation paths so default levels load reliably and contract drift is prevented.

The work targets broader cross-runtime contract consistency (Rust core, WASM payloads, and web validation), with implementation centered on `gsnake-levels` as the source of generated `levels.json`.

## 2. Goals

- Ensure generated `levels.json` always includes a valid `totalFood` value for each level.
- Preserve compatibility with existing level source definitions by auto-updating old sources that omit `totalFood`.
- Keep explicit `totalFood` values when provided in source data.
- Ensure WASM artifacts are rebuilt after level generation changes so embedded levels stay in sync.
- Add E2E coverage that fails if the web root shows startup error state (`Level Load Failed`).

## 3. User Stories

### US-001: Generate schema-compliant levels with totalFood

**Description:** As a game runtime consumer, I want generated levels to always include `totalFood` so startup validation succeeds across web and WASM consumers.

**Acceptance Criteria:**

- [ ] `gsnake-levels` generator outputs `totalFood` for every generated level in `gsnake-core/engine/core/data/levels.json`.
- [ ] For each level, computed fallback is `food.length + floatingFood.length + fallingFood.length` when no explicit value exists.
- [ ] Generated output remains valid against `contracts/level-definition.schema.json`.
- [ ] Typecheck/lint passes.

### US-002: Preserve explicit totalFood while migrating old level sources

**Description:** As a level author, I want old source definitions without `totalFood` to be updated automatically while preserving explicit values when already defined.

**Acceptance Criteria:**

- [ ] Generator behavior supports: explicit source `totalFood` takes precedence; missing `totalFood` is auto-derived.
- [ ] Existing old level source definitions without `totalFood` are updated as part of generation/migration workflow.
- [ ] Generator tests cover both explicit and missing `totalFood` cases.
- [ ] Typecheck/lint passes.

### US-003: Keep WASM embedded levels synchronized with generated levels

**Description:** As a web player user, I want embedded default levels in WASM to match the latest generated levels so startup behavior is deterministic.

**Acceptance Criteria:**

- [ ] Project workflow includes rebuilding WASM after generated levels update.
- [ ] `gsnake-core/engine/bindings/wasm/pkg/*` reflects regenerated level data.
- [ ] No manual patching of generated `levels.json` is required.
- [ ] Typecheck/lint passes.

### US-004: Prevent startup regression with root-load E2E coverage

**Description:** As a QA engineer, I want an E2E test that validates root startup does not show initialization failure so this regression is caught in CI.

**Acceptance Criteria:**

- [ ] E2E suite includes a test that visits `http://localhost:3000/` (or `/` via baseURL) and waits for playable game readiness.
- [ ] Test asserts startup error dialog/message is absent, specifically that `Level Load Failed` and `Failed to initialize game engine. Please reload and try again.` are not visible.
- [ ] Test uses deterministic readiness checks (no arbitrary fixed sleeps).
- [ ] Typecheck/lint passes.
- [ ] Verify in browser using dev-browser skill.

## 4. Functional Requirements

- FR-1: `gsnake-levels` generator must emit `totalFood` on every generated level record.
- FR-2: When source level definition includes `totalFood`, generator must use that explicit value.
- FR-3: When source level definition omits `totalFood`, generator must compute and emit `food + floatingFood + fallingFood`.
- FR-4: Generation pipeline must update legacy level sources lacking `totalFood` to the new contract shape.
- FR-5: Generated `gsnake-core/engine/core/data/levels.json` must remain machine-generated from `gsnake-levels` only.
- FR-6: Post-generation workflow must rebuild WASM artifacts so embedded `getLevels()` payload reflects updated generated levels.
- FR-7: E2E tests must include root startup validation to detect initialization-failure regressions.
- FR-8: CI/local test workflow must fail if startup error UI appears on root load.

## 5. Non-Goals (Out of Scope)

- No gameplay logic changes in Rust engine food/exit/completion behavior.
- No redesign of editor UX or level authoring UI.
- No introduction of new level fields beyond required contract alignment for existing fields.
- No support for manually editing `gsnake-core/engine/core/data/levels.json` outside generation flow.

## 6. Design Considerations

- Reuse existing startup readiness selectors already used by E2E tests where possible.
- Keep error assertion text checks explicit to ensure the regression signal is clear in test failures.

## 7. Technical Considerations

- Current mismatch exists between web schema validation (`totalFood` required) and generated default levels missing the field.
- `gsnake-levels` is the source of truth for generated levels; all fixes should be made upstream there.
- WASM package embeds levels; regeneration without WASM rebuild can leave runtime artifacts stale.
- Contract drift checks/tests should be updated to enforce generator output invariants.

## 8. Success Metrics

- Root web app load succeeds with playable grid and no level-init error (A).
- `npm run test:e2e` passes with the new root-load regression test (B).
- Automated tests in generator/runtime layers fail if generated levels omit `totalFood` (C).
- Legacy level sources without `totalFood` are migrated and generation remains deterministic.

## 9. Open Questions

- Should migration of old level sources happen as a one-time scripted rewrite, or as implicit normalization inside generation output only?
- Should CI add an explicit guard that rejects PRs where generated `levels.json` and WASM artifacts are out of sync?
