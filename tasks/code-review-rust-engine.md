# Code Review: Rust Game Engine

## Summary

The engine core is well-structured, makes consistent use of `Result` for error propagation, avoids `unwrap()`/`panic!()` in production paths, and is backed by a solid test suite that includes unit tests, contract tests, and property-based tests. The most critical issues are a latent integer-overflow path in the grid-size validation, a logical gap where stones can be pushed through food and the exit without detection, and several missing test cases for the newer `CellType` variants. Secondary concerns include unclear semantics when `total_food` is zero versus genuinely absent, the silent discard of the `on_frame` callback error on registration, and the use of the unmaintained `wee_alloc` crate.

______________________________________________________________________

## Findings

### [SEVERITY: High] Integer overflow when computing `MAX_GRID_CELLS` boundary in 32-bit test helper

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs:900-906`
**Issue:** The test `test_engine_creation_accepts_maximum_valid_grid_size` casts `MAX_GRID_CELLS` (a `usize` equal to `2_000_000`) to `i32` with `as`:

```rust
level.grid_size = GridSize::new(MAX_GRID_CELLS as i32, 1);
```

`MAX_GRID_CELLS` is `2_000_000`, which fits in `i32` (`i32::MAX` is about `2.1 billion`), so this specific constant is safe. However, the adjacent test at line 912 does:

```rust
level.grid_size = GridSize::new((MAX_GRID_CELLS as i32) + 1, 1);
```

If `MAX_GRID_CELLS` were ever raised above `i32::MAX` (e.g., to match a 64-bit `usize`), both of these casts would silently produce negative values, the `validate_grid_size` check at line 381 (`width <= 0`) would then accept the truncated negative value, and the test assertions would become meaningless. The Cargo.toml workspace already suppresses the `cast-possible-truncation` and `cast-possible-wrap` clippy lints (lines 38-40), so no automated warning will fire.
**Impact:** If `MAX_GRID_CELLS` is increased beyond `i32::MAX`, the boundary tests silently stop testing what they claim, and the guard against oversized grids breaks. This is a latent defect today but a correctness bug if the constant is tuned upward.
**Suggestion:** Use `i32::try_from(MAX_GRID_CELLS).expect("MAX_GRID_CELLS must fit in i32")` in the test helpers, and add a `const _: () = assert!(MAX_GRID_CELLS <= i32::MAX as usize);` compile-time assertion in `engine.rs`.

______________________________________________________________________

### [SEVERITY: High] Stone push ignores food and exit cells as blocking terrain

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/stone_mechanics.rs:110-145`
**Issue:** `is_space_available` checks only obstacles, other stones, and snake segments. It does **not** check regular food, floating food, falling food, or the exit cell:

```rust
fn is_space_available(pos: Position, level_state: &LevelState) -> bool {
    // ... bounds check ...
    // Check for obstacles
    // Check for other stones
    // Check for snake segments
    true  // food and exit are not checked
}
```

A stone can therefore be pushed horizontally into a cell that contains regular food, floating food, falling food, or the exit tile. After the push the stone occupies the same cell as the food or exit, which leads to:

- The food item still being present in `level_state.food` at that position, causing it to be eaten the next time the snake moves there — but the stone is also there, making the level inconsistent.
- A stone can be pushed onto the exit tile, rendering the exit permanently inaccessible without any indication to the player.
  **Impact:** Wrong game behavior reachable from valid player input. A stone pushed onto a food cell creates a corrupted state where both objects visually overlap and the food is still collectable while the stone is there. A stone pushed onto the exit can soft-lock the level.
  **Suggestion:** Add food, floating food, falling food, and exit (when `exit_is_solid`) to the blocking checks inside `is_space_available`. Mirror the checks that `can_object_fall` already applies in `gravity.rs`.

______________________________________________________________________

