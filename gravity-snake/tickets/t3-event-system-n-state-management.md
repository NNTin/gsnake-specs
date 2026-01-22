## Objective

Implement the event-driven architecture connecting GameEngine to Svelte stores, enabling reactive UI updates while maintaining decoupling.

## Scope

**Included:**

- Svelte writable stores for game state, snake, and level
- `connectGameEngineToStores()` adapter function
- Event listener implementation
- Store initialization with default values

**Excluded:**

- GameEngine implementation (separate ticket)
- UI components (separate ticket)
- Keyboard handling (separate ticket)

## Spec References

- spec:epics/gravity-snake/tech-plan.md - Tech Plan: Store Definitions
- spec:epics/gravity-snake/tech-plan.md - Tech Plan: Event-Driven Architecture

## Key Deliverables

**Stores** (`/src/stores.ts`):

```typescript
export const gameState = writable<GameState>({
  status: GameStatus.Playing,
  currentLevel: 1,
  moves: 0,
  foodCollected: 0,
  totalFood: 0
});

export const snake = writable<Snake>({
  segments: [],
  direction: null
});

export const level = writable<Level | null>(null);
```

**Adapter Function:**

```typescript
export function connectGameEngineToStores(engine: GameEngine): void {
  engine.addEventListener((event) => {
    switch (event.type) {
      case 'stateChanged':
        gameState.set(event.state);
        break;
      case 'snakeChanged':
        snake.set(event.snake);
        break;
      case 'levelChanged':
        level.set(event.level);
        break;
    }
  });
}
```

## Acceptance Criteria

- ✅ Three stores created with correct initial values
- ✅ Stores are reactive (can be subscribed to)
- ✅ `connectGameEngineToStores()` properly wires events to stores
- ✅ Store updates trigger when GameEngine emits events
- ✅ Stores can be imported and used in Svelte components

## Dependencies

- `ticket:T1` - Requires types and project structure

