## Overview

This document defines the user flows for Gravity Snake, a turn-based puzzle game where players navigate a snake through levels using directional movement and gravity mechanics. The game emphasizes strategic planning over reflexes.

______________________________________________________________________

## Flow 1: Game Initialization

**Description:** The player loads the game and begins playing.

**Trigger:** Page load

**Steps:**

1. Game loads and immediately displays Level 1
1. Game field shows:

- Snake at starting position (3 segments)
- Food item(s) placed in level
- Static obstacles/platforms
- Exit/goal tile

3. Score display shows: Level 1, Length 3, Moves 0
1. Restart button is visible and active
1. Game is ready for player input (keyboard listening active)

**Exit:** Player is in active gameplay state, ready to make first move

______________________________________________________________________

## Flow 2: Core Gameplay Loop

**Description:** The moment-to-moment interaction where the player makes moves, gravity applies, and game state updates.

**Trigger:** Player presses arrow key or WASD key

**Steps:**

1. Player presses directional key (North/South/East/West)
1. Snake head moves one tile in the pressed direction
1. Snake body follows (each segment moves to previous segment's position)
1. Moves counter increments by 1
1. Gravity applies instantly:

- Snake falls downward until head hits obstacle, floor, or snake body
- Snake appears at final position (no animation)

6. Game checks collision:

- If snake head hits obstacle or own body → Go to Flow 4 (Game Over)
- Otherwise, continue

7. Game checks food collection:

- If snake head is on food tile → Snake length increases by 1, food respawns at new location
- Length display updates

8. Game checks exit condition:

- If snake head is on exit tile AND food has been collected → Go to Flow 3 (Level Completion)
- Otherwise, continue

9. Game returns to waiting for next player input

**Exit:** Game is ready for next player move, or transitions to Level Completion/Game Over

```mermaid
sequenceDiagram
    participant Player
    participant Game
    participant Snake
    participant Physics
    
    Player->>Game: Press directional key
    Game->>Snake: Move head in direction
    Snake->>Snake: Body follows head
    Game->>Game: Increment moves counter
    Game->>Physics: Apply gravity
    Physics->>Snake: Fall to bottom/obstacle
    Snake-->>Game: Final position
    Game->>Game: Check collision
    alt Collision detected
        Game->>Game: Trigger Game Over
    else No collision
        Game->>Game: Check food collection
        alt Food collected
            Game->>Snake: Increase length
            Game->>Game: Respawn food
        end
        Game->>Game: Check exit condition
        alt On exit with food collected
            Game->>Game: Trigger Level Complete
        else Not complete
            Game->>Player: Ready for next input
        end
    end
```

______________________________________________________________________

## Flow 3: Level Completion & Progression

**Description:** Player successfully completes a level by collecting food and reaching the exit.

**Trigger:** Snake head reaches exit tile after collecting all required food

**Steps:**

1. Level completion is detected
1. Next level loads instantly (no transition message)
1. Game field updates to show new level layout:

- Snake resets to starting position (3 segments)
- New obstacle/platform configuration
- New food placement
- New exit location

4. Score display updates: Level number increments
1. Moves counter resets to 0
1. Length resets to 3
1. Game is ready for player input

**Special Case - Final Level:**

- If player completes the last level:
  - "All Levels Complete!" overlay appears
  - Message: "Congratulations! You've beaten all levels."
  - Subtext: "Refresh the page to play again."
  - No buttons (game ends)

**Exit:** Player is in active gameplay state on new level, or game has ended (final level)

______________________________________________________________________

## Flow 4: Game Over & Recovery

**Description:** Player collides with obstacle or own body, triggering game over state.

**Trigger:** Snake head collides with obstacle or snake body segment

**Steps:**

1. Collision is detected
1. "Game Over" overlay appears immediately
1. Overlay displays two buttons:

- "Restart Level" (primary action)
- "Back to Level 1" (secondary action)

4. Game waits for player choice:

- **If "Restart Level":** Current level reloads with reset state (snake at start, moves = 0, length = 3)
- **If "Back to Level 1":** Level 1 loads with reset state

5. Overlay closes
1. Game is ready for player input

**Exit:** Player is in active gameplay state on chosen level

______________________________________________________________________

## Flow 5: Manual Restart

**Description:** Player clicks the Restart button during active gameplay.

**Trigger:** Player clicks "Restart" button in header

**Steps:**

1. Current level immediately reloads
1. Game state resets:

- Snake returns to starting position (3 segments)
- Moves counter resets to 0
- Length resets to 3
- Food and obstacles return to initial positions

3. Game is ready for player input

**Exit:** Player is in active gameplay state on same level with fresh start

______________________________________________________________________

## Flow 6: Input Blocking During Gravity

**Description:** Ensures player cannot input moves while gravity is resolving.

**Trigger:** Gravity begins applying after a player move

**Steps:**

1. Player makes a move (directional key press)
1. Input handler is disabled
1. Gravity resolves instantly (snake falls to final position)
1. Input handler is re-enabled
1. Game is ready for next player input

**Note:** Since gravity is instant (no animation), this blocking is imperceptible to the player but prevents input queue issues.

______________________________________________________________________

## Key Interaction Principles

### Information Hierarchy

- **Primary:** Game field (largest visual element, center focus)
- **Secondary:** Score metrics (level, length, moves) - always visible but not intrusive
- **Tertiary:** Restart button - available but not prominent

### State Communication

- **Active Play:** Clean game field, no overlays
- **Game Over:** Modal overlay with clear options
- **Game Complete:** Modal overlay with congratulatory message
- **Level Transition:** Instant (no loading state needed)

### Input Model

- Keyboard-only (arrow keys or WASD)
- Turn-based (one move at a time)
- Input blocked during gravity resolution
- No move queuing or buffering
- **Opposite direction prevention:** Cannot make 180-degree turns (e.g., cannot move South if currently moving North). This prevents accidental self-collision and maintains game flow.

### Visual Feedback

- Snake head: Distinct color from body
- Snake body: Connected segments
- Food: Clearly visible and distinct
- Obstacles: Solid, impassable appearance
- Exit: Visually distinct goal marker
- No animations (instant state changes)