### [SEVERITY: High] Gravity does not check whether the snake passes through food during a fall

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/gravity.rs:74-112`
**Issue:** `can_snake_fall` stops the snake above food (treating food as a platform), but it does so on a per-step basis inside a `while can_snake_fall` loop. This is correct for settlement, but the snake can still land *on* a food cell (the check stops it one cell above the food), and the food is never consumed as part of the gravity fall. The `check_and_eat_food` call in `engine.rs` line 111 only checks the new head position after an explicit horizontal move — it does not run after gravity-induced falls. If gravity drops the snake head directly onto a food cell (e.g., the food shifts downward on the same column as the snake), the food occupies the same cell as the snake without being eaten.

In practice `can_snake_fall` halts the snake one cell *above* food, so the snake never actually lands on the food. However, the comment at line 91-97 says food "acts as platform for snake", which relies on this pre-stop invariant holding in all cases. The single-segment snake case is fine, but with a multi-segment snake that has segments at varying columns, only the failing column stops the snake — food in a non-head column still acts as a floor. That food item is never eaten even if the snake slides along it.
**Impact:** Medium gameplay impact — food can become permanently unreachable if a multi-segment snake's body lands beside it, though in practice the food-as-platform stops the snake before overlap rather than during.
**Suggestion:** After each step of gravity-induced movement, run `check_and_eat_food` on the new head position. Alternatively, document explicitly that food is only consumed by deliberate moves, not by gravity, and reflect this in the level design guidelines.

______________________________________________________________________

### [SEVERITY: Medium] `on_frame` callback error is silently discarded on registration

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/src/lib.rs:44-47`
**Issue:** The `on_frame` method emits an initial frame to the newly registered callback via `emit_frame()`, but its `Result` is discarded with `let _ =`:

```rust
pub fn on_frame(&mut self, callback: Function) {
    self.on_frame_callback = Some(callback);
    let _ = self.emit_frame();  // error silently dropped
}
```

`on_frame` itself has no return type, so there is no natural way to surface the error. However, if `emit_frame` fails (e.g., JSON serialization of the initial frame fails, or the JS callback throws), the caller receives no indication. The JS side will observe that `onFrame` returned `undefined` and may believe the callback was registered successfully and the initial frame was delivered, when in fact it was not.
**Impact:** Serialization failures or JS callback errors during initial frame delivery are invisible to the JS caller, potentially causing the UI to display a stale or empty state with no error to act on.
**Suggestion:** Change `on_frame` to return `Result<(), JsValue>` and propagate the error. If changing the signature is undesirable (e.g., due to API stability concerns), log the error to the browser console via `web_sys::console::error_1`.

______________________________________________________________________

### [SEVERITY: Medium] `total_food = 0` is ambiguous: zero food required vs. field absent

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs:25-29`
**Issue:** The engine uses `total_food == 0` as a sentinel to trigger auto-counting:

```rust
let total_food = if level.total_food > 0 {
    level.total_food
} else {
    (level.food.len() + ...) as u32
};
```

A level that genuinely has zero food items (e.g., a puzzle where the snake only needs to reach the exit) will auto-calculate `total_food = 0`, which happens to be correct. But a level definition that explicitly sets `total_food: 0` intending to override the count to zero cannot be distinguished from "field not set". If a designer writes `"totalFood": 0` in the JSON expecting no food to be required, the engine will recount from the food vectors. If those vectors are non-empty for cosmetic reasons, the win condition is broken.

Additionally, in `models.rs` line 183, `LevelDefinition::new()` initialises `total_food: 0`, yet the comment in `engine.rs` says "if `total_food` is explicitly set (non-zero), use it" — the distinction between "explicitly set to zero" and "default zero" is unenforceable from the type system alone.
**Impact:** Level definitions that legitimately need `total_food = 0` cannot express this unambiguously; the behaviour depends on the food vector contents rather than the explicit field.
**Suggestion:** Change `total_food` to `Option<u32>` in `LevelDefinition`. `None` means "auto-count"; `Some(n)` means "exactly n items required, including zero". Update the JSON deserialisation accordingly.

______________________________________________________________________

### [SEVERITY: Medium] `GameStatus::AllComplete` is defined but never set by the engine

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/models.rs:40`
**Issue:** `GameStatus` includes `AllComplete`, but `GameEngine` only ever sets `GameStatus::Playing`, `GameStatus::GameOver`, and `GameStatus::LevelComplete`. No code path in `engine.rs` transitions to `AllComplete`. The WASM binding, the CLI binding, and all tests deal only with `LevelComplete`.

