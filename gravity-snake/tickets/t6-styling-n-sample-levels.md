## Objective

Apply visual styling to match the wireframe design and create 3-5 sample levels that demonstrate the gravity mechanic and increasing difficulty.

## Scope

**Included:**

- CSS styling for all components
- Color scheme matching wireframe
- Responsive layout
- Cell type visual differentiation
- Modal styling
- 3-5 complete sample levels with increasing difficulty
- Level design that showcases gravity mechanic

**Excluded:**

- Advanced animations (out of MVP scope)
- Mobile/touch optimization (out of MVP scope)
- Sound effects (out of MVP scope)

## Spec References

- spec:epics/gravity-snake/core-flows.md - Core Flows: UI Layout (wireframe)
- spec:epics/gravity-snake/tech-plan.md - Tech Plan: Styling Approach
- spec:epics/gravity-snake/epic-brief.md - Epic Brief: Simple visual style (colored blocks)

## Key Deliverables

**Styling:**

- Global styles (typography, colors, resets)
- Component-scoped styles in each .svelte file
- CSS Grid layout for game field
- Cell type classes: `.snake-head`, `.snake-body`, `.food`, `.obstacle`, `.exit`
- Modal overlay styling
- Button styling
- Responsive container (max-width: 600px)

**Color Scheme** (from wireframe):

- Snake head: #2196F3 (blue)
- Snake body: #64B5F6 (light blue)
- Food: #FF5722 (orange-red)
- Obstacle: #424242 (dark gray)
- Exit: #4CAF50 (green)
- Background: #f5f5f5
- Grid cells: white

**Sample Levels** (`/src/data/levels.json`):

**Level 1: "First Steps"**

- Simple introduction
- Minimal obstacles
- One food item
- Clear path to exit

**Level 2: "Gravity Basics"**

- Introduces gravity mechanic
- Platform obstacles
- Food requires falling to reach

**Level 3: "Platform Puzzle"**

- Multiple platforms at different heights
- Strategic food placement
- Requires planning moves

**Level 4: "Narrow Passages"**

- Tight spaces
- Multiple food items
- Risk of self-collision

**Level 5: "The Challenge"**

- Complex obstacle layout
- Multiple food items
- Requires careful planning

## Acceptance Criteria

- ✅ Visual appearance matches wireframe design
- ✅ All cell types are visually distinct
- ✅ Layout is clean and centered
- ✅ Modals are properly styled
- ✅ 5 levels created with increasing difficulty
- ✅ Each level is completable
- ✅ Levels demonstrate gravity mechanic effectively
- ✅ Game looks polished and professional (within MVP scope)

## Dependencies

- `ticket:T5` - Requires fully integrated game
