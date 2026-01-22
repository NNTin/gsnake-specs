## Objective

Implement all Svelte components for the game UI including the grid, header, modals, and layout structure.

## Scope

**Included:**

- All Svelte components per Tech Plan hierarchy
- Component logic for rendering and user interactions
- Store subscriptions for reactive updates
- Component-scoped styles (basic structure, no polish)
- Grid rendering with CSS Grid
- Modal overlays for game over and completion

**Excluded:**

- Visual polish and final styling (separate ticket)
- Game logic (in GameEngine)
- Keyboard handling (separate ticket)

## Spec References

- spec:epics/gravity-snake/tech-plan.md - Tech Plan: Component Architecture
- spec:epics/gravity-snake/core-flows.md - Core Flows: UI Layout (wireframe)

## Key Deliverables

**Component Hierarchy** (`/src/components/`):

1. **App.svelte** - Root component, initializes engine (stub for now)
2. **GameContainer.svelte** - Main layout, subscribes to stores
3. **Header.svelte** - Score display and restart button container
4. **ScoreDisplay.svelte** - Shows level, length, moves
5. **RestartButton.svelte** - Restart button with click handler
6. **GameGrid.svelte** - 15×15 CSS Grid, computes cell types
7. **Cell.svelte** - Individual cell with type-based styling
8. **Overlay.svelte** - Modal overlay container
9. **GameOverModal.svelte** - Game over UI with two buttons
10. **GameCompleteModal.svelte** - All levels complete message

## Component Responsibilities

**GameGrid.svelte:**

- Subscribe to `snake` and `level` stores
- Compute cell type for each position (225 cells)
- Render Cell components with appropriate types
- Use CSS Grid for layout

**ScoreDisplay.svelte:**

- Subscribe to `gameState` and `snake` stores
- Display: Level number, snake length, moves count
- Format values for display

**Overlay.svelte:**

- Subscribe to `gameState` store
- Show/hide based on game status
- Render appropriate modal component

## Acceptance Criteria

- ✅ All 10 components created and properly nested
- ✅ Components subscribe to correct stores
- ✅ Grid renders 15×15 cells using CSS Grid
- ✅ Cell types display with distinct visual appearance
- ✅ Score display shows correct values from stores
- ✅ Modals show/hide based on game status
- ✅ Restart button calls GameEngine method (stub OK)
- ✅ Components are reactive to store changes

## Dependencies

- `ticket:T1` - Requires types and project structure
- `ticket:T3` - Requires stores to subscribe to