If the engine is supposed to signal "all levels finished" after the last level completes, that logic is absent. If `AllComplete` is managed by the host application (JS layer), there is no documentation of this contract, and the variant's presence inside the core engine type is misleading.
**Impact:** Either the final-level win condition is silently missing, or the enum carries a dead variant that inflates the JS-side type surface with a value that will never arrive from the engine.
**Suggestion:** Either implement the `AllComplete` transition in the engine (requires the engine to know how many total levels exist) and add a test, or remove the variant from the core `GameStatus` enum and let the JS layer handle the "all done" state by tracking `LevelComplete` events itself. Document the decision explicitly.

______________________________________________________________________

### [SEVERITY: Medium] Stone push does not check the out-of-bounds path for all stones in a multi-stone row

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/stone_mechanics.rs:57-68`
**Issue:** `can_push_stones` only checks the **last** stone's target position:

```rust
let last_stone = stones[stones.len() - 1];
let target_pos = get_next_position(last_stone, direction);
is_space_available(target_pos, level_state) || is_spike_at(target_pos, &level_state.spikes)
```

The individual intermediate stones are not re-checked against `is_space_available`. This is correct for contiguous rows, because `find_stone_row` only adds a stone to the row if the very next position contains a stone. But after `execute_stone_push` moves stones (from last to first), each stone's *new* position is the old position of the stone ahead of it — which was just vacated. The logic works if the row is truly contiguous.

However, `find_stone_row` uses `get_next_position` which includes North and South moves (it is a generic function), but `try_push_stone` calls it only after already rejecting vertical pushes. If `find_stone_row` were ever called from another site without the vertical guard, North/South offsets could produce an incorrect row. This is not a current bug but a fragile coupling.
**Impact:** Low immediate risk. The contiguous-row invariant holds given the current call site. The concern is future maintainability if `find_stone_row` is reused.
**Suggestion:** Restrict `find_stone_row` to only accept horizontal directions (add an assertion or change the function signature to take a horizontal-only enum), making the invariant explicit rather than implicit.

______________________________________________________________________

### [SEVERITY: Medium] `wee_alloc` is unmaintained and has known memory issues

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/Cargo.toml:21`
**Issue:** `wee_alloc = "0.4"` is set as the global WASM allocator. The `wee_alloc` crate has been formally unmaintained since 2022, has known memory fragmentation and correctness issues under sustained allocation patterns, and is no longer recommended by the `wasm-bindgen` team. The Rust WASM working group now recommends using the default system allocator or `dlmalloc`.
**Impact:** In a game engine that allocates `Vec` frequently for snake segments, food lists, and stone lists, prolonged sessions may exhibit gradual heap corruption or memory growth in WASM environments.
**Suggestion:** Remove `wee_alloc` and its `#[global_allocator]` declaration, relying on the default allocator. If binary size is a concern, enable `opt-level = "z"` and LTO (already set in `[profile.release]`) or evaluate `lol_alloc` as a maintained alternative.

______________________________________________________________________

### [SEVERITY: Medium] `is_settled_falling_food` is recursive without a depth bound

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/gravity.rs:158-181`
**Issue:** `is_settled_falling_food` determines whether a falling food item at position `pos` has solid support by checking what is at `pos.y + 1`. If that cell is also a falling food item, the function calls itself recursively:

```rust
|| is_settled_falling_food(below, level_state)
```

For a column of N falling food items stacked vertically, the recursion depth is N. There is no depth limit. With a grid height up to `MAX_GRID_CELLS / 1 = 2,000,000` rows theoretically possible (though 2 M rows of food is unrealistic), a sufficiently tall column could overflow the stack.

In practice, game levels will have at most a handful of falling food items, making a stack overflow unreachable from real gameplay. However, the function is called inside `can_object_fall` which is called inside a loop in `apply_gravity_to_falling_food`, so the recursion runs repeatedly per gravity step.
**Impact:** Low for real game grids. A malformed or adversarial level with a very tall column of falling food could cause a stack overflow.
**Suggestion:** Replace the recursive check with an iterative scan downward, or add a depth counter with a reasonable cap (e.g., grid height).

______________________________________________________________________

### [SEVERITY: Medium] Missing round-trip and serialization tests for newer `CellType` variants

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/tests/contract_tests.rs:28-47, 234-250`
**Issue:** Both `test_celltype_serialization` (line 28) and `test_celltype_roundtrip` (line 234) enumerate only six `CellType` variants: `Empty`, `SnakeHead`, `SnakeBody`, `Food`, `Obstacle`, `Exit`. The three newer variants — `FloatingFood`, `FallingFood`, and `Stone`, and `Spike` — are completely absent from the serialization contract tests.

