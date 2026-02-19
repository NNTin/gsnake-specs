# Code Review: Svelte UI Components

## Summary

The Svelte UI layer is generally well-structured, with clean separation between game logic and rendering and a sensible store-driven reactive model. The most critical issues are accessibility failures: the `Modal` component lacks an accessible name (`aria-labelledby`), and the `LevelSelectorOverlay` is missing its dialog role, `aria-modal`, and an Escape-key dismiss path entirely. A secondary cluster of issues concerns UX state inconsistencies — the level selector overlay can phantom-reappear after a modal is dismissed, `GameCompleteModal` offers no clickable path to restart, and `SpriteLoader` has no error handling leaving the game grid silently invisible if the sprite fetch fails.

______________________________________________________________________

## Findings

### [SEVERITY: High] Modal dialog missing accessible name

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-ui/components/Modal.svelte:19`

**Issue:** The `<section>` rendered as `role="dialog"` has `aria-modal="true"` but no `aria-labelledby` or `aria-label`. The WAI-ARIA spec requires every dialog to have an accessible name so screen readers can announce it. The `header` slot is populated with an `<h2>` by every consumer (e.g., `GameOverModal`, `EngineErrorModal`), but the dialog element does not reference it.

**Impact:** Screen reader users navigating to any modal (Game Over, Engine Error, Level Load Failed, All Levels Complete) hear no modal title. The dialog cannot be identified by assistive technology, failing WCAG 2.1 criterion 4.1.2.

**Suggestion:** Generate a stable id for the header element and wire it up with `aria-labelledby`. A minimal fix: add `let headerId` generated via a counter or crypto, set `id={headerId}` on the `<header>`, and set `aria-labelledby={headerId}` on the `<section>`.

______________________________________________________________________

### [SEVERITY: High] LevelSelectorOverlay is not a dialog — missing role, aria-modal, and Escape dismiss

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/components/LevelSelectorOverlay.svelte:25-60`

**Issue:** The `<div class="level-overlay">` is a full-screen fixed backdrop containing a floating panel. It behaves like a modal dialog but has no `role="dialog"`, no `aria-modal="true"`, and no `aria-labelledby`. Additionally, there is no keyboard listener to dismiss it with Escape — the only close path is clicking the "Close" button with a pointer. The `LevelSelectorButton` that opens it also has no `aria-expanded` attribute.

**Impact:** Keyboard-only and screen-reader users cannot tell they have entered a modal context. Focus is not trapped inside the panel, so Tab will cycle through the underlying game UI (which remains in the DOM behind the overlay). Screen readers will not restrict their virtual buffer to the panel. The overlay cannot be closed by keyboard alone without Tab-navigating to the "Close" button.

**Suggestion:** Add `role="dialog"`, `aria-modal="true"`, and `aria-labelledby` pointing at the "Choose a Level" heading. Add a `svelte:window on:keydown` listener that calls `close()` on `Escape`. Add `tabindex="-1"` to the panel `<div>` and call `panelEl.focus()` on open. Add `aria-expanded={$levelSelectorOpen}` and `aria-controls` to `LevelSelectorButton`.

______________________________________________________________________

### [SEVERITY: High] LevelSelectorOverlay can phantom-reappear after game-state modal is dismissed

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/components/LevelSelectorOverlay.svelte:24` and `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/components/Overlay.svelte:16`

**Issue:** The `LevelSelectorOverlay` (z-index: 120) and the game-state `SharedOverlay` (z-index: 900) are visually independent but share no state. If a user opens the level selector and then dies (status becomes `GameOver`), the `SharedOverlay` rises over the level selector (900 > 120) showing the Game Over modal. When the player dismisses the modal (restarts), `levelSelectorOpen` is still `true` because nothing ever clears it on a game-state transition. The level selector panel reappears without any user gesture that opened it.

**Impact:** After restarting from Game Over (or dismissing any error modal), the level selector unexpectedly reappears, blocking the game grid and confusing the player.

**Suggestion:** In `stores.ts`, the `connectGameEngineToStores` handler (or a reactive statement in `App.svelte`) should call `levelSelectorOpen.set(false)` whenever a `frameChanged` event brings a new game into the `Playing` state following a level load or restart. Alternatively, `LevelSelectorOverlay` could react to `$gameState.status !== 'Playing'` and close itself.

______________________________________________________________________

### [SEVERITY: Medium] SpriteLoader has no error handling — silent blank grid on fetch failure

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/components/SpriteLoader.svelte:7-10`

