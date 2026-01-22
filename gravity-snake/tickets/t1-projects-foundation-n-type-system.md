## Objective

Set up the Svelte + TypeScript + Vite project foundation and define all core type definitions that will be used throughout the application.

## Scope

**Included:**

- Initialize Vite + Svelte + TypeScript project
- Configure TypeScript with strict mode
- Define all core TypeScript interfaces and enums
- Set up project structure (folders for engine, components, stores, data)
- Configure build and dev scripts
- Add basic HTML template

**Excluded:**

- Any game logic implementation
- UI components
- Styling

## Spec References

- spec:epics/gravity-snake/tech-plan.md - Tech Plan: Data Model section
- spec:epics/gravity-snake/tech-plan.md - Tech Plan: Architectural Approach (Vite + Svelte)

## Key Deliverables

1. **Project Structure:**
  ```
   /src
     /engine      (GameEngine, KeyboardHandler)
     /components  (Svelte components)
     /stores      (Svelte stores)
     /data        (levels.json)
     /types       (TypeScript definitions)
  ```
2. **Type Definitions** (in `/src/types/`):
  - `Position` interface
  - `Direction` enum
  - `CellType` enum
  - `GameStatus` enum
  - `Snake` interface
  - `Level` interface
  - `GameState` interface
  - `Grid` type and `GridCache` interface
  - `GameEvent` type and `GameEventListener` type
3. **Configuration:**
  - `tsconfig.json` with strict mode
  - `vite.config.ts` for Svelte
  - `package.json` with scripts

## Acceptance Criteria

- ✅ Project builds successfully with `npm run build`
- ✅ Dev server runs with `npm run dev`
- ✅ All TypeScript types compile without errors
- ✅ Project structure matches Tech Plan organization
- ✅ TypeScript strict mode enabled and passing

## Dependencies

None - this is the foundation ticket