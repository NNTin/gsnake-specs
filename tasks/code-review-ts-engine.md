# Code Review: TypeScript Engine Layer

## Summary

The TypeScript engine wrapper and state management layer are well-structured for a game of this scope. The dominant risks are: (1) `WasmGameEngine` sets `this.initialized = true` only after the full async init sequence completes, so a concurrent second `init()` call during the await chain will pass the guard and proceed in parallel rather than being rejected; (2) the `onFrame` callback registered on the previous `RustEngine` instance is not cleaned up before constructing a new one in `loadLevelByIndex`, which can cause stale frame emissions if the WASM engine ever fires the old callback after a reset or level transition; (3) `KeyboardHandler` subscribes to the Svelte store in its constructor but only has a single-listener slot in `connectGameEngineToStores`—calling `connectGameEngineToStores` more than once silently stacks listeners; and (4) `GameStatus` in the type definitions includes `"LevelComplete"` but `KeyboardHandler.handleKeyPress` has no `case "LevelComplete"` branch in its switch statement, meaning all keypresses during level-complete state are silently dropped. Overall code quality is good, error normalization is consistently applied, and the test suite covers the most important paths, but the gaps noted below warrant attention before extending the codebase.

______________________________________________________________________

## Findings

### [SEVERITY: High] Double-init race condition: `initialized` flag is set after the await chain, not before it

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/WasmGameEngine.ts:26-78`
**Issue:** The `initialized` guard at line 26 is checked at the start of `init()` and `this.initialized = true` is set at line 77 — after the entire async init sequence completes. If `init()` is called twice in quick succession (e.g., the component mounts twice due to HMR or a framework re-render), the second call passes the guard on line 26 because `initialized` is still `false` during the first call's awaits. Both calls then run `await init_wasm()`, `getLevels()`, and `loadLevelByIndex()` concurrently, leaving `this.wasmEngine`, `this.levels`, and `this.currentLevelIndex` in a non-deterministic end state.
**Impact:** In development with HMR or any scenario where the component re-mounts while init is still in flight, two WASM modules initialise and two sets of events are emitted to every listener. The store values (`frame`, `gameState`, `level`) end up reflecting whichever init call resolved last, while the other call's `RustEngine` instance is leaked.
**Suggestion:** Set `this.initialized = true` (or a separate `this.initInProgress = true`) immediately inside the guard, before the first `await`, and clear it on failure.

______________________________________________________________________

### [SEVERITY: High] `onFrame` callback from the old `RustEngine` instance is not unregistered before replacement

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/WasmGameEngine.ts:98-130`
**Issue:** `loadLevelByIndex` overwrites `this.wasmEngine` with a new `RustEngine` instance at line 99. The old instance's `onFrame` callback (registered at line 110) is never explicitly unregistered. The WASM comment at line 117 states that "onFrame callbacks only fire on processMove", implying the old instance won't fire after replacement. However, this is a behavioural assumption about the Rust WASM module that is not enforced at the TypeScript layer. If a future WASM change fires the callback asynchronously after `processMove` returns (e.g., due to internal async scheduling), the stale callback closure still holds a reference to `this` and will call `this.handleFrameUpdate`, injecting a frame from the wrong level into the store. Additionally, the old `RustEngine` object is never explicitly freed; if the WASM module uses reference-counted or manually managed memory, this leaks it.
**Impact:** Stale frame events from a previous level could overwrite `frame` and `gameState` stores with data from the wrong level mid-gameplay, causing silent visual corruption and incorrect score display.
**Suggestion:** Before constructing a new `RustEngine`, null-out the old one and/or call a cleanup/free method if the WASM API exposes one. At minimum, wrap the old callback registration so the stale instance's callback becomes a no-op after replacement (e.g., use a generation counter and discard callbacks from past generations inside `handleFrameUpdate`).

______________________________________________________________________