If the string representation of any of these variants changes (e.g., a rename or a `serde(rename_all)` change), no test will catch the regression before it reaches the JS frontend.
**Impact:** Breaking changes to the wire format for `FloatingFood`, `FallingFood`, `Stone`, or `Spike` would go undetected by the test suite.
**Suggestion:** Extend both test functions to cover all 10 `CellType` variants. The exhaustive list makes future additions visible as a test gap immediately when the enum changes.

______________________________________________________________________

### [SEVERITY: Low] `drop(direction)` is a no-op and adds confusion

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/bindings/wasm/src/lib.rs:147`
**Issue:**

```rust
let direction_str = direction.as_string().ok_or_else(|| { ... })?;
drop(direction);  // explicit drop of a JsValue
parse_direction_str(&direction_str).map_err(...)
```

`direction` (a `JsValue`) is on the stack and would be dropped automatically when the function returns. The explicit `drop(direction)` call has no semantic effect on the JS side (the reference count in the WASM JS glue is managed by `wasm-bindgen` and is decremented when the Rust binding function returns). The comment-free `drop` suggests the author intended to "release" the JS object early, but `JsValue` does not hold a live handle that benefits from early release within a synchronous function.
**Impact:** No runtime impact. Minor readability confusion.
**Suggestion:** Remove the `drop(direction)` call.

______________________________________________________________________

### [SEVERITY: Low] `LevelDefinition::new()` initialises `exit_is_solid: Some(true)` but `LevelState::from_definition` defaults missing value to `true`

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/models.rs:182, 219`
**Issue:** `LevelDefinition::new()` always sets `exit_is_solid: Some(true)`. `LevelState::from_definition` uses `unwrap_or(true)` to handle a `None` value. Since the constructor never produces `None`, the `unwrap_or` fallback is unreachable from any level created via `new()`. The field is meaningful only for levels deserialized from JSON where `exitIsSolid` is absent. This creates a subtle distinction between "programmatically created level" (always `Some(true)`) and "JSON-deserialized level" (may be `None`), making the API harder to reason about.
**Impact:** No current bug. Future code relying on `exit_is_solid` being `None` to detect "not specified" will be surprised that `new()` always emits `Some(true)`.
**Suggestion:** Either change `new()` to set `exit_is_solid: None` (relying on the `unwrap_or(true)` default), or document clearly that `None` and `Some(true)` are semantically equivalent for `exit_is_solid`.

______________________________________________________________________

