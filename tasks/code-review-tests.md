# Code Review: Test Suite

## Summary

The gSnake test suite has genuine strengths: the Rust engine is tested at a unit level with deterministic contract-serialization checks and a property-based gravity harness, and the TypeScript contract tests cover all enum variants and fixture round-trips thoroughly. However, three categories of concern stand out. First, several tests in both `enums.test.ts` and `e2e/contract.spec.ts` are structurally tautological — they always pass regardless of what the production code does — giving false confidence on the cross-language contract boundary. Second, the `gsnake-editor/src/tests/integration.test.ts` file contains two tests that unconditionally pass (`expect(true).toBe(true)`) and verify nothing about actual integration behavior. Third, the E2E suite is fully Chromium-only, uses brittle timing (`waitForTimeout`) in one place, and completely omits coverage for the `AllComplete` game state flow, the `LevelComplete` keyboard path, and the keyboard `r`/`q` shortcuts in play.

______________________________________________________________________

## Findings

### [SEVERITY: Critical] Enum contract tests verify type-system tautologies, not runtime behavior

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/tests/contract/enums.test.ts:10-27` (Direction block); similarly lines 29-78 (CellType) and 80-110 (GameStatus)

**Issue:** Every test in this file constructs a hardcoded `const valid: CellType[] = [...]` array literal and then calls `expect(valid).toHaveLength(10)`. The array is written by the test author, not read from the source-of-truth type. The TypeScript compiler already guarantees that a literal matching `CellType[]` is well-typed; the runtime assertion adds nothing. If the Rust side added a new `CellType` variant and the TypeScript union was updated but this list was not, the test would still pass because the hardcoded length would simply be updated to match the old count. Similarly, `toContain(dir)` where both the set and the value are literals is trivially true. The tests never call any production code.

**Impact:** A new `CellType` variant introduced in Rust (e.g., `Portal`) that is missing from the TypeScript union would not be caught by these tests. The false confidence is particularly dangerous because the file is named `enums.test.ts` — reviewers assume it is the authoritative enum-coverage gate.

**Suggestion:** Import the actual type guard arrays used in production (e.g., from `types.test.ts`'s `VALID_CELL_TYPES`) or from the generated TypeScript bindings (`bin/export_ts.rs` output). Assert that the runtime set of valid strings exactly matches the Rust-derived expected set. Alternatively, remove `enums.test.ts` entirely because `types.test.ts` already provides real runtime type-guard coverage for all variants.

______________________________________________________________________

### [SEVERITY: Critical] Integration test file always passes unconditionally

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/tests/integration.test.ts:38-39`, `65-67`

**Issue:** Both tests in this file end in `expect(true).toBe(true)` in their catch/non-OK branches — the branches that are always taken in a normal CI environment where the gsnake-web server is not running. The test names (`"should skip integration tests if GSNAKE_WEB_URL is unreachable"`, `"should validate gsnake-web test endpoint if reachable"`) imply conditional skipping, but Vitest does not skip tests — it runs them, and they always pass. There is no `test.skip` or `test.todo`; the catch block silently marks the run as green.

**Impact:** Any future regression in CORS, the editor API, or the web server startup process would produce no signal from this test file. The file occupies two test slots in the suite and creates the misleading impression that integration behavior is verified.

**Suggestion:** Replace the unconditional-pass pattern with Vitest's `test.skipIf` using an environment variable guard, or convert to proper E2E tests in Playwright (where both servers are guaranteed to be running). If the intent is purely to document expected integration points, delete the file and add a comment in the Playwright spec.

______________________________________________________________________

### [SEVERITY: Critical] `contract.spec.ts` — extended CellType variants excluded from E2E grid validation

**File:** `/home/nntin/git/gSnake/e2e/contract.spec.ts:171`

**Issue:** The E2E test `"all grid cells contain valid CellType values"` checks cells against `['Empty', 'SnakeHead', 'SnakeBody', 'Food', 'Obstacle', 'Exit']` — a list of six variants — but the production `CellType` enum (and its TypeScript counterpart) includes ten values: `FloatingFood`, `FallingFood`, `Stone`, and `Spike` are absent from the allowlist. Any grid cell rendered with `Stone` would be flagged as an unknown type and fail — but any cell that *should* display `Stone` but instead displays `Empty` due to a regression would pass silently.

