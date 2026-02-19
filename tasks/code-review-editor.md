# Code Review: Level Editor

## Summary

The level editor is well-structured for its scope: a thin Svelte SPA backed by an Express API that acts as a relay for game-testing. The JSON schema validation on the server side is thorough and the test coverage is meaningful. However, there are several correctness bugs that can produce silently broken level exports (snake segments duplicated in the grid model, undo bypassed for a destructive palette action, coordinate-bounds not enforced on import), security concerns around unconstrained SVG injection via `{@html}` and an origin-whitelist that is re-read on every request (creating a TOCTOU-style inconsistency window), and multiple data-loss UX paths where the user receives no warning before work is discarded.

## Findings

### [SEVERITY: High] Switching to the snake tool silently deletes the existing snake without an undo entry

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:173-196`
**Issue:** `handleEntitySelect` immediately mutates `snakeSegments` and `cells` to erase the current snake whenever the user selects the snake tool while a different tool was active. This mutation bypasses the command/undo system entirely — no `captureState` / `executeCommand` call is made — so the destruction cannot be undone with Ctrl+Z.
**Impact:** A user who accidentally clicks "Snake" in the palette after building a snake loses their entire snake layout with no recovery path. The comment in the code says "FIX BUG #2" but the fix itself introduced a new, more severe data-loss regression.
**Suggestion:** Either remove the auto-clear behaviour (require the user to explicitly delete segments with Shift+click) or, if the auto-clear is intentional, wrap the state change inside `createCellModificationCommand` / `executeCommand` so it is undoable.

______________________________________________________________________

### [SEVERITY: High] File-input DOM element leaks when user cancels the file picker

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:411-476`
**Issue:** `handleLoad` appends an `<input type="file">` to `document.body` and registers an `onchange` handler that removes the element in its `finally` block. However, if the user opens the file picker and cancels without selecting a file, `onchange` never fires. The hidden `<input>` element remains attached to the DOM permanently, accumulating on every cancelled invocation.
**Impact:** Each cancelled load attempt leaks a hidden DOM node. On long editing sessions the accumulation is benign, but it also means the cleanup path in the `finally` block cannot be relied upon — if the element has already been removed for some other reason `removeChild` will throw an uncaught `NotFoundError`.
**Suggestion:** Use `input.addEventListener('cancel', () => document.body.removeChild(input))` (supported in modern browsers) or clean up in both the `change` handler and a one-shot `focus` event on `window`. Alternatively, reuse a single file input element across invocations.

______________________________________________________________________

### [SEVERITY: High] `handleTest` duplicates level-building logic divergently from `buildLevelExportPayload`

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:517-594`
**Issue:** `handleTest` manually constructs the level JSON object by iterating over `cells` and converting snake segments, reproducing the exact logic that already exists in `buildLevelExportPayload` (`levelModel.ts`). The two implementations have diverged in a subtle but important way: `handleTest` uses a manual string-capitalize trick for the snake direction (`snakeDirection.charAt(0).toUpperCase() + snakeDirection.slice(1)`) rather than the `toRuntimeSnakeDirection` switch in `levelModel.ts`. Additionally, the test payload omits the `difficulty` field entirely, while the schema permits it and `buildLevelExportPayload` always includes it.
**Impact:** If `levelModel.ts` changes its direction conversion or entity ordering, `handleTest` will silently produce a different (potentially invalid) payload than what the user would export. The missing `difficulty` in the test payload also means the game sees a different level definition during testing than what will be exported.
**Suggestion:** Replace the hand-rolled object in `handleTest` with a call to `buildLevelExportPayload`, using a fixed test-only `levelId` and a default or UI-selected difficulty. This keeps the two paths in sync automatically.

______________________________________________________________________

### [SEVERITY: High] Negative coordinates accepted by the server schema but silently dropped on import

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:68-99` and `/home/nntin/git/gSnake/contracts/level-definition.schema.json:103-115`
**Issue:** The JSON schema allows position coordinates as low as `-2147483648`. The server test at line 165 of `server.test.ts` explicitly confirms that a payload with `snake: [{ x: -1, y: -2 }]` is accepted with HTTP 200. However, when the editor loads such a level, `placeEntitiesFromLevelData` silently filters out any position where `pos.y < 0 || pos.x < 0`, meaning entities at negative coordinates are dropped without any user notification. The snake and exit entries disappear; the level loads successfully but is corrupted.
**Impact:** A level file produced by or for a game engine that supports negative coordinates will silently lose entities on round-trip through the editor. The user sees a partial grid with no error, may re-save, and permanently destroys the original data.
**Suggestion:** Either (a) reject negative coordinates at load time with a clear error message, or (b) if the editor intends to support negative coordinates, expand the grid model to accommodate them. A minimum: display a warning toast listing any entities that were out-of-bounds and dropped.

