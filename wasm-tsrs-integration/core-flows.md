# Core Flows: Stable Core Interface (wasm-bindgen + ts-rs)

## Flow 1: UI Developer Integrates Engine Interface

Description: UI developer consumes the strict interface to render and control gameplay with no manual type mapping or normalization.
Trigger / Entry: Developer integrates the engine into the Svelte client or updates a UI feature that consumes engine data.
Steps:

1. Import shared domain types from the ts-rs generated bundle.
1. Initialize the wasm module (`init()`), then fetch levels via `getLevels()`.
1. Construct `WasmGameEngine` with a `LevelDefinition` and register `onFrame` callback.
1. Render strictly from `Frame.grid` and `Frame.state` as provided; do not derive gameplay state.
1. Send user input via `processMove(direction)` using `Direction` enum values.
   Feedback & State:

- Success: UI renders grid and state without any runtime normalization or manual mapping.
- Error: Invalid input or wasm errors surface as typed error payloads with deterministic `kind`.
  Exit: UI reliably drives the engine and renders frames using shared types.

## Flow 2: Engine Developer Updates Core Models (Strict Contract)

Description: Engine developer updates domain models while preserving the strict contract baseline.
Trigger / Entry: Developer changes core gameplay data structures or wasm API.
Steps:

1. Update Rust domain models with serde + ts-rs derives and update wasm API signatures.
1. Regenerate TS types and confirm all enums serialize as strings matching ts-rs output.
1. Remove or update any UI code that normalizes or maps payloads.
1. Add/adjust integration tests to validate serialized payloads and error shapes.
   Feedback & State:

- Success: Types generate cleanly and UI compiles with no runtime normalization.
- Error: Type drift or enum serialization mismatch detected in tests or build.
  Exit: Core models evolve and the UI consumes strict payloads without adapter code.

## Flow 3: Engine Initialization Fails

Description: UI developer handles wasm initialization failure during startup.
Trigger / Entry: Engine initialization throws or fails to load.
Steps:

1. Attempt engine initialization at app startup.
1. If initialization fails, retry once.
1. If it fails again, surface a fatal error state and log the typed error payload.
   Feedback & State:

- Error: Fatal initialization failure state shown instead of gameplay UI.
  Exit: User sees a clear failure state and the developer has actionable diagnostics.

## Flow 4: Type Drift Detected

Description: UI developer encounters payloads that do not match the strict contract.
Trigger / Entry: UI consumes a payload or type that diverges from generated ts-rs types.
Steps:

1. Build the UI against generated types from the current Rust models.
1. Fail the build or integration tests on any enum/string mismatch or missing field.
1. Surface a clear error message that identifies the mismatched type.
   Feedback & State:

- Error: Type drift is detected at build/test time, not at runtime.
  Exit: Developer updates Rust models or UI usage to match the strict contract.
