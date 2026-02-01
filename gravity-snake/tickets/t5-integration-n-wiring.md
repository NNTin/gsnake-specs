## Objective

Wire together all components: keyboard input handling, GameEngine initialization, store connections, and level data loading. Make the game fully playable.

## Scope

**Included:**

- `KeyboardHandler` class for input capture
- App.svelte initialization logic
- Level data loading (create minimal test level)
- Connect GameEngine to stores via adapter
- Wire restart button to GameEngine
- Wire modal buttons to GameEngine methods
- End-to-end integration testing

**Excluded:**

- Sample level design (separate ticket)
- Visual styling polish (separate ticket)

## Spec References

- spec:epics/gravity-snake/tech-plan.md - Tech Plan: Input Handling
- spec:epics/gravity-snake/tech-plan.md - Tech Plan: Integration Points
- spec:epics/gravity-snake/core-flows.md - Core Flows: All flows

## Key Deliverables

**KeyboardHandler** (`/src/engine/KeyboardHandler.ts`):

- Map arrow keys and WASD to Direction enum
- Check `canProcessInput()` before calling `processMove()`
- Prevent default browser behavior for game keys
- Attach/detach event listeners

**App.svelte Integration:**

```typescript
onMount(() => {
  gameEngine = new GameEngine();
  gameEngine.init(levels);
  connectGameEngineToStores(gameEngine);
  
  keyboardHandler = new KeyboardHandler(gameEngine);
  keyboardHandler.attach();
  
  return () => keyboardHandler.detach();
});
```

**Minimal Test Level** (`/src/data/levels.json`):

- One simple level for testing
- Small grid (can use 15×15)
- Few obstacles
- One food item
- Clear path to exit

**Component Wiring:**

- RestartButton calls `gameEngine.restartLevel()`
- GameOverModal buttons call appropriate methods
- All components properly connected to stores

## Acceptance Criteria

- ✅ Arrow keys and WASD control snake movement
- ✅ Game initializes and loads first level on page load
- ✅ Snake moves in response to keyboard input
- ✅ Gravity applies after each move
- ✅ Collision detection triggers game over
- ✅ Food collection works and grows snake
- ✅ Exit condition triggers level complete
- ✅ Restart button reloads current level
- ✅ Game over modal buttons work correctly
- ✅ All user flows from Core Flows are functional
- ✅ Game is fully playable end-to-end

## Dependencies

- `ticket:T2` - Requires GameEngine
- `ticket:T3` - Requires stores and adapter
- `ticket:T4` - Requires UI components
