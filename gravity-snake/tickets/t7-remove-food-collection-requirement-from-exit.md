## Objective

Change the exit condition to allow level completion without collecting all food. Food becomes an optional challenge for score/length rather than a requirement.

## Current Behavior

file:engine/GameEngine.ts line 264:

```typescript
return this.gameState.foodCollected === this.gameState.totalFood;
```

Exit only works when ALL food is collected.

## Desired Behavior

Exit should work immediately when the snake head reaches the exit tile, regardless of food collection status. Food becomes optional - players can choose to collect it for higher score/length or skip it to complete levels faster.

## Implementation Changes

**GameEngine.ts - `checkExit()` method:**

- Remove the food collection check
- Return true if head is on exit tile
- Simplify to: `return head.x === this.level.exit.x && head.y === this.level.exit.y`

**GameState interface:**

- `foodCollected` and `totalFood` still track food for scoring
- But they don't gate level completion

## Impact on Game Design

This changes the game from a "collect-all" puzzle to a "reach-the-exit" puzzle where:

- Food is optional bonus (increases length, potentially helps reach platforms)
- Players can optimize for speed (skip food) or completeness (collect all)
- Levels become more flexible in design

## Acceptance Criteria

- ✅ Exit works when snake head reaches exit tile (no food check)
- ✅ Food collection still tracked in gameState
- ✅ Food still grows snake when collected
- ✅ Level completes immediately upon reaching exit
- ✅ All 5 levels can be completed by reaching exit

## Related Specs

- spec:epics/gravity-snake/core-flows.md - Core Flows: Flow 2 (exit condition needs update)
- spec:epics/gravity-snake/tech-plan.md - Tech Plan: checkExit() method

## Dependencies

None - can be implemented independently