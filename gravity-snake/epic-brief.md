## Summary

Gravity Snake is a strategic puzzle game that reimagines the classic Snake experience by introducing gravity physics and turn-based movement. Unlike traditional Snake games that rely on reflexes and continuous movement, Gravity Snake requires players to think ahead about how gravity will affect their position after each deliberate move. The snake only moves when the player acts, and after each action, gravity pulls the snake downward until it hits an obstacle or the floor. This creates a puzzle-like experience where players must navigate through increasingly complex levels with static obstacles and platforms. The MVP will be built as a Svelte SPA with minimal UI, focusing on core mechanics and level progression to validate the unique gameplay concept.

## Context & Problem

**Who's Affected:**

- Casual gamers looking for strategic, puzzle-oriented experiences
- Snake game enthusiasts seeking fresh takes on the classic formula
- Players who prefer thoughtful, turn-based gameplay over reflex-based challenges

**The Gap:**\
Traditional Snake games follow a well-established pattern: continuous movement, increasing speed, and reflex-based gameplay. While this formula has proven successful, it leaves little room for strategic thinking or puzzle-solving. Players who enjoy games like Sokoban, Stephen's Sausage Roll, or other grid-based puzzlers don't have a Snake variant that caters to their preference for deliberate, consequence-driven decision-making.

**Current Pain:**\
There's no readily available Snake game that combines:

- Turn-based, move-action mechanics (snake only moves on player input)
- Gravity physics that create strategic depth
- Level-based progression with designed obstacles
- Puzzle-solving elements where players must plan moves ahead

**What We're Building:**\
A web-based Snake game that transforms the genre from a reflex test into a strategic puzzle. By introducing gravity (snake falls after each move) and removing auto-movement, we create a game where every action has consequences that must be anticipated. Static obstacles and platforms in each level create puzzle scenarios where players must figure out the correct sequence of moves to collect food without trapping themselves.

**Success Looks Like:**

- Players can complete levels through strategic planning rather than quick reflexes
- The gravity mechanic creates interesting puzzle scenarios
- Level progression provides increasing challenge through obstacle complexity
- The core gameplay loop is engaging enough to validate further development

**Scope Appetite:**\
This is an MVP focused on proving the core mechanic. We're building:

- Essential game mechanics (movement, gravity, collision, food collection)
- Basic level progression system
- Minimal UI (game field, score, restart)
- Simple visual style (colored blocks)

We're explicitly NOT building (for MVP):

- Advanced UI features (menus, settings, high scores, level select)
- Visual polish (animations, particle effects, themed graphics)
- Mobile/touch controls
- Sound effects or music
- Level editor or user-generated content
