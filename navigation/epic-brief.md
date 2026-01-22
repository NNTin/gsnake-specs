## Summary

This Epic adds comprehensive keyboard navigation and shortcuts to gSnake, eliminating the need for mouse interaction during gameplay and modal interactions. Players will be able to restart levels ('r' key), return to level 1 ('q' key), and navigate modal dialogs entirely via keyboard. The feature targets keyboard-first players who currently experience flow interruption when forced to switch from keyboard (gameplay) to mouse (modal buttons). By enabling a fully keyboard-driven experience, we reduce friction in the retry loop and maintain player focus, particularly benefiting those who replay levels frequently or prefer speedrunning-style gameplay.

## Context & Problem

**Who's Affected:**  
Keyboard-first players who prefer not to use a mouse, especially those who retry levels frequently or engage in speedrunning-style play.

**Current Pain:**  
gSnake's gameplay is entirely keyboard-driven (WASD/Arrow keys for movement), but when the GameOverModal or GameCompleteModal appears, players must switch to their mouse to click "Restart Level" or "Back to Level 1" buttons. This context switch breaks the player's flow state and adds friction to the retry loop—a critical interaction in a puzzle game where players often attempt levels multiple times to find the optimal solution.

**Where in the Product:**

- During active gameplay (GameStatus.Playing)
- GameOverModal (appears when player fails a level)
- GameCompleteModal (appears when all levels are completed)

**Impact:**  
The forced keyboard-to-mouse transition disrupts the natural rhythm of play-retry-play cycles, making the game feel less polished and creating unnecessary friction for players who want to stay in a flow state. This is particularly noticeable for experienced players who iterate rapidly on level solutions.

## Scope

**In Scope:**

- Keyboard shortcuts during gameplay ('r' to restart level, 'q' to return to level 1, Escape does nothing)
- "Any-key restart" behavior in GameOverModal (any key except 'q' and Escape restarts level, 'q' and Escape go to level 1)
- Keyboard shortcuts in GameCompleteModal ('r', 'q', and Escape all restart from level 1)
- Modifier key filtering (Ctrl/Alt/Shift/Meta combinations are ignored to preserve browser shortcuts)
- Maintaining existing directional key behavior (WASD/Arrows for movement)

**Out of Scope:**

- In-game or README documentation of shortcuts (deferred for future consideration)
- Changes to visual design or modal layout
- Additional keyboard shortcuts beyond 'r' and 'q'
- Customizable key bindings

## Success Criteria

- Players can complete entire gameplay sessions without touching the mouse
- Keyboard shortcuts work consistently across all game states (gameplay, GameOverModal, GameCompleteModal)
- Existing directional controls (WASD/Arrows) continue to function normally during gameplay
- "Any-key restart" behavior in GameOverModal provides instant, frictionless retry experience
- Browser shortcuts (Ctrl+R, Alt+Q, etc.) continue to work normally

