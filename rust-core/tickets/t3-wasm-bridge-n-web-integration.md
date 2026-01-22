## Objective

Expose the Rust game engine to the web frontend via WebAssembly and replace the existing TypeScript engine logic.

## Scope

**Included:**

- Set up `gsnake-wasm` crate with `wasm-bindgen`
- Implement the WASM bridge to call JS callbacks on state changes
- Update `App.svelte` to initialize the WASM module
- Refactor `stores.ts` to receive and process `Frame` objects from WASM
- Remove or deprecate the original `engine/GameEngine.ts`

**Excluded:**

- CLI implementation
- Native Rust filesystem access (levels are injected via JS)

## Spec References

- spec:epics/rust-core/tech-plan.md - Tech Plan: Architectural Approach (WASM)
- spec:epics/rust-core/core-flows.md - Flow 1: WASM-Powered Gameplay

## Key Deliverables

1.  **WASM Bridge:** `gsnake-wasm/src/lib.rs` with JS bindings.
2.  **Frontend Integration:** Updated `App.svelte` and `stores.ts` using the WASM engine.

## Acceptance Criteria

- ✅ Web game is fully playable using the WASM engine
- ✅ Performance is equivalent or better than the TS engine
- ✅ UI Modals (Game Over/Complete) trigger correctly based on WASM events

## Dependencies

- T2: Core Game Logic Implementation
