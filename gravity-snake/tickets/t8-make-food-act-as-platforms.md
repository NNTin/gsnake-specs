## Objective

Add food as a platform type that stops the snake from falling, allowing the snake to stand on food items. This adds strategic depth - food becomes both a collectible and a platform.

## Current Behavior

file:engine/GameEngine.ts lines 213-227 (`canSnakeFall()`):

- Only checks obstacles and floor
- Snake falls through food

## Desired Behavior

Food should stop gravity just like obstacles do. When any snake segment would land on a food tile during gravity, the snake stops falling.

**Important:** Food is still collectible when the snake head is on it (during the movement phase, before gravity).

## Implementation Changes

**GameEngine.ts - `canSnakeFall()` method:**

Add food check to the collision detection:

```typescript
// Check if position has food
if (this.level.food.some(f => f.x === nextX && f.y === nextY)) {
  return false;  // Can't fall through food
}
```

**Logic Flow:**

1. Player moves snake head onto food → food is collected, snake grows
1. Gravity applies → remaining food items act as platforms
1. Snake can stand on uncollected food

## Strategic Implications

This creates interesting puzzle scenarios:

- Food can be used as stepping stones to reach higher platforms
- Collecting food removes platforms, potentially trapping the player
- Order of food collection becomes strategically important
- Players must plan: "Do I collect this food now or use it as a platform?"

## Acceptance Criteria

- ✅ Snake stops falling when any segment would land on food
- ✅ Food still collectible when head moves onto it (before gravity)
- ✅ Collected food no longer acts as platform (removed from level.food array)
- ✅ Multiple food items can act as platforms simultaneously
- ✅ Gravity physics work correctly with food as obstacles

## Related Specs

- spec:epics/gravity-snake/tech-plan.md - Tech Plan: applyGravity() and canSnakeFall()
- spec:epics/gravity-snake/core-flows.md - Core Flows: Flow 2 (gravity mechanics)

## Dependencies

None - can be implemented independently
