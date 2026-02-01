## Summary

gSnake currently supports a limited set of game objects (obstacles, food, snake, exit), resulting in repetitive level designs that lack variety. This Epic extends the game object system with four new object types—floating food, falling food, stones, and spikes—plus an enhanced exit mechanic. These additions enable new puzzle mechanics including Sokoban-style block pushing, physics-based challenges, and dynamic environments with reactive falling objects. The implementation maintains the clean separation between `gsnake-core` (game logic) and `gsnake-web` (rendering) through minimal contract changes. Success is measured by correctly functioning mechanics, improved system extensibility for future objects, and enhanced player engagement through more varied puzzle designs.

## Context & Problem

**Who's Affected:**

- **Players** experience repetitive gameplay due to limited puzzle variety with only basic obstacles and food collection
- **Level designers** lack creative building blocks to express diverse puzzle concepts (block-pushing, timing challenges, physics-based puzzles)
- **Future contributors** face a rigid object system that makes adding new game elements difficult

**Current State:**\
The game uses a simple object model defined in file:gsnake-core/engine/core/src/models.rs with `CellType` enum (Empty, SnakeHead, SnakeBody, Food, Obstacle, Exit) and position-based collections in `LevelDefinition`. All objects have static behavior except the snake, which responds to gravity. Food acts as both collectible and platform. Obstacles are solid barriers. The exit requires all food collection before level completion.

**The Problem:**\
Levels feel samey because all puzzles rely on the same basic mechanics: navigate around obstacles, collect food, reach exit. There's no way to create:

- Block-pushing puzzles (Sokoban-style)
- Dynamic environments where objects react to player actions
- Timing-based challenges with falling objects
- Hazards that aren't just solid walls
- Exits with different platform behaviors

This limitation stems from the small object vocabulary and lack of interactive/dynamic object behaviors beyond the snake's movement and gravity.

## Goals

**Primary Goal:** Enable new gameplay experiences through expanded object vocabulary

- Sokoban-style mechanics (pushable stones)
- Physics-based puzzles (falling objects, reactive environments)
- Timing challenges (objects falling when support removed)
- Hazard variety (spikes vs obstacles)

**Secondary Goals:**

- Empower level designers with more creative building blocks
- Maintain clean core-web separation (all logic in `gsnake-core`)
- Keep contract changes minimal (add types, don't restructure)
- Improve system extensibility for future object additions

## Scope

**In Scope:**

- Four new object types: floating food, falling food, stones, spikes
- Enhanced exit with solid/fall-through property
- Flexible food requirement system (per-level `totalFood` field)
- Core mechanics implementation in `gsnake-core`
- Contract extensions (new `CellType` variants, `LevelDefinition` fields)
- Basic demonstration levels showcasing each new object

**Out of Scope:**

- Extensive level design and balancing (small scope: core mechanics only)
- UI/UX changes beyond rendering new cell types
- Comprehensive playtesting and iteration
- Animation or visual polish
- Level editor enhancements

## Constraints

- **Architectural:** All game behavior logic must remain in `gsnake-core`; `gsnake-web` only handles rendering
- **Contract:** Minimize changes to the core-web interface; prefer additive changes over restructuring
- **Scope:** Small effort (1-2 weeks) focused on core mechanics, not extensive content creation
- **Compatibility:** Existing levels must continue to work without modification

## Success Criteria

1. **Mechanics Work Correctly:** All new objects behave as specified (see detailed requirements from clarification)
1. **System Extensibility:** Adding future object types follows the same pattern established here
1. **Enhanced Engagement:** New objects enable puzzle types that weren't possible before
1. **Clean Implementation:** Contract changes are minimal and additive; existing code paths remain stable