______________________________________________________________________

### [SEVERITY: Medium] `{@html spriteContent}` injects unvalidated SVG fetched from a configured URL

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/SpriteLoader.svelte:7-19`
**Issue:** `SpriteLoader` fetches the sprite sheet from `spritesUrl` (provided by the `gsnake-web-ui` package) and injects the raw response text into the DOM via `{@html spriteContent}`. There is no content-type check, no sanitization, and no integrity verification on the fetched content.
**Impact:** If `spritesUrl` ever resolves to an attacker-controlled URL (e.g., through a compromised dependency, a misconfigured Vite alias, or a network interception in a non-TLS dev environment), arbitrary HTML/script can be injected into the editor page. The eslint-disable comment acknowledges the risk but provides no mitigation.
**Suggestion:** Add a `Content-Type: image/svg+xml` check on the response before injecting. Consider using `DOMParser` to parse the SVG and only insert the parsed SVG element, which strips scripts. At minimum, document the trust assumption (same-origin controlled asset) so future maintainers understand the boundary.

______________________________________________________________________

### [SEVERITY: Medium] CORS allowed-origins list is re-evaluated on every request

**File:** `/home/nntin/git/gSnake/gsnake-editor/server.ts:37-44`
**Issue:** Inside the CORS `origin` callback, `resolveAllowedCorsOrigins()` is called with no arguments on every incoming request. `resolveAllowedCorsOrigins` reads `process.env[ALLOWED_ORIGINS_ENV]` each time. This means that if the environment variable changes at runtime (possible in some hosting environments or through test harness mutation), the effective CORS policy changes mid-flight without a server restart.
**Impact:** In the test suite, `beforeEach` deletes and reassigns `process.env.GSNAKE_EDITOR_ALLOWED_ORIGINS`, which works correctly with the current implementation, but demonstrates that the policy is not frozen at startup. In production, a race between a request and an env mutation could produce non-deterministic CORS decisions. More concretely, caching the resolved list at startup would be simpler, faster, and more predictable.
**Suggestion:** Resolve and freeze the allowed-origins list once at module load time (e.g., `const ALLOWED_ORIGINS = resolveAllowedCorsOrigins()`) and reference the constant inside the callback. If runtime reconfiguration is genuinely needed, make that an explicit, documented behaviour.

______________________________________________________________________

### [SEVERITY: Medium] Load validation in `App.svelte` and `EditorLayout.svelte` is duplicated and inconsistently applied

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/App.svelte:29-45` and `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:429-445`
**Issue:** Both the landing-page load path (`App.svelte::handleLoadExisting`) and the in-editor load path (`EditorLayout.svelte::handleLoad`) independently re-implement the same six-point validation: `isValidLevelId`, `gridSize` shape, `snake` non-empty array, `snakeDirection` presence, and grid dimension bounds (5–50). Any change to validation rules (e.g., changing the max grid size) must be updated in two places, and the two copies are already slightly divergent: `App.svelte` does not reset undo/redo history before loading (because loading from the landing page creates a fresh editor, which is correct), while `EditorLayout.svelte` does. The duplication creates maintenance risk.
**Impact:** A future change to validation logic applied to only one path will silently accept invalid levels through the other path.
**Suggestion:** Extract a `validateLevelFileData(data: unknown): asserts data is LevelData` function (or one returning a result type) into a shared module (e.g., `src/lib/levelValidation.ts`) and call it from both sites.

______________________________________________________________________