**Impact:** Levels that use stones, spikes, or floating/falling food are not validated at the E2E layer. Regressions in the WASM→TypeScript mapping for these extended variants would go undetected.

**Suggestion:** Expand the `validCellTypes` constant to include all ten members of the `CellType` union (`FloatingFood`, `FallingFood`, `Stone`, `Spike`). Add a second dedicated E2E test using a level that is known to contain each extended cell type, and assert that at least one cell of each expected type appears in the rendered grid.

______________________________________________________________________

### [SEVERITY: High] Rust contract tests omit extended CellType variants from round-trip coverage

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/tests/contract_tests.rs:234-250`

**Issue:** `test_celltype_roundtrip` iterates over only six of the ten `CellType` variants: `Empty`, `SnakeHead`, `SnakeBody`, `Food`, `Obstacle`, `Exit`. The four extended variants (`FloatingFood`, `FallingFood`, `Stone`, `Spike`) introduced by the gravity/stone mechanics system are not round-tripped. Similarly, `test_celltype_serialization` (lines 28-47) tests only the same six variants.

**Impact:** A serialization annotation change (e.g., a `#[serde(rename)]` typo) on `FloatingFood` or `FallingFood` would not be caught by the contract test suite. The TypeScript side's fixture tests rely on these exact string values (`"FloatingFood"`, `"FallingFood"`), so a casing drift would cause a cross-language breakage that the Rust tests would miss.

**Suggestion:** Add `FloatingFood`, `FallingFood`, `Stone`, and `Spike` to both the serialization assertion block and the round-trip loop in `contract_tests.rs`. The fix is two lines per test.

______________________________________________________________________

### [SEVERITY: High] `KeyboardHandler` test has no coverage for `LevelComplete` state or `r`/`q` shortcuts in playing state

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/tests/unit/KeyboardHandler.test.ts`

**Issue:** The `KeyboardHandler` source (`KeyboardHandler.ts:75-87`) handles `r` and `q` keys during the `"Playing"` state: pressing `r` restarts the level and `q` loads level 1. The `"LevelComplete"` state is completely absent from the switch statement in `handleKeyPress` (the source falls through to the `default` which is not shown — there is no case for it at all), meaning key presses during `LevelComplete` do nothing. None of these behaviors are tested: the `r` key in playing state, the `q` key in playing state, and the `LevelComplete` state's complete key-press passthrough.

**Impact:** If someone accidentally removed or altered the `r`/`q` handling in `handlePlayingState`, the test suite would not catch it. The `LevelComplete` state represents a significant user moment (the level is won); having no test for it means the keyboard silence during that state could be broken without notice.

**Suggestion:** Add tests: (1) press `r` in `"Playing"` state — expect `restartLevel` called; (2) press `q` in `"Playing"` state — expect `loadLevel(1)` called; (3) set status to `"LevelComplete"`, press any key, expect neither `restartLevel` nor `processMove` nor `loadLevel` to be called.

______________________________________________________________________

### [SEVERITY: High] `WasmGameEngine` test does not verify the `loadLevel` public API or re-initialization guard

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/tests/unit/WasmGameEngine.test.ts`

**Issue:** The source's `loadLevel(levelNumber)` method is the primary way the UI switches levels (called directly by `LevelSelectorOverlay`). The test suite never calls `engine.loadLevel(n)` directly; it only tests the internal `nextLevel()` and `resetLevel()` paths. Additionally, `WasmGameEngine.init()` has a guard (`if (this.initialized) return` with a console warning) that is never tested. The `loadLevel(0)` and `loadLevel(n > levels.length)` out-of-range paths are also untested.

**Impact:** A bug in the 0-based↔1-based index conversion inside `loadLevel` (e.g., `loadLevelByIndex(levelNumber - 1)`) could silently load the wrong level and would not be caught. The guard preventing double-initialization could be accidentally removed.

