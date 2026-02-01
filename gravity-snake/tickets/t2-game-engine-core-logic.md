## Objective

Implement the GameEngine class with all core game logic including movement, gravity physics, collision detection, food collection, and level management.

## Scope

**Included:**

- `GameEngine` class with complete game logic
- Movement algorithm (directional input, snake segment updates)
- Gravity physics (entire snake falls together, checks all segments)
- Collision detection (obstacles, boundaries, self-collision)
- Food collection tracking
- Level loading and progression
- Exit condition checking
- Input processing lock (`isProcessing` flag)
- Grid computation and caching
- Event emission (stubs for now, full integration in T3)

**Excluded:**

- UI components
- Keyboard input handling (separate ticket)
- Svelte store integration (separate ticket)
- Actual level data (sample levels in T6)

## Spec References

- spec:epics/gravity-snake/tech-plan.md - Tech Plan: Game Engine Module
- spec:epics/gravity-snake/tech-plan.md - Tech Plan: Key Methods (processMove, applyGravity)
- spec:epics/gravity-snake/core-flows.md - Core Flows: Flow 2 (Core Gameplay Loop)

## Key Deliverables

**GameEngine Class** (`/src/engine/GameEngine.ts`):

Public API:

- `init(levels: Level[]): void`
- `processMove(direction: Direction): void`
- `restartLevel(): void`
- `loadLevel(levelNumber: number): void`
- `addEventListener(listener: GameEventListener): void`
- `removeEventListener(listener: GameEventListener): void`
- `canProcessInput(): boolean`

Private Methods:

- `emit(event: GameEvent): void`
- `moveSnake(direction: Direction): void`
- `applyGravity(): void` - Iterative, checks all segments
- `checkCollision(pos: Position): boolean`
- `checkFoodAtPosition(pos: Position): boolean`
- `canSnakeFall(): boolean`
- `checkExit(pos: Position): boolean`
- `isValidPosition(pos: Position): boolean`
- `computeGrid(): Grid`
- `getGrid(): Grid`

## Critical Logic

**Movement Order:**

1. Calculate new head position
1. Check for food BEFORE modifying snake
1. If food: unshift head (grow), don't pop tail
1. If no food: unshift head, pop tail
1. Apply gravity to entire snake

**Gravity Physics:**

- Check ALL segments for collision before each downward move
- If ANY segment would collide, stop falling
- Move all segments down simultaneously

## Acceptance Criteria

- ✅ GameEngine can be instantiated and initialized with level data
- ✅ `processMove()` correctly handles all 4 directions
- ✅ Gravity applies to entire snake, stops when any segment hits obstacle
- ✅ Collision detection works for obstacles, boundaries, and self
- ✅ Food collection increments counter and grows snake
- ✅ Exit condition triggers when all food collected
- ✅ Level progression loads next level correctly
- ✅ Input lock prevents concurrent move processing
- ✅ Grid caching works with dirty flag
- ✅ Events are emitted for state changes

## Dependencies

- `ticket:T1` - Requires types and project structure