### [SEVERITY: Medium] Reactive grid-reset runs on every render, not just when dimensions change

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:41-51`
**Issue:** The reactive statement `$: if (gridWidth || gridHeight) { cells = ... }` runs whenever either `gridWidth` or `gridHeight` is read as part of any reactive evaluation chain, not just when the values genuinely change. In Svelte 4 the `$:` block re-runs whenever any of its dependencies are accessed. Because `gridWidth` and `gridHeight` are also referenced in the template and in `handleLoad`, any update that touches those variables triggers a full grid wipe.
**Impact:** Specifically, during `handleLoad`, `gridWidth` and `gridHeight` are set to the loaded level's dimensions on lines 452-453, which correctly triggers the reactive reset before `tick()`. However, the logic depends on the precise ordering of reactive evaluation — if another reactive statement or a future refactor reads `gridWidth` without triggering a `tick()` before `placeEntitiesFromLevelData`, the grid will be wiped after entities are placed, producing a blank grid.
**Suggestion:** Use a dedicated function `resetGrid(width, height)` called explicitly at the right points instead of relying on reactive auto-triggering. This makes the control flow explicit and eliminates the order-dependency hazard.

______________________________________________________________________

### [SEVERITY: Medium] `handleTest` posts to a hardcoded `localhost:3001` URL

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:598`
**Issue:** The fetch call in `handleTest` targets `'http://localhost:3001/api/test-level'` as a string literal. The game URL for the new tab is correctly read from `import.meta.env.VITE_GSNAKE_WEB_URL`, but the API server URL is not configurable.
**Impact:** In any deployment that runs the editor API on a different port or hostname (including Docker setups, CI environments, or remote development), the test feature silently fails with a network error. The user sees a generic "Make sure the test server is running on port 3001" message that may be misleading.
**Suggestion:** Introduce a `VITE_GSNAKE_API_URL` (or similar) environment variable with `localhost:3001` as the default, mirroring the pattern already used for `VITE_GSNAKE_WEB_URL`.

______________________________________________________________________

### [SEVERITY: Medium] Snake-segment auto-clear in `handleEntitySelect` does not reset `snakeDirection`

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:177-193`
**Issue:** When the snake-tool-switch clears all snake segments, `snakeDirection` is left at whatever value the user set previously. This is minor but inconsistent: after the clear the editor shows an empty grid with a dangling direction that does not correspond to any snake.
**Impact:** The exported level will contain the old direction even though the snake was completely redrawn, which may be intentional (user keeps their direction choice) or surprising. Because the clear is not in the undo stack, undoing back to the pre-clear state also does not restore the direction if the user had changed it.

______________________________________________________________________

### [SEVERITY: Medium] `handleEntitySelect` snake-clear does not record undo — snake-segment index inconsistency after clear

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:181-190`
**Issue:** When clearing segments, the loop sets `entity = null` and `isSnakeSegment = false` on each cell but does not reset `snakeSegmentIndex` (the field is not touched in the loop body). After the clear, `snakeSegments` is set to `[]`, so no cell should still have an index — but if Svelte's reactivity has not yet flushed, a transient state can exist where cells have `entity = null` but a stale `snakeSegmentIndex` value.
**Impact:** Low probability in practice because the cells are reset before any new render; however, this is a fragile pattern. If a future change reads `snakeSegmentIndex` from a cell without also checking `isSnakeSegment`, it will see stale data.
**Suggestion:** Explicitly set `snakeSegmentIndex = undefined` in the clear loop alongside the other fields.

______________________________________________________________________

### [SEVERITY: Medium] `handleTest` does not validate the level before posting (no snake check)

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:517-637`
**Issue:** `handleSave` calls `validationData` checks (snake count, food count, exit presence) before opening the save modal and displaying warnings. `handleTest` performs no equivalent check. A user can click "Test" with an empty grid (no snake, no food, no exit) and a payload with `snake: []` will be posted to the server. The server-side schema does not require the `snake` array to be non-empty (the schema has no `minItems` constraint on `snake`), so the server accepts it with HTTP 200 and the game engine receives an invalid level.
**Impact:** The game engine may crash or behave undefined when it receives a level with an empty snake array.
**Suggestion:** Apply the same pre-flight validation used in `handleSave` before the `fetch` call in `handleTest`, and surface warnings to the user before proceeding.

______________________________________________________________________

### [SEVERITY: Medium] JSON schema does not enforce `snake` array minimum length or coordinate bounds relative to grid

**File:** `/home/nntin/git/gSnake/contracts/level-definition.schema.json:50-53`
**Issue:** The `snake` field is defined as `"type": "array"` with no `minItems` constraint, so an empty snake array passes schema validation. Additionally, no schema-level cross-validation enforces that snake or food coordinates fall within `[0, gridSize.width) × [0, gridSize.height)`. Both checks are left to the game engine.
**Impact:** The server's `POST /api/test-level` endpoint accepts and stores levels with zero snake segments. When the game engine fetches such a level, it may panic or produce undefined behaviour.
**Suggestion:** Add `"minItems": 1` to the `snake` array definition in the schema to enforce at least a head segment at the contract boundary.

______________________________________________________________________

### [SEVERITY: Low] `difficulty` field is not validated at the schema level

**File:** `/home/nntin/git/gSnake/contracts/level-definition.schema.json:30-32`
**Issue:** `difficulty` is declared as `"type": "string"` with no `enum` constraint, so any string value (e.g., `"insane"`, `""`, `"null"`) passes schema validation. The TypeScript types in `levelModel.ts` correctly restrict `Difficulty` to `"easy" | "medium" | "hard"`, but that constraint is not reflected in the canonical contract.
**Impact:** A level produced by an external tool with an unrecognized difficulty string will pass server validation and be stored, potentially causing unexpected behaviour in the game engine's difficulty logic.
**Suggestion:** Add `"enum": ["easy", "medium", "hard"]` to the `difficulty` property in the schema.

______________________________________________________________________

### [SEVERITY: Low] `isMainModule` detection logic is fragile on some platforms

**File:** `/home/nntin/git/gSnake/gsnake-editor/server.ts:144-145`
**Issue:** The check `import.meta.url.endsWith(process.argv[1]) || import.meta.url === \`file://${process.argv[1]}\``uses string comparison between`import.meta.url`(which may include`file://`and potentially percent-encoded characters) and`process.argv[1]`(a raw OS path). On Windows or when the file is run with a relative path, the comparison can fail in either direction. **Impact:** On Windows the server may fail to start when run directly, or start unintentionally when imported in tests. In practice the tests pass because they import the module without executing it as the main entry, but the detection is not robust. **Suggestion:** Use the established pattern`if (process.argv[1] === fileURLToPath(import.meta.url))\` which correctly handles encoding and cross-platform path normalization.

______________________________________________________________________

### [SEVERITY: Low] `EntityPalette.svelte` constructs drag-image with inline `innerHTML` from controlled data

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EntityPalette.svelte:41`
**Issue:** `dragImage.innerHTML = \`<svg viewBox="0 0 32 32" width="28" height="28"><use href="#${entity.spriteId}"></use></svg>\``sets innerHTML where`entity.spriteId`is a value from the statically defined`entities`array. Because`entities`is a hardcoded constant, the`spriteId`values are safe. However, if`entities`were ever made dynamic (e.g., loaded from a config file), this would become an XSS vector. **Impact:** No immediate risk given the current static list, but the pattern is fragile. **Suggestion:** Use`document.createElementNS`/`setAttribute\` to build the SVG element programmatically instead of innerHTML, which eliminates the XSS risk regardless of how the data is sourced.

______________________________________________________________________

### [SEVERITY: Low] `handleNewLevel` in EditorLayout navigates away with no unsaved-work warning

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:403-406`
**Issue:** Clicking "New Level" immediately dispatches the `newLevel` event, which causes `App.svelte` to hide the editor and reset state. There is no confirmation dialog warning the user that unsaved changes will be lost.
**Impact:** A user who has spent time building a level and accidentally clicks "New Level" loses all their work without any opportunity to recover it.
**Suggestion:** Before dispatching `newLevel`, check whether the undo stack is non-empty (a proxy for "unsaved changes exist") and display a confirmation dialog. Only dispatch if the user confirms.

______________________________________________________________________

### [SEVERITY: Low] `SpriteLoader.svelte` does not handle fetch errors

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/SpriteLoader.svelte:7-10`
**Issue:** The `onMount` async function fetches the sprite sheet and sets `spriteContent` from `response.text()`. There is no `try/catch`, no check on `response.ok`, and no fallback when the fetch fails. If the sprite URL is unavailable (e.g., during development before the vendor assets are built), the promise rejects as an unhandled rejection and the editor silently renders with no sprites.
**Impact:** The user sees a blank entity palette and unlabelled grid cells with no indication that anything is wrong.
**Suggestion:** Add a `try/catch` block and display a console error or a toast notification when sprite loading fails, so the developer/user understands the source of the visual degradation.

______________________________________________________________________

### [SEVERITY: Low] `handleLoad` file input is appended to `document.body` before the click, introducing a brief visible-but-hidden element

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/EditorLayout.svelte:474-476`
**Issue:** The hidden `<input>` is appended to `document.body` before `input.click()` is called. Although `display: none` prevents visual rendering, the element exists in the accessibility tree momentarily. More importantly, if `input.click()` throws (which it can in sandboxed environments), the element is left attached without any cleanup because the `onchange` handler (and its `finally` block) never runs.
**Impact:** Minor accessibility noise and a potential leak in sandboxed environments.
**Suggestion:** Wrap the `appendChild` + `click` in a try/catch that removes the element on failure.

______________________________________________________________________

### [SEVERITY: Low] `GridSizeModal.svelte` does not validate that width/height are integers

**File:** `/home/nntin/git/gSnake/gsnake-editor/src/lib/GridSizeModal.svelte:20-28`
**Issue:** The validation in `handleCreate` only checks `width < 5 || width > 50 || height < 5 || height > 50`. HTML `<input type="number">` allows fractional values; a user who manually types `7.5` will have `width = 7.5` after Svelte's `bind:value`. The check passes (7.5 is between 5 and 50) and a grid with fractional dimensions is created.
**Impact:** The grid is created with `gridWidth = 7.5`, which causes `Array.from({ length: 7.5 }, ...)` to be called. JavaScript coerces this to `Array.from({ length: 7 })`, silently truncating. The stored `gridWidth` is `7.5`, so the exported `gridSize.width` will be `7.5`, failing the schema's `"type": "integer"` constraint.
**Suggestion:** Add `Number.isInteger(width) && Number.isInteger(height)` to the validation check, and/or use `step="1"` on the inputs.

______________________________________________________________________

## Positive Observations

- The JSON schema validation on the server (`levelDefinitionValidator.ts`) is thorough, using AJV 2020-12 with `allErrors: true` and `additionalProperties: false`, which provides strong contract enforcement for the test-level API endpoint.
- The schema-drift test (`levelDefinitionSchemaDrift.test.ts`) is an excellent safeguard that detects when the editor's vendored schema diverges from the canonical contract schema, preventing silent API mismatches.
- The undo/redo implementation using the command pattern (`createCellModificationCommand`, `captureState`, `restoreState`) is clean and correct for all cell-click and cell-drop paths; deep-copying cells and snakeSegments correctly avoids shared-reference bugs.
- `buildLevelExportPayload` in `levelModel.ts` is a pure function with a single code path for serialising the grid, making it easy to test in isolation. The unit tests in `types.test.ts` cover its contract thoroughly including field ordering.
- The server's CORS middleware correctly refuses origins not on the allowlist and returns a structured `403` rather than swallowing the error, making CORS rejections visible to callers.
- `resolveAllowedCorsOrigins` is sensibly designed for testability: it accepts a `rawOrigins` parameter with a default that reads from the environment, allowing tests to override it without monkey-patching globals.
- The `totalFood` field is derived deterministically from `food.length + floatingFood.length + fallingFood.length` in both `buildLevelExportPayload` and `handleTest`, keeping it consistent with the semantic contract.
- Optional entity arrays (`floatingFood`, `fallingFood`, `stones`, `spikes`) are correctly handled with `|| []` defaults on import, maintaining backward compatibility with levels that pre-date those fields.
- The `generateLevelId` function correctly maps the `Uint32Array` zero value to `1` to keep IDs truthy, and the `RandomValuesProvider` interface allows full deterministic testing without mocking `crypto`.