**Suggestion:** Add tests: (1) call `engine.loadLevel(2)` on a two-level setup, verify `levelChanged` emits with level 2; (2) call `engine.init(...)` twice, verify the second call is a no-op (no double WASM init); (3) call `engine.loadLevel(0)` and verify an error or rejection is thrown.

______________________________________________________________________

### [SEVERITY: High] `CompletionTracker` has no test for corrupted (non-array) localStorage content

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/tests/unit/CompletionTracker.test.ts`

**Issue:** The source's `getCompletedLevels` catches JSON parse errors and returns `[]`, and also handles the case where `JSON.parse` succeeds but the value is not an array (via the `Array.isArray` check). The test for `"filters out non-numeric values"` covers bad values inside an array, but no test exercises: (a) a non-array JSON value (e.g., `"42"`, `{}`, `null`); (b) malformed JSON that throws on parse; (c) `markCompleted` when `localStorage.setItem` throws (quota exceeded).

**Impact:** A future refactor that accidentally removed the `Array.isArray` guard would pass all existing tests. Storage quota errors in `markCompleted` are silently swallowed with no test verifying the behavior.

**Suggestion:** Add tests: (1) set storage to `"42"` (a number, not array) — expect `getCompletedLevels()` returns `[]`; (2) set storage to `"not json{"` — expect `getCompletedLevels()` returns `[]`; (3) optionally mock `localStorage.setItem` to throw and verify `markCompleted` does not propagate the exception.

______________________________________________________________________

### [SEVERITY: High] `editor-integration.spec.ts` uses `waitForTimeout(2000)` — unconditional sleep

**File:** `/home/nntin/git/gSnake/e2e/editor-integration.spec.ts:57`

**Issue:** After the popup page loads, the test calls `await popup.waitForTimeout(2000)` — a fixed 2-second sleep — before asserting anything. This is a classic timing anti-pattern: it is too slow when the page loads quickly (needlessly inflating CI time) and too fast when the page is slow (causing false failures). The assertion that follows (`if (bodyText && bodyText.includes(errorText))`) also inverts the usual test logic — it throws on finding an error string rather than asserting a positive condition.

**Impact:** The test is fragile on slow CI machines and gives no meaningful positive assertion that the game actually loaded successfully. The inverted assertion means the test passes vacuously if `body` is empty or if `textContent` returns `null`.

**Suggestion:** Replace `waitForTimeout(2000)` with `await popup.waitForSelector('[data-element-id="game-field"]', { timeout: 8000 })`. Then assert `await expect(popup.locator('[data-element-id="game-field"]')).toBeVisible()` as the primary positive assertion. This is exactly the pattern used in the more mature `level-editor.spec.ts` at line 180.

______________________________________________________________________

### [SEVERITY: High] `gravity.spec.ts` makes fixed assumptions about level 1 completion path

**File:** `/home/nntin/git/gSnake/e2e/gravity.spec.ts:33-36`

**Issue:** The E2E gameplay test hard-codes `for (let i = 0; i < 11; i++) { pressAndExpectMoveIncrement('ArrowRight') }` and then asserts `levelCompleteBanner` is visible. This assumes that exactly 11 right presses from the default start position of level 1 completes the level. If the level definition changes, this test breaks silently — the banner would not appear, but the assertion would wait for `movesDisplay` to show `12` (which it might), and then fail only on the banner check. It also means the test is testing level data rather than engine behavior.

**Impact:** Any change to level 1's grid, snake position, or food count will break this test in a confusing way. The test is brittle to level design changes.

**Suggestion:** Add a `webServer` or `test.use` fixture that injects a known test-only minimal level (e.g., via the editor API `POST /api/test-level`). Load it with `/?test=true` so the game uses the known-minimal level, then complete it deterministically with a small number of moves that can be derived from the level definition itself.

______________________________________________________________________

### [SEVERITY: Medium] `contract.spec.ts` — `waitForTimeout(100)` timing dependency for 180-degree turn test

**File:** `/home/nntin/git/gSnake/e2e/contract.spec.ts:83-84`

**Issue:** The `"inputRejected error on 180-degree turn"` test presses `ArrowRight`, then calls `await page.waitForTimeout(100)`, then presses `ArrowLeft`. The 100 ms sleep is intended to ensure the first move is processed before the second is sent, but Playwright's `waitForFunction` after the first press would be a more reliable mechanism. If the engine processes moves asynchronously (e.g., via a debounce or animation frame), 100 ms may not be enough under load.

**Impact:** The test may intermittently fail on slow CI machines, producing a false negative. Alternatively, if the engine rejects the second keypress before processing the first (input lock), the `__gsnakeContract.error` may never be set.

**Suggestion:** Replace `waitForTimeout(100)` with `await page.waitForFunction(moves => (window as any).__gsnakeContract?.frame?.state?.moves === moves + 1, initialMoves)` — the same pattern used elsewhere in the same file (lines 45-48).

______________________________________________________________________

### [SEVERITY: Medium] `snake-bugs.spec.ts` tests API persistence, not the actual rendering bug

**File:** `/home/nntin/git/gSnake/e2e/snake-bugs.spec.ts:44-116`

**Issue:** The test is titled `"snake appears on first draw and clears on second draw"` and described as testing a drawing bug fix. However, all three steps only use the editor API (`request.post`, `request.get`) to verify that the stored JSON was replaced — no browser page is loaded, no canvas or SVG is inspected. The test verifies that the HTTP endpoint replaces stored data, which is already covered by `server.test.ts`. The actual rendering behavior (whether the old snake's cells visually clear in the game) is completely untested.

**Impact:** If the drawing bug were reintroduced — a stale snake render persisting on screen from a previous level — this test would still pass because it never opens the browser.

**Suggestion:** Either rename and reframe the test as an API-storage test and move it to the server tests, or add a proper browser-based regression test: load a level with snake at positions A, load a second level with snake at positions B, and assert that cells at positions A no longer have the `.is-snake-segment` class in the rendered grid.

______________________________________________________________________

### [SEVERITY: Medium] `buildLevelExportPayload` has no test for empty-snake or null-exit edge cases

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/tests/types.test.ts`