### [SEVERITY: Low] Collision check order — stones not checked as a collision target after gravity moves snake onto them

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs:276-299`
**Issue:** `check_collision` tests for spikes, out-of-bounds, obstacles, and self-collision, but **not for stones**. After gravity runs and the snake's head lands on a stone cell, no collision is detected. Stones are treated as platforms (via `can_snake_fall`), so the snake stops one cell above them. However, `check_collision` is called before gravity at line 122, on the position after the horizontal move but before gravity. If a stone were already at the new head position (because it wasn't pushed successfully), the move would have been rejected at line 92-107 via the `is_stone` check. So in the current flow there is no path where the snake head is directly on a stone post-move.

The omission is not a current bug but is a hidden dependency between `check_collision` and the earlier stone-push guard. If the stone-push guard were relaxed or the call order changed, the snake could land on a stone without a collision being detected.
**Impact:** Low. Currently safe due to call-order dependency. The absence of a stone check in `check_collision` is a maintainability hazard.
**Suggestion:** Add a stone-presence check to `check_collision` (or add a comment explaining why it is intentionally absent), so the function is a self-contained, correct collision oracle.

______________________________________________________________________

### [SEVERITY: Low] `generate_frame` silently returns an empty grid for invalid runtime state

**File:** `/home/nntin/git/gSnake/gsnake-core/engine/core/src/engine.rs:155-169`
**Issue:** If `level_state.grid_size` is invalid at runtime (non-positive dimensions or cell count exceeds `MAX_GRID_CELLS`), `generate_frame` silently returns `Frame::new(Vec::new(), self.game_state.clone())`. The engine was already validated at construction (`validate_grid_size` in `new()`), but the runtime `grid_size` field is `pub` and can be mutated by tests or future code. The test `test_frame_generation_invalid_runtime_grid_is_safe` (line 933) explicitly covers this path and expects an empty grid, so the silent-return is an intentional contract.

However, this contract is not documented via a doc-comment on `generate_frame`, and the empty-grid return is indistinguishable from a valid 0×0 grid from the caller's perspective.
**Impact:** No current runtime impact. Callers (including the WASM binding at line 72) do not check whether the returned grid is empty before sending it to JS.
**Suggestion:** Add a doc-comment to `generate_frame` explaining the empty-grid invariant. Consider returning `Option<Frame>` or `Result<Frame, EngineError>` to make the failure explicit for callers.

______________________________________________________________________

### [SEVERITY: Low] Duplicate position-equality checks instead of using `PartialEq`

**File:** Multiple locations in `engine.rs`, `gravity.rs`, `stone_mechanics.rs`
**Issue:** Throughout the codebase, position equality is checked with explicit field comparisons:

```rust
// engine.rs:253, 294, 324, 332, 340
.any(|o| o.x == pos.x && o.y == pos.y)
```

`Position` derives `PartialEq`, so these could simply be:

```rust
.any(|o| o == &pos)
// or equivalently
.any(|o| *o == pos)
```

The explicit field comparisons are scattered across at least 12 call sites in three files.
**Impact:** No correctness impact. Minor code noise; if `Position` ever gains a third field (e.g., `z` for a future 3D mode), the explicit comparisons would silently omit the new field.
**Suggestion:** Use `PartialEq` comparisons uniformly. A `clippy` lint (`clippy::nonminimal_bool` or a custom lint) could enforce this.

______________________________________________________________________

## Positive Observations

- **No `unwrap()`/`panic!()` in production paths.** Every fallible operation in `engine.rs` and `lib.rs` returns `Result`, and the WASM bindings translate all errors to structured `ContractError` objects with typed `ContractErrorKind` variants. This is exemplary.

- **Grid size is validated at engine construction, not lazily.** `validate_grid_size` runs in `GameEngine::new`, uses `checked_mul` to prevent overflow, and converts to `EngineError` variants. The error messages are precise and include both dimensions.

- **Gravity is order-independent for stones (property test coverage).** `gravity_property_tests.rs` includes a proptest that verifies stone gravity converges to the same layout regardless of the order stones appear in the `stones` vector. This catches subtle ordering bugs that unit tests would miss.

- **Win condition is checked before gravity.** The engine explicitly resolves the exit check at line 125 before applying gravity (lines 132-141), and this is covered by `test_win_condition_checked_before_gravity`. The priority order (spike collision → win → gravity) is clearly intentional and tested.

- **Input locking is correctly scoped.** The `input_locked` flag is set before any state mutation and always cleared on every return path (success, blocked push, opposite direction). There is no path that leaves the engine permanently locked.

- **`LevelDefinition` uses `#[serde(default)]` for optional game mechanics.** Fields like `stones`, `spikes`, `floating_food`, `falling_food` default to empty vectors, making old level JSON files forward-compatible with new engine features without breaking deserialization.

- **`ContractError` is rich and JS-friendly.** The structured error type with `kind`, `message`, `context`, and `rejection_reason` gives the JS frontend enough information to display user-facing messages and for automated tests to assert on error kinds, not just opaque strings.

- **`console_error_panic_hook` is installed.** The WASM binding installs the panic hook at startup, so any unexpected panic produces a readable stack trace in the browser console rather than a silent abort.