**Issue:** The `onMount` function fetches the SVG sprite sheet and assigns `response.text()` directly to `spriteContent`. There is no `try/catch`, no check of `response.ok`, and no fallback. If the fetch fails (network error, bundler misconfiguration, CDN issue) the component silently leaves `spriteContent` as `""`, the `{@html}` block is skipped, and all `<Cell>` SVG `<use>` elements reference symbol IDs that are absent from the document. Every cell renders as an invisible empty element.

**Impact:** The entire game grid is visually blank with no error message. The player sees the header controls, scores, and the grey game-field container, but all cells are invisible. There is no error state in the UI.

**Suggestion:** Wrap the fetch in try/catch, check `response.ok`, and surface a recoverable error (either via a dedicated store or an inline error state) so the user sees a meaningful message rather than a broken-looking blank board.

______________________________________________________________________

### [SEVERITY: Medium] GameCompleteModal offers no clickable restart — mouse users are trapped

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/components/GameCompleteModal.svelte:5-9`

**Issue:** The modal for the `AllComplete` game state tells the player "Refresh the page to play again" and provides no button. The `KeyboardHandler` does support `r`, `q`, and `Escape` to call `gameEngine.loadLevel(1)` from the `AllComplete` state, but this capability is undisclosed in the modal and unavailable to mouse-only users.

**Impact:** Mouse-only players cannot restart without a full browser refresh, losing their conceptual context. The instruction to "refresh the page" is technically incorrect for keyboard users and misleading for all users since the engine supports in-app restart.

**Suggestion:** Add a "Play Again" button to `GameCompleteModal` that calls `gameEngine.loadLevel(1)`. This requires passing `gameEngine` as a prop (consistent with `GameOverModal`). Update the body text to match.

______________________________________________________________________

### [SEVERITY: Medium] Unkeyed `{#each}` in GameGrid causes full-grid DOM diffing every frame

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/components/GameGrid.svelte:17`

**Issue:** The cells are rendered with `{#each cells as cell}` without a key expression. On every `frame` store update (each player move), Svelte must diff the entire flat array of cells against the DOM and update every changed node. For typical snake grids (e.g., 15×15 = 225 cells), each frame re-evaluates all 225 `Cell` components. Additionally, `cells = grid.flat()` creates a new array every time `$frame` changes because the reactive statements (`$: grid`, `$: cells`) re-run unconditionally on every frame.

**Impact:** Per-frame DOM work scales with grid area rather than with the number of actually changed cells (typically 2–4 per move). On lower-end devices or very large levels, this can cause frame-rate degradation during fast gameplay.

**Suggestion:** Add a stable key to the `{#each}` block using cell coordinates: `{#each cells as cell, i (i)}`. Since the grid dimensions are fixed per level, index is a stable identity. For a deeper fix, restructure to iterate rows and columns separately so Svelte can target individual cells by a `(row, col)` key.

______________________________________________________________________

### [SEVERITY: Medium] Modal backdrop double-stacking produces unintended dark overlay

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-ui/components/Overlay.svelte:13-21` and `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-ui/components/Modal.svelte:40-49`

**Issue:** The application renders game-state modals by composing `SharedOverlay` (the `Overlay` component from `gsnake-web-ui`, z-index: 900, background: `rgba(0,0,0,0.65)`) wrapping a `Modal` component (z-index: 1000, background: `rgba(0,0,0,0.65)`). Both components use their own full-screen fixed backdrop. When any modal is shown (`GameOver`, `EngineError`, etc.) two dark translucent layers stack on top of each other, making the background roughly equivalent to `rgba(0,0,0,0.87)` — substantially darker than designed.

**Impact:** The game interface behind the modal is nearly black rather than the intended dim grey, reducing readability and looking unpolished.

**Suggestion:** Remove the outer `SharedOverlay` wrapper from `Overlay.svelte` (the app-level component) since `Modal.svelte` provides its own backdrop. Alternatively, render modals directly without wrapping them in `SharedOverlay`, or make `SharedOverlay` transparent when it contains a `Modal`.

______________________________________________________________________

### [SEVERITY: Medium] WasmGameEngine event listeners cannot be removed — potential leak on remount

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/engine/WasmGameEngine.ts:207-209` and `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/stores/stores.ts:49-67`

**Issue:** `WasmGameEngine.addEventListener` pushes to a private `listeners` array with no corresponding `removeEventListener`. `connectGameEngineToStores` in `stores.ts` calls this once and returns `void`, so there is no unsubscribe handle. `App.svelte` does not attempt to remove the store listener in `onDestroy`. In the current architecture this is benign because `App` only mounts once per page load, but it means any future Hot Module Replacement cycle, test harness mount/unmount, or framework-level remount will accumulate duplicate store listeners on the same engine instance.

**Impact:** In testing environments (confirmed present: `LevelUiFlows.test.ts` creates and destroys components), if the engine were shared across test cases, store updates would fire for every previously registered listener. In production, the risk is low but the design violates the principle of symmetric resource management.

**Suggestion:** Add a `removeEventListener(listener: GameEventListener): void` method to `WasmGameEngine` that splices from the `listeners` array. Have `connectGameEngineToStores` return the unsubscribe function. Call it in `App.svelte`'s `onDestroy`.

______________________________________________________________________

### [SEVERITY: Low] RestartButton has redundant `aria-disabled` alongside native `disabled`

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/components/RestartButton.svelte:19-20`

**Issue:** The button is rendered with both `disabled={disabled}` and `aria-disabled={disabled ? 'true' : 'false'}`. When the native `disabled` attribute is present on a button, assistive technology already reports it as disabled; adding `aria-disabled` is redundant and can confuse some AT combinations. Furthermore, `disabled` prop is always `false` in practice since `Header.svelte` never passes a `disabled` prop to `RestartButton` (line 14 of `Header.svelte`), making the dead-code guard in `handleRestart` (`if (disabled) return;`) unreachable.

**Impact:** Minor: no functional regression, but the defensive disabled guard and aria attribute are dead code paths that add maintenance overhead.

**Suggestion:** Remove `aria-disabled` (native `disabled` suffices). Either remove the `disabled` prop entirely if it is intentionally unused, or document the intended use case for when it would be passed.

______________________________________________________________________

### [SEVERITY: Low] Modal `closeOnBackdrop`/`onClose` are dead props in all current consumers

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-ui/components/Modal.svelte:3-14`

**Issue:** `Modal.svelte` exposes `onClose: (() => void) | undefined = undefined` and `closeOnBackdrop = true`. The `handleBackdropClick` handler conditionally calls `onClose?.()`. However, none of the four modal consumers (`EngineErrorModal`, `GameOverModal`, `GameCompleteModal`, `LevelLoadErrorModal`) pass `onClose`. This means backdrop clicks silently do nothing across the entire application.

**Impact:** Users who click outside a modal expecting it to close are not served. The feature is implemented but never wired up. There is also no discoverable documentation that backdrop-dismiss is intentionally disabled for all game modals.

**Suggestion:** Either wire `onClose` in each consumer where backdrop dismiss is appropriate (e.g., `LevelLoadErrorModal` with `dismiss`), or change the default to `closeOnBackdrop = false` to make the intent explicit and avoid misleading future maintainers.

______________________________________________________________________

### [SEVERITY: Low] SpriteLoader uses `{@html}` on fetched content

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/components/SpriteLoader.svelte:18`

**Issue:** `{@html spriteContent}` injects the fetched SVG directly into the DOM. The URL is derived from `spritesUrl`, which is a Vite asset import (`"./assets/sprites.svg?url"`) resolving to a developer-controlled static file bundled at build time. In the current setup the content is not user-controlled and the SVG has no embedded `<script>` tags. However, if a supply-chain compromise, a misconfigured CDN, or a future refactor changed `spritesUrl` to an external origin, the absence of sanitization would become exploitable XSS.

**Impact:** Currently negligible because `spritesUrl` is a same-origin bundled asset and the SVG has been verified to contain only static shape elements. The risk is architectural — the code makes no defense against an unexpected content change.

**Suggestion:** Add a comment documenting the trust assumption (same-origin bundled asset only). Long-term, prefer inlining the SVG at build time (Vite's `?raw` import + `{@html}` at import time rather than fetch-at-runtime) to eliminate the network dependency and the runtime `@html` injection. If a fetch is required, at minimum check `response.ok` and guard that the response `Content-Type` is `image/svg+xml`.

______________________________________________________________________

### [SEVERITY: Low] Hardcoded `localhost:3001` URL appears in production bundle

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/components/App.svelte:119`

**Issue:** The test-server URL `http://localhost:3001/api/test-level` is hard-coded in the main application component. It is only reached when the query parameter `?test=true` is present, so it does not execute in normal play, but it is included in every production build.

**Impact:** The URL leaks internal development infrastructure details to anyone who reads the source bundle. It also creates a misleading CORS/network error if a user discovers the parameter and tries it in a non-development environment.

**Suggestion:** Move the test server base URL to a build-time environment variable (e.g., `import.meta.env.VITE_TEST_SERVER_URL`) defaulting to `http://localhost:3001`. This removes the literal from the bundle and gives CI/CD control over the value without code changes.

______________________________________________________________________

### [SEVERITY: Low] Cell.svelte has a misaligned `export let` declaration

**File:** `/home/nntin/git/gSnake/gsnake-web/packages/gsnake-web-app/components/Cell.svelte:4`

**Issue:** The `export let type: CellType;` declaration is at column 0 (no indentation) while the rest of the `<script>` block is indented by two spaces. This appears to be an accidental outdent rather than intentional style.

**Impact:** None functional. Minor readability and linting inconsistency.

**Suggestion:** Indent the `export let type: CellType;` line to match the rest of the script block.

______________________________________________________________________

## Positive Observations

- **Store-driven architecture is clean.** All game state flows through Svelte writable stores and is updated by a single `connectGameEngineToStores` listener. UI components subscribe to the relevant slice (e.g., `GameGrid` only reads `frame`, `ScoreDisplay` reads `gameState` and `snakeLength`) which keeps reactive dependency graphs appropriately narrow.

- **Completion tracking is correct and idempotent.** The reactive block in `App.svelte` (lines 81–89) uses `lastCompletedId` to deduplicate repeated `LevelComplete` frames for the same level, preventing double-writes to `localStorage`. The event ordering guarantee (levelChanged → frameChanged with Playing status) ensures `lastCompletedId` resets cleanly on every level transition.

- **Defensive null handling in GameGrid.** `$frame?.grid ?? []` and `grid[0]?.length ?? 0` prevent crashes when the frame store is null before the engine emits its first frame, and the `isInitializing` guard in `App.svelte` ensures `GameGrid` is not mounted until the engine has produced a valid frame.

- **Focus management in GameOverModal.** The `onMount` + `bind:this` pattern to focus the "Restart Level" button on modal open (lines 9–11) is correct and gives keyboard users an immediate action target when the game ends.

- **Input validation before engine initialization.** All custom level sources (URL param, test mode) are schema-validated with `isLevelDefinition`/`isLevelArray` guards before being passed to the engine, with user-facing error messages surfaced through `levelLoadError`.

- **KeyboardHandler lifecycle is properly paired.** `attach()` and `detach()` are called symmetrically in `App.svelte`'s `onMount`/`onDestroy`, and the store subscription inside `KeyboardHandler` is cleaned up in `detach()`, preventing a common subscription leak pattern.

- **SVG sprite lookup is type-safe at compile time.** `CellType` is a TypeScript union and all ten values (`Empty`, `SnakeHead`, `SnakeBody`, `Food`, `Obstacle`, `Exit`, `FloatingFood`, `FallingFood`, `Stone`, `Spike`) have corresponding `<symbol id="...">` entries in the sprites SVG, verified by code review.
