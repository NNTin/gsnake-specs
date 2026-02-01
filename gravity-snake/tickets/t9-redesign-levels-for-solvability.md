## Objective

Redesign levels 1-5 to ensure they are all solvable and provide good difficulty progression. Fix unreachable platform issues and validate that each level can be completed.

## Current Issues

- **Levels 1-2:** Have unreachable platforms (snake too small to reach certain areas)
- **Levels 3-5:** Not tested, may have similar issues
- **General:** Level design doesn't account for new mechanics (food as platforms, no food requirement for exit)

## Desired Outcome

5 well-designed levels that:

- Are all solvable (verified by playtesting)
- Demonstrate gravity mechanics effectively
- Provide increasing difficulty
- Account for new mechanics (food as platforms, optional food collection)
- Have sensible platform heights relative to snake length

## Implementation Approach

**For Each Level:**

1. Playtest to verify solvability
1. Identify unreachable areas or impossible scenarios
1. Redesign obstacle/food placement to fix issues
1. Consider new mechanics (food as platforms)
1. Verify difficulty progression

**Level Design Principles:**

- Snake starts at length 3
- Platforms should be reachable with current snake length OR after collecting food
- Food placement should create interesting strategic choices (collect now vs use as platform)
- Exit should be reachable (with or without food collection)
- Each level should teach or reinforce a gravity mechanic concept

## Suggested Level Themes

**Level 1: "First Steps"** - Tutorial

- Simple horizontal movement
- One food item
- Easy exit
- Teaches basic controls

**Level 2: "The Drop"** - Introduce Gravity

- Requires falling to reach areas
- Food as platform introduction
- Moderate difficulty

**Level 3: "Platform Puzzle"** - Strategic Food Use

- Multiple food items as platforms
- Choice: collect or use as stepping stone
- Medium difficulty

**Level 4: "Precision"** - Tight Spaces

- Narrow passages
- Risk of self-collision
- Requires careful planning

**Level 5: "The Gauntlet"** - Final Challenge

- Complex multi-step puzzle
- All mechanics combined
- Highest difficulty

## Acceptance Criteria

- ✅ All 5 levels are solvable (verified by playtesting)
- ✅ No unreachable platforms or impossible scenarios
- ✅ Difficulty progression is smooth (1 easiest → 5 hardest)
- ✅ Each level demonstrates gravity mechanics
- ✅ Food placement creates strategic choices
- ✅ Levels account for new mechanics (food as platforms, optional collection)

## Related Specs

- spec:epics/gravity-snake/epic-brief.md - Epic Brief: Level-based progression
- spec:epics/gravity-snake/tech-plan.md - Tech Plan: Level Data Structure

## Dependencies

- `ticket:T7` - Should be done after exit condition change
- `ticket:T8` - Should be done after food platform feature