### [SEVERITY: High] Missing `"LevelComplete"` case in `KeyboardHandler` switch statement

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/KeyboardHandler.ts:62-73`
**Issue:** `GameStatus` (defined in `types/models.ts:23-27`) has four variants: `"Playing"`, `"GameOver"`, `"LevelComplete"`, and `"AllComplete"`. The switch statement in `handleKeyPress` handles `"Playing"`, `"GameOver"`, and `"AllComplete"` but has no `case "LevelComplete":` branch. TypeScript does not warn about this because the switch does not have an exhaustiveness check (no `default` that asserts `never`). When the status is `"LevelComplete"`, all keydown events are silently discarded: direction keys have no effect, and `r`/`q` do not advance or reset the level.
**Impact:** After a level is completed, the player cannot use the keyboard at all until the `LevelComplete` status is externally cleared (which only happens when the next level loads). Any workflow that relies on keyboard input to advance — e.g., "press any key to continue" — is broken in this state. This is a logic bug reachable from normal gameplay on every level completion.
**Suggestion:** Add a `case "LevelComplete":` handler that at minimum advances to the next level on a directional or confirmation key, and add a TypeScript exhaustiveness guard (`default: { const _exhaustive: never = this.currentStatus; }`) to catch future status additions at compile time.

______________________________________________________________________

### [SEVERITY: Medium] `connectGameEngineToStores` accumulates listeners on every call

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/stores/stores.ts:49-68`
**Issue:** `addEventListener` in `WasmGameEngine` (line 208 in `WasmGameEngine.ts`) pushes listeners into an unbounded array. `connectGameEngineToStores` always calls `addEventListener` without any deduplication or removal mechanism. There is no corresponding `removeEventListener`. If `connectGameEngineToStores` is called more than once on the same engine instance — e.g., due to Svelte HMR during development, or if a test does not properly isolate the engine — the same event will be dispatched to multiple listener instances, causing double (or more) store updates per event.
**Impact:** In development with HMR, stores are updated multiple times per frame, causing spurious re-renders and making `snakeLength` and `frame` inconsistent. In production this is less likely to trigger but remains a latent hazard if the component lifecycle is changed.
**Suggestion:** Expose a `removeEventListener` on `WasmGameEngine` and return a cleanup function from `connectGameEngineToStores`. Alternatively, make the listener slot a single property instead of an array, since the current architecture only ever needs one consumer.

______________________________________________________________________

### [SEVERITY: Medium] `isInitializing` can be left as `true` permanently if `onDestroy` fires before `init()` resolves

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/components/App.svelte:17-71`
**Issue:** In `App.svelte`, `isMounted` is checked in the `finally` block at line 68 before setting `isInitializing = false`. If the component is destroyed while `gameEngine.init()` is still running, `isMounted` becomes `false` and `isInitializing` is never set back to `false`. This affects the component's own render, but more importantly the `gameEngine` continues to run — emitting `levelChanged` and `frameChanged` events into the Svelte stores — after the component that owns it has been destroyed and `keyboardHandler.detach()` has been called. Those store writes are observable by any other component still subscribed.
**Impact:** In development (HMR), every hot-reload that interrupts init leaves a zombie `WasmGameEngine` writing to global stores until the page is refreshed.
**Suggestion:** Track an `abortSignal` or cancelled flag inside `onMount` and have the engine's event listener do nothing (or detach itself) after `onDestroy` fires. The `isMounted` guard in the `finally` block is correct for the local UI state, but the engine-to-store listener also needs to be torn down.

______________________________________________________________________

### [SEVERITY: Medium] `normalizeStartLevel` silently coerces negative decimal strings to `1` via two separate code paths with subtle differences

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/WasmGameEngine.ts:140-156`
**Issue:** The string path uses `/^\d+$/` which only matches non-negative integers. The string `"-1"` fails the regex and correctly returns `1`. However, the number path at line 142 checks `Number.isInteger(startLevel) && startLevel > 0` — `Number.isInteger(-1)` is `true` and `-1 > 0` is `false`, so `-1` also returns `1` correctly. The logic is correct but the two paths are inconsistently documented: the comment in `CLAUDE.md` says "non-positive numbers" should resolve to 1, but a caller passing the float `2.7` as a number will hit `Number.isInteger(2.7)` which is `false` and fall to the `return 1` at line 155, which is correct but non-obvious since `2.7` might be a reasonable off-by-one in a URL parser. The function lacks inline documentation explaining why floats-as-numbers default to 1.
**Impact:** Low risk in practice because callers pass `URLSearchParams.get()` strings (always string type) or hardcoded integers. However, the inconsistency between the `typeof startLevel === "number"` path and the `string` path creates a maintenance hazard where a future caller passing a parsed float could be surprised.
**Suggestion:** Add a brief comment on the number branch explaining that fractional numbers are intentionally rejected, consistent with the string path's `^\d+$` regex. Consider consolidating by always coercing to string before parsing.

______________________________________________________________________

### [SEVERITY: Medium] `CompletionTracker.markCompleted` does not guard against `localStorage.setItem` throwing

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/CompletionTracker.ts:23-30`
**Issue:** `getCompletedLevels` wraps `localStorage` operations in a try/catch (line 13). `markCompleted` calls `getCompletedLevels()` (which is safe) but then calls `window.localStorage.setItem(STORAGE_KEY, ...)` at line 28 without a try/catch. `setItem` can throw `DOMException: QuotaExceededError` in private browsing mode or when the storage quota is full. If this happens, the exception escapes to the caller in `App.svelte` (line 86), which has no catch block around the reactive statement, so the exception propagates to the Svelte scheduler and is logged as an unhandled error. Worse, the `updated` array is already computed in memory but not persisted, so the return value and the persisted state are out of sync.
**Impact:** In private browsing mode (where `localStorage` quota is often 0 bytes) or on storage-constrained devices, every level completion produces an uncaught exception. Because the reactive block in `App.svelte` that calls `markCompleted` runs outside a try/catch, this could corrupt Svelte's reactive graph update.
**Suggestion:** Wrap `setItem` in a try/catch in `markCompleted`, consistent with `getCompletedLevels`.

______________________________________________________________________

### [SEVERITY: Medium] `handleContractError` double-logs when the error is already a `ContractError`

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/WasmGameEngine.ts:221-240`
**Issue:** `handleContractError` calls `normalizeContractError` which returns the original `ContractError` unchanged if `isContractError` returns true (line 256). It then emits `engineError` AND logs via `console.error` at line 233-235. The `console.error` at line 239 is only reached when `contractError` is `null`, which only happens if `normalizeContractError` returns `null` — but looking at `normalizeContractError`, it always returns a non-null value (it falls back to constructing a new object). So the `console.error(fallbackMessage, error)` on line 239 is dead code: it is unreachable because `contractError` will never be `null`.
**Impact:** Dead code is misleading to future maintainers — it implies there is a logging code path that bypasses the ContractError event, but it is never reached. The real path always emits `engineError` AND logs, which is correct, but the structure obscures this.
**Suggestion:** Remove the dead `console.error(fallbackMessage, error)` branch on line 239, or change `normalizeContractError` to explicitly return `null` for a meaningful case and document that case.

______________________________________________________________________

### [SEVERITY: Medium] `getLevels()` return value not guarded by WASM type cast in `init`

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/WasmGameEngine.ts:52`
**Issue:** The expression `getLevels() as unknown as LevelDefinition[]` uses a double cast (`as unknown as T`) to bypass TypeScript's type checker. This pattern is a red flag: it asserts that the value returned by the WASM binding conforms to `LevelDefinition[]` without any runtime validation. If the Rust side changes the shape of `LevelDefinition` (e.g., adds a required field, renames a property, or changes a type), this cast will silently succeed at compile time but produce runtime failures when the game engine tries to access the missing properties.
**Impact:** Any schema drift between the Rust type definitions and the TypeScript `LevelDefinition` type (which is generated by `ts-rs` and therefore should match, but is not validated at this boundary at runtime) will surface as obscure runtime errors rather than clear contract violations, making debugging harder.
**Suggestion:** Pass the result of `getLevels()` through the same `isLevelArray` guard used in `fetchCustomLevels` (from `levelDefinitionGuard.ts`). If validation fails, emit an `initializationFailed` error with a descriptive message pointing to the WASM/TS type mismatch.

______________________________________________________________________

### [SEVERITY: Low] `handlePlayingState` calls `event.preventDefault()` conditionally and redundantly

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/KeyboardHandler.ts:89-95`
**Issue:** Arrow key events have `event.preventDefault()` called unconditionally at lines 44-55. WASD keys do not appear in the unconditional list at line 44-55. In `handlePlayingState` at line 92, for WASD keys the code checks `!event.defaultPrevented` before calling `preventDefault()`. This is correctly ordered but relies on the outer guard having run. The condition is vacuously correct for arrow keys (they are already prevented) but slightly convoluted: the fact that `event.defaultPrevented` can only be `false` at that point for WASD keys is non-obvious. If someone later adds WASD to the unconditional prevent list, this branch becomes dead.
**Suggestion:** Move WASD keys into the unconditional `preventDefault` list at lines 44-55 for clarity, and simplify line 92-94 to an unconditional `event.preventDefault()`.

