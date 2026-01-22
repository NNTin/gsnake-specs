## Overview

This document defines the workflows for the E2E testing infrastructure. It covers both the **Developer Experience (DX)**—how engineers interact with the test suite—and the **Automated User Journeys**—the specific game scenarios simulated by Playwright to validate core mechanics.

---

## Flow 1: Developer Running E2E Suite

**Description:** A developer runs the automated test suite locally to validate game mechanics before committing changes.

**Trigger:** Developer runs `npm run test:e2e` in the terminal

**Steps:**

1.  Developer executes test command
2.  Playwright starts in headless mode (no visible browser UI)
3.  Local dev server spins up (or connects to existing one)
4.  Tests execute in parallel workers
5.  Terminal displays live progress (dots/ticks)
6.  **If Success:** Terminal shows green "Passed" summary with duration
7.  **If Failure:** Terminal shows red "Failed" summary with specific error details and diffs

**Exit:** Developer receives pass/fail confirmation in CLI

---

## Flow 2: Debugging via Allure Report

**Description:** A developer investigates a test failure using the visual reporting tool to see exactly what happened.

**Trigger:** Developer runs `npm run report` after a failed test run

**Steps:**

1.  Allure server starts and opens default web browser
2.  Dashboard displays overview of test suite status
3.  Developer navigates to "Behaviors" or "Suites" tab
4.  Developer selects the failed test case
5.  Report displays:
    *   **Step-by-step execution log** (e.g., "Press ArrowRight", "Expect .cell to have class")
    *   **Screenshot** of the failure state
    *   **Video recording** of the entire test session
6.  Developer plays video to observe the exact moment of failure (e.g., snake didn't fall when expected)

**Exit:** Developer identifies the root cause and closes the report server

---

## Flow 3: Test Scenario - Gravity Mechanics (PoC)

**Description:** The specific game scenario Playwright will execute to validate the "The Drop" level mechanics (gravity and movement).

**Trigger:** Automated test runner initiates the "Gravity Mechanics" spec

**Steps:**

1.  **Setup:**
    *   Browser loads the game application
    *   Script waits for DOM to be ready
    *   Script forces Game Engine to load "Level 2: The Drop" (or uses keyboard 'q' -> 'r' sequence if cheats aren't exposed)
    *   *Verification:* Assert snake head is at start position (2, 2)
2.  **Action - Move to Ledge:**
    *   Simulate `ArrowRight` key press (x3 times)
    *   Snake moves East across the platform
    *   *Verification:* Snake head reaches edge of platform (x=3)
    *   *Verification:* Snake does *not* fall yet (supported by body segments on platform)
3.  **Action - Walk Off Ledge:**
    *   Simulate `ArrowRight` key press (x3 more times) until all body segments leave the platform range (x=0 to x=3)
    *   Snake is now completely suspended over empty space at y=3
4.  **Reaction - Gravity Application:**
    *   Game engine processes gravity (instant update in MVP)
    *   Snake y-coordinates increase
5.  **Validation:**
    *   *Verification:* Assert Snake Head is NOT at the y-level of the ledge (y=2)
    *   *Verification:* Assert Snake Head has landed on the floor/platform below (e.g., y=7 or floor at y=14 depending on specific level geometry)

**Exit:** Test assertion passes if final coordinates match expected gravity resting point