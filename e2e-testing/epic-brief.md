## Summary

This Epic establishes the foundation for automated end-to-end testing of Gravity Snake using Playwright. It focuses on validating the core game engine mechanics—specifically movement and gravity—by simulating user keyboard inputs and asserting against the DOM state in a real browser environment. The initiative includes integrating Allure for reporting and video recording for debugging, creating a scalable framework for future regression tests.

## Context & Problem

**Who's Affected:**
Developers maintaining the game engine and contributors adding new features or refactoring core logic.

**Current Pain:**
Verifying game physics (like gravity, collision, and movement) currently relies entirely on manual playtesting. This process is repetitive, time-consuming, and prone to human error. Regressions in core mechanics can easily slip through during refactors, leading to broken gameplay.

**The Solution:**
Implement a robust E2E testing infrastructure that interacts with the game exactly as a player does—via keyboard events—and verifies the outcome by inspecting the DOM (grid cells).

**Scope & Constraints:**

- **Framework:** Playwright with Allure reporting.
- **Initial Scope:** A Proof of Concept (PoC) test verifying the snake walks off a platform and falls.
- **Environment:** Tests run against a local build served at the root path (no subpath), effectively testing the build artifact directly.
- **Out of Scope:** Publishing tests to external pages or testing the GitHub Pages subpath deployment logic.