______________________________________________________________________

### [SEVERITY: Low] `KeyboardHandler` stores `currentStatus` via Svelte store subscription, creating a tight coupling between keyboard handling and the store module

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/KeyboardHandler.ts:28-31`
**Issue:** `KeyboardHandler` directly imports and subscribes to `gameState` from `../stores/stores`. This creates a module-level dependency from the engine layer into the store layer. The engine layer (under `engine/`) is otherwise independent of the store layer (under `stores/`). This circular-ish dependency (engine imports stores, stores imports engine) is currently handled only because `stores.ts` imports `WasmGameEngine` for typing purposes only (not for the store subscription direction), but it does make unit testing more brittle: the `KeyboardHandler.test.ts` file must set `gameState` to control the status, tying the test to the store implementation.
**Impact:** Low in practice. The current test covers the behavior correctly. The concern is architectural: if the store or its initial value changes (e.g., default status changes from `"Playing"` to something else), tests that don't call `gameState.set(...)` before instantiating `KeyboardHandler` will use the wrong initial status silently. The `beforeEach` in the test does set `gameState` correctly, so this is not currently a bug.
**Suggestion:** Accept the current design as intentional (per the CLAUDE.md pattern), but add a comment in `KeyboardHandler` explaining why it subscribes to the store directly rather than receiving status via a callback or constructor parameter.

______________________________________________________________________

### [SEVERITY: Low] `loadLevel` and `restartLevel` are thin public wrappers that add noise without adding safety

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/WasmGameEngine.ts:81-87`
**Issue:** `loadLevel(levelNumber)` calls `loadLevelByIndex(levelNumber - 1)` and `restartLevel()` calls `resetLevel()`. The `resetLevel` method in turn calls `loadLevelByIndex(this.currentLevelIndex)`. The chain `restartLevel() -> resetLevel() -> loadLevelByIndex()` is three hops deep for a single action. If `loadLevel(0)` is called from keyboard code, it translates to `loadLevelByIndex(-1)`, which throws an `Error` at line 91. This is caught correctly by callers that `await` it (e.g., `nextLevel`), but `loadLevel` is called from `KeyboardHandler` without `await` and without a catch block, so the rejection goes unhandled.
**Impact:** If `loadLevel(0)` is ever called (the keyboard handler always passes `1`, so this is unlikely in practice), the promise rejects silently. More generally, `loadLevel` and `restartLevel` are async but are called without `await` in `KeyboardHandler`, meaning any errors they produce (including the `loadLevelByIndex` bounds check) result in unhandled promise rejections that reach the browser console but not the `engineError` store.
**Suggestion:** Either: (a) make `KeyboardHandler` await these calls and handle rejections; or (b) add internal try/catch inside `loadLevel` and `restartLevel` that route errors through `handleContractError`, consistent with `processMove`.

______________________________________________________________________

### [SEVERITY: Low] `snakeLength` is derived by scanning the full grid every frame rather than from the game state

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/stores/stores.ts:70-79`
**Issue:** `countSnakeSegments` iterates over every cell in the `frame.grid` to count `SnakeHead` and `SnakeBody` cells. For a large grid this is O(width × height) per frame. The `GameState` struct already tracks `foodCollected` and `moves` directly; the snake length could be exposed by the Rust engine as a direct field in `GameState` instead of being derived client-side by scanning the entire grid.
**Impact:** Negligible for typical grid sizes (e.g., 20×20 = 400 cells). At very large grids this adds unnecessary CPU work on every move.
**Suggestion:** Consider requesting that the Rust engine expose `snakeLength` directly in `GameState`. Until then, the current approach is acceptable and is correctly tested.

______________________________________________________________________

### [SEVERITY: Low] `CompletionTracker.isCompleted` calls `getCompletedLevels` on every invocation, re-parsing `localStorage` every time

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/CompletionTracker.ts:19-21`
**Issue:** `isCompleted(levelId)` calls `this.getCompletedLevels()` which reads from `localStorage`, JSON-parses the string, and filters the result, all for a single lookup. In the current codebase this is called from Svelte reactive statements that re-evaluate on store changes. The `completedLevels` Svelte store in `stores.ts` already holds the parsed array, making `CompletionTracker.isCompleted` largely redundant in a UI context where the store is available.
**Impact:** Minor performance cost, negligible in practice. More importantly, the class has two sources of truth for "completed levels": the `localStorage` state (queried each time) and the `completedLevels` Svelte store. They are kept in sync by `App.svelte`, but nothing enforces that all callers use the store.
**Suggestion:** This is acceptable as-is for a utility class. If performance were a concern, memoize the last-read value or simply direct UI code to use the Svelte store instead of calling `isCompleted`.

______________________________________________________________________

## Positive Observations

- **Consistent error normalization.** Every `catch` block in `WasmGameEngine` routes through `handleContractError` → `normalizeContractError`, which guarantees that the `engineError` store always receives a well-typed `ContractError` object regardless of whether the WASM module throws a plain `Error`, a structured contract object, or an unexpected value.

- **Event listener isolation pattern.** Using an internal `listeners: GameEventListener[]` array in `WasmGameEngine` and a typed `GameEvent` discriminated union keeps the engine decoupled from Svelte reactivity. This makes the engine fully testable outside of a Svelte environment, as demonstrated by the clean `WasmGameEngine.test.ts` suite.

- **Initial frame emission design.** The explicit `getFrame()` + `handleFrameUpdate()` call at lines 120-122 of `WasmGameEngine.ts` correctly handles the documented WASM constraint that `onFrame` only fires on `processMove`. The comment and the `frame-emission.md` reference make this intent clear to future maintainers.

- **Start-level normalization completeness.** `normalizeStartLevel` handles all edge cases enumerated in the CLAUDE.md spec (NaN, empty string, decimals, negative values, out-of-range, null) and the scenario table in `WasmGameEngine.test.ts` exhaustively covers them. This is a well-guarded boundary.

- **`CompletionTracker` defensive `localStorage` usage.** `getCompletedLevels` wraps all `localStorage` access in try/catch and filters non-numeric entries, making it robust against corrupted storage. The `typeof window === "undefined"` guards make the class safe for server-side rendering contexts.

- **`KeyboardHandler` modifier key guard.** The early return on modifier combinations (ctrl, alt, shift, meta) at lines 35-37 is clean and correct, and is tested with a dedicated case. The `CLAUDE.md` note that this guard should be preserved even for movement keys is well-reasoned.

- **Store update ordering in `connectGameEngineToStores`.** On a `frameChanged` event, `frame`, `gameState`, `snakeLength`, and `engineError` are all updated atomically within a single synchronous listener call. Because Svelte batches updates within the same microtask, this avoids intermediate renders where `frame` has changed but `gameState` has not, preventing a class of visual glitches.

- **Test coverage of contract error passthrough.** The test at line 265 of `WasmGameEngine.test.ts` explicitly verifies that a structured `ContractError` thrown by the Rust engine is forwarded to the `engineError` event unchanged (not wrapped in a new object), which correctly validates the `isContractError` fast-path in `normalizeContractError`.