**Issue:** `buildLevelExportPayload` can return `exit: null` when no exit cell is placed in the grid (the loop initializes `exit = null` and only sets it if a cell with `entity === "exit"` is found). The single happy-path test uses a grid with all entities. There is no test for: (a) no exit placed (returns `exit: null`); (b) no snake segments (`snakeSegments: []`); (c) duplicate exits (only the last one wins — is that intentional?).

**Impact:** If downstream consumers assume `exit` is always a `Position` object and the schema guard allows `null` exit (it does in `level-editor.spec.ts`'s local `LevelDefinition` type at line 16), a level exported without an exit would pass the schema but fail at game engine initialization. This edge case is never exercised.

**Suggestion:** Add tests for: (1) `snakeSegments: []` yields `snake: []`; (2) no exit cell yields `exit: null`; (3) two exit cells in the grid yields `exit` equal to the last one encountered (to document and lock the behavior).

______________________________________________________________________

### [SEVERITY: Medium] `EditorLayout.saveLoad.test.ts` load test does not assert all optional entity types

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/tests/EditorLayout.saveLoad.test.ts:185-227`

**Issue:** The `"loads a valid level file and renders the imported grid state"` test places obstacles, food, exit, floating food, falling food, stones, and spikes in the loaded level. However, the assertions only check: grid dimensions, two snake cells, one obstacle cell, one food cell, and one exit cell (`has-entity`). The positions of floating food (2,0), falling food (4,0), stones (0,5), and spikes (2,2) are loaded but never asserted as having any class. The `has-entity` class is also not specific enough — it does not distinguish between food and stones.

**Impact:** A regression in how the editor renders stone or spike cells after loading would go undetected.

**Suggestion:** Add assertions for `expect(getGridCell(container, 0, 2)).toHaveClass("has-entity")` (floating food at x=2, y=0), `getGridCell(container, 0, 4)` (falling food), `getGridCell(container, 5, 0)` (stone), and `getGridCell(container, 2, 2)` (spike). If the editor uses distinct CSS classes per entity type, assert those.

______________________________________________________________________

### [SEVERITY: Medium] `stores.test.ts` — no test for `countSnakeSegments` with mixed/extended cell types

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/tests/unit/stores.test.ts:97-114`

**Issue:** The `"syncs frame, gameState, snakeLength, and clears previous engine errors"` test verifies `snakeLength` is 2 by examining a frame where grid row 0 has `["SnakeHead", "SnakeBody", "Empty", "Empty"]`. The `countSnakeSegments` function (stores.ts:70-80) iterates all cells and counts `SnakeHead | SnakeBody`. There is no test for a frame with zero snake cells, or with a longer snake spread across multiple rows, or with stone/spike cells that should not increment the count.

**Impact:** A future typo in `countSnakeSegments` that accidentally includes `"Stone"` in the count would pass all current tests. A frame with no snake segments (empty snake, a valid game-over frame) would make `snakeLength` return 0, which is correct but untested.

**Suggestion:** Add: (1) a frame with a 5-segment snake across two rows — verify `snakeLength` is 5; (2) a frame with no `SnakeHead` or `SnakeBody` cells — verify `snakeLength` is 0; (3) a frame with all non-snake extended types — verify `snakeLength` is 0.

______________________________________________________________________

### [SEVERITY: Medium] Rust engine has no test for `AllComplete` game state transition

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs` (no test for `AllComplete`)

**Issue:** The `GameStatus::AllComplete` variant is tested for serialization in `contract_tests.rs` but never produced by any engine test. The WASM binding layer (not reviewed here) or the TypeScript `WasmGameEngine` wraps the engine, so `AllComplete` must be driven from the TypeScript side by calling `nextLevel()` past the last level. The Rust engine itself does not set `AllComplete` — this is a TypeScript-layer concern — but the status is part of the Rust type. No Rust-level test verifies that `GameStatus::AllComplete` can be deserialized from a Rust-produced JSON payload and used in downstream logic.

**Impact:** If the serialized string for `AllComplete` drifted (e.g., renamed to `"Complete"` in serde rename), the Rust contract tests would catch it, but there is a gap: the transition test that would catch behavioral drift in the TypeScript wrapper going from `LevelComplete` to `AllComplete` is absent.

**Suggestion:** Add a test in `WasmGameEngine.test.ts` that covers: given a one-level game, after `nextLevel()` is called, `events` does not include additional `levelChanged`/`frameChanged` events (because `nextLevel()` is a no-op past the last level). For the `AllComplete` state itself, add an E2E test that verifies the `AllComplete` overlay appears and the `q` keyboard shortcut returns to level 1.

______________________________________________________________________

### [SEVERITY: Medium] `workflow.spec.ts` and `complete-workflow.spec.ts` duplicate the same API test

**File:** `/home/nntin/git/gSnake/e2e/workflow.spec.ts` and `/home/nntin/git/gSnake/e2e/complete-workflow.spec.ts`

**Issue:** Both files perform the same `POST /api/test-level` upload and `GET /api/test-level` retrieval with CORS header verification. `workflow.spec.ts` (lines 20-89) and `complete-workflow.spec.ts` (lines 5-64) are nearly identical. The only addition in `complete-workflow.spec.ts` is the `isLevelDefinition` schema validation step and the `totalFood` calculation check — both valuable — but they are buried in a file that duplicates everything else.

**Impact:** Maintenance burden: any change to the API contract (e.g., a port number change) must be updated in two places. The duplication also inflates CI time.

**Suggestion:** Consolidate into `complete-workflow.spec.ts` (which is the more thorough test). Delete or significantly trim `workflow.spec.ts`. The unique CORS multi-port check in `workflow.spec.ts` (lines 74-88) is worth keeping — it could be moved to `server.test.ts` which already tests CORS policy in detail.

______________________________________________________________________

### [SEVERITY: Medium] Playwright config has no server for the editor API in `webServer`

**File:** `/home/nntin/git/gSnake/playwright.config.ts:33-44`

**Issue:** The `webServer` array starts two servers (the web app on port 3000 and the editor UI on port 3003) but does not start the editor API server on port 3001. Several E2E tests (`workflow.spec.ts`, `complete-workflow.spec.ts`, `snake-bugs.spec.ts`, `level-editor.spec.ts`) make direct HTTP requests to `http://localhost:3001/api/test-level`. Locally, this works only if the developer has manually started the API server. In CI the comment `// In CI, servers are started in workflow steps` documents that this is handled externally, but there is no fallback guard.

**Impact:** Running `npx playwright test` locally without the API server started produces cryptic network failures rather than a clear "service not ready" error. The `waitForService` helper in `level-editor.spec.ts` (lines 31-50) handles this gracefully for that test, but other spec files call the API directly without any retry.

**Suggestion:** Add the editor API server to the `webServer` array: `{ command: 'npm --prefix gsnake-editor run server', url: 'http://localhost:3001/health', reuseExistingServer: true, timeout: 30000 }`. Alternatively, add a global `setup.ts` fixture that uses `waitForService` for all three endpoints before any test runs.

______________________________________________________________________

### [SEVERITY: Low] `Cell.test.ts` — opacity test list is incomplete and uses assertion-by-coincidence

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/tests/unit/Cell.test.ts:51-65`

**Issue:** The `"applies opacity"` test parameterizes over 6 cell types but `Empty`, `SnakeBody`, `FallingFood`, `Exit`, `Obstacle`, and `Spike` are only partially represented. `SnakeBody` is absent from the opacity test entirely, and `FallingFood`'s opacity is not asserted. If a developer changes the Spike opacity from `"0.8"` to `"0.75"`, this test catches it — but if they add a new cell type with no opacity CSS, the type would be silently untested.

**Impact:** Low — visual regressions in opacity are cosmetic and unlikely to cause gameplay bugs.

**Suggestion:** Either explicitly assert that all 10 `CellType` variants have a specific expected opacity (using `it.each` with all members), or document that only certain types have non-default opacity and test exactly those.

______________________________________________________________________

### [SEVERITY: Low] `LandingPage.test.ts` tests only static markup existence without interaction

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/tests/LandingPage.test.ts`

**Issue:** All four tests verify that static text/buttons are rendered, but none verify that clicking `"Create New Level"` or `"Load Existing Level"` triggers the expected navigation event or callback. The component presumably emits events or navigates on click.

**Impact:** Low — markup regressions are caught, but behavioral regressions (e.g., the button's `on:click` handler being disconnected) are not.

**Suggestion:** Add a test that clicks the `"Create New Level"` button and verifies the expected Svelte event is dispatched (or, if using a callback prop, that it is invoked).

______________________________________________________________________

### [SEVERITY: Low] `fixtures.test.ts` — `"context uses camelCase keys"` test is vacuously satisfied

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/tests/contract/fixtures.test.ts:165-173`

**Issue:** The test iterates over `Object.keys(error.context)` and asserts that each value is a `"string"`. However, it does not verify that the keys themselves are camelCase (the comment says "context uses camelCase keys" but the assertion checks value types, not key naming). The actual fixture `error-with-context.json` has keys `"input"` and `"expected"` — single-word keys that would pass regardless of casing convention.

**Impact:** Very low — a context key with `snake_case` (e.g., `"rejection_reason"`) would not be caught.

**Suggestion:** Add a check like `expect(key).toMatch(/^[a-z][a-zA-Z0-9]*$/)` to each key to actually verify camelCase naming.

______________________________________________________________________

### [SEVERITY: Low] `server.test.ts` — test for negative coordinates documents, not validates

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/tests/server.test.ts:165-177`

**Issue:** The test `"should accept negative coordinates that satisfy schema integer constraints"` explicitly documents that negative coordinates are accepted by the schema. However, negative coordinates (e.g., `x: -1`) are outside the playfield grid and would cause game engine errors at runtime. Accepting them at the API level and then failing at the engine level is a potential footgun. The test currently just confirms the schema permits them; there is no note about whether this is intentional or a schema gap.

**Impact:** Low for the test suite itself — the test is correct. The issue is schema design, not test quality, but noting it here as it represents a contract that could mislead.

**Suggestion:** Add an inline comment in the test explaining that negative coordinates are intentionally allowed by the schema (e.g., to support relative positioning or future coordinate systems), and add a corresponding note in the schema file. If they are not intentional, add a `minimum: 0` constraint to the JSON schema and update this test to expect rejection.

______________________________________________________________________

## Coverage Summary

**gsnake-core (Rust engine):** Well covered. The `engine.rs` inline tests cover the main gameplay loop comprehensively (movement, food, gravity, collision, win/loss, edge cases). The `gravity.rs` unit tests and property-based tests in `gravity_property_tests.rs` are strong. The `stone_mechanics.rs` unit tests are solid. **Gap:** Extended `CellType` variants (`FloatingFood`, `FallingFood`, `Stone`, `Spike`) missing from contract serialization and round-trip tests.

**gsnake-web (TypeScript, contract layer):** `types.test.ts` is the best test in the project — real runtime type guards with negative cases for all variants. `fixtures.test.ts` is solid. `enums.test.ts` is tautological and should be reconsidered. **Gap:** No test for `WasmGameEngine.loadLevel()`, no coverage of `AllComplete` state transition from the engine test, and no test for `countSnakeSegments` with edge case frames.

**gsnake-web (Svelte UI components):** Good coverage of `KeyboardHandler`, `RestartButton`, `Header`, `Cell`, and `LevelUiFlows`. `CompletionTracker` has gaps at error paths. **Gap:** `LevelComplete` keyboard behavior, `r`/`q` in playing state, corrupted localStorage content.

**gsnake-editor (TypeScript):** `types.test.ts`, `EditorLayout.gridInteractions.test.ts`, `EditorLayout.saveLoad.test.ts`, and `server.test.ts` are solid. The `integration.test.ts` file is a dead weight. `levelDefinitionSchemaDrift.test.ts` is thorough and valuable. **Gap:** `buildLevelExportPayload` edge cases (null exit, empty snake), loaded level rendering coverage for all entity types.

**E2E (Playwright):** `level-editor.spec.ts` is the strongest E2E test — comprehensive create-save-load-test workflow with robust wait strategies. `gravity.spec.ts` covers gameplay flow but couples to level 1's specific data. `contract.spec.ts` has good coverage of the frame structure contract but misses extended CellType variants and uses one timing anti-pattern. **Gap:** No E2E test for `AllComplete` state, no cross-browser testing (Chromium only), API server not in `webServer` config.

______________________________________________________________________

## Positive Observations

1. **Property-based gravity tests** (`gravity_property_tests.rs`): Using `proptest` for the stone-gravity and snake-gravity invariants is exactly the right tool for this domain. The three properties (in-bounds, idempotent, deterministic for equivalent states) are well-chosen and would catch a broad class of implementation bugs.

1. **Schema drift protection** (`levelDefinitionSchemaDrift.test.ts` in both editor and web): The parity-assertion pattern — compile both the canonical AJV validator and the local guard, run the same inputs through both, and assert their Boolean results match — is an elegant way to keep distributed validators synchronized without coupling implementation details.

1. **WasmGameEngine mock pattern** (`WasmGameEngine.test.ts`): The use of `vi.hoisted` for shared mock state with an in-module `MockRustEngine` class is the correct approach for testing the TypeScript wrapper without a real WASM binary. The event-stream assertion (`events.map(e => e.type)` toEqual) gives deterministic ordering guarantees.

1. **`data-element-id` selectors in E2E tests**: Using explicit `data-element-id` attributes instead of CSS class names or text content for Playwright locators makes the E2E tests resilient to styling and copy changes. This pattern is consistently applied across `gravity.spec.ts` and `level-editor.spec.ts`.

1. **`level-editor.spec.ts` service-readiness helper**: The `waitForService` function with configurable acceptable status codes and retry count is a clean, reusable pattern for robust E2E service bootstrapping that avoids fixed sleeps.

1. **Thorough server API tests** (`server.test.ts`): The CORS policy tests, structured error response shape assertions, `405 Method Not Allowed` coverage, and malformed JSON body test together form a complete HTTP contract test for the editor API.
