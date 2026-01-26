# PRD: Level Editor Testing & Gap Identification

## Introduction

This PRD defines comprehensive testing requirements for the gSnake Level Editor to validate implementation against the specification. The primary focus is identifying gaps, missing features, and areas requiring improvement through systematic UI/UX testing using Browser MCP. This document also includes user stories for implementing any identified fixes.

## Goals

- Validate all specification requirements from `gsnake-specs/level-editor/` are implemented correctly
- Identify missing features, incomplete implementations, and bugs
- Test all user flows defined in the specification
- Verify UI/UX matches wireframes and design specifications
- Document current implementation status with evidence (screenshots, recordings)
- Create actionable user stories for fixing identified gaps
- Ensure seamless integration between editor, backend, and gsnake-web

## User Stories - Testing Phase

### US-T001: Test Landing Page
**Description:** As a tester, I want to verify the landing page displays correctly and functions as specified so that users can start creating or loading levels.

**Acceptance Criteria:**
- [ ] Navigate to `http://localhost:5173` using Browser MCP
- [ ] Verify page title is "gSnake Level Editor"
- [ ] Verify subtitle is "Create and edit levels for gSnake" (or similar)
- [ ] Verify "Create New Level" button is visible and styled per wireframe (green primary button)
- [ ] Verify "Load Existing Level" button is visible and styled per wireframe (secondary button)
- [ ] Verify centered card layout with proper spacing and shadows
- [ ] Verify responsive design (resize browser window)
- [ ] Take screenshot and save to `gsnake-specs/test-results/landing-page.png`

### US-T002: Test Create New Level Flow
**Description:** As a tester, I want to verify the "Create New Level" flow works correctly so that users can configure grid dimensions and access the editor.

**Acceptance Criteria:**
- [ ] Click "Create New Level" button using Browser MCP
- [ ] Verify grid size modal appears with overlay background
- [ ] Verify modal title is "Create New Level" or "Configure Grid Size"
- [ ] Verify width input field exists with default value 15, min 5, max 50
- [ ] Verify height input field exists with default value 15, min 5, max 50
- [ ] Verify "Cancel" button exists
- [ ] Verify "Create" button exists (primary green style)
- [ ] Test invalid input: Enter width=3, click Create, verify alert/error
- [ ] Test valid input: Enter width=20, height=20, click Create
- [ ] Verify editor interface loads with 20x20 grid
- [ ] Test Cancel button returns to landing page
- [ ] Take screenshots of modal and editor states

### US-T003: Test Editor Interface Layout
**Description:** As a tester, I want to verify the editor interface displays all required components correctly so that users can edit levels effectively.

**Acceptance Criteria:**
- [ ] Verify toolbar exists at top with light background
- [ ] Verify toolbar contains "New Level" button
- [ ] Verify toolbar contains "Load" button
- [ ] Verify toolbar contains "Undo" button
- [ ] Verify toolbar contains "Redo" button
- [ ] Verify toolbar contains "Direction" dropdown with options: North, South, East, West
- [ ] Verify toolbar contains "Test Level" button (red/danger style)
- [ ] Verify toolbar contains "Save Level" button (green/primary style)
- [ ] Verify left sidebar exists with entity palette
- [ ] Verify sidebar heading "Entity Palette" or similar
- [ ] Verify all 8 entity types are present: Snake, Obstacle, Food, Exit, Stone, Spike, Floating Food, Falling Food
- [ ] Verify each entity has colored icon matching specification
- [ ] Verify grid canvas is centered in main area
- [ ] Verify grid cells are 32x32px with 1px gaps
- [ ] Take full-page screenshot

### US-T004: Test Entity Palette Selection
**Description:** As a tester, I want to verify entity selection works correctly so that users can choose which entity to place.

**Acceptance Criteria:**
- [ ] Verify Snake entity is selected by default (highlighted border/background)
- [ ] Click Obstacle entity using Browser MCP
- [ ] Verify Obstacle becomes selected (highlighted)
- [ ] Verify previous selection (Snake) is no longer highlighted
- [ ] Click each of the 8 entity types in sequence
- [ ] Verify each selection updates visual state correctly
- [ ] Verify console logs (if any) show correct entity selection
- [ ] Take screenshot showing selected entity

### US-T005: Test Entity Placement - Click Mode
**Description:** As a tester, I want to verify click-to-place entity placement works correctly so that users can build levels efficiently.

**Acceptance Criteria:**
- [ ] Select "Obstacle" entity from palette
- [ ] Click on empty grid cell using Browser MCP
- [ ] Verify obstacle appears on clicked cell (dark grey color #666)
- [ ] Click on 5 different empty cells
- [ ] Verify all 5 obstacles appear on grid
- [ ] Click on cell with existing obstacle
- [ ] Verify obstacle remains (or is replaced, per spec)
- [ ] Select "Food" entity from palette
- [ ] Click on empty cell
- [ ] Verify food appears (orange color #ff9800)
- [ ] Click on cell with obstacle
- [ ] Verify food replaces obstacle (entity replacement behavior)
- [ ] Test all 8 entity types can be placed
- [ ] Take screenshot of grid with multiple entities

### US-T006: Test Snake Placement
**Description:** As a tester, I want to verify multi-segment snake placement works correctly so that users can create snake starting positions.

**Acceptance Criteria:**
- [ ] Select "Snake" entity from palette
- [ ] Click on empty grid cell (becomes head)
- [ ] Verify cell shows "H" label or similar head indicator
- [ ] Verify cell has green color (#4caf50)
- [ ] Click on adjacent cell
- [ ] Verify cell shows "1" label (body segment 1)
- [ ] Click on 3 more cells
- [ ] Verify segments show labels: H, 1, 2, 3, 4
- [ ] Verify all segments are visually connected/highlighted
- [ ] Select different entity (e.g., Food) to finalize snake
- [ ] Verify snake segments remain on grid
- [ ] Verify can place new entity
- [ ] Take screenshot of multi-segment snake

### US-T007: Test Entity Placement - Drag and Drop
**Description:** As a tester, I want to verify drag-and-drop entity placement works correctly so that users have flexible placement options.

**Acceptance Criteria:**
- [ ] Hover over "Stone" entity in palette using Browser MCP
- [ ] Initiate drag operation
- [ ] Verify custom drag image appears with entity color
- [ ] Drag over grid cell
- [ ] Verify drop indicator/hover effect on cell
- [ ] Drop on empty cell
- [ ] Verify stone entity appears on cell (#9e9e9e color)
- [ ] Drag "Spike" entity from palette
- [ ] Drop on cell with stone
- [ ] Verify spike replaces stone (entity replacement)
- [ ] Test drag-and-drop for all entity types
- [ ] Verify drag-drop does not change selected entity in palette
- [ ] Take screenshot showing drag operation

### US-T008: Test Entity Removal
**Description:** As a tester, I want to verify entity removal (Shift+Click) works correctly so that users can delete placed entities.

**Acceptance Criteria:**
- [ ] Place obstacle on grid cell
- [ ] Hold Shift key and click on obstacle cell using Browser MCP
- [ ] Verify obstacle is removed (cell becomes empty)
- [ ] Place snake segments (H, 1, 2)
- [ ] Shift+Click on segment 1
- [ ] Verify segment 1 is removed
- [ ] Verify remaining segments update indices (H, 1)
- [ ] Shift+Click on empty cell
- [ ] Verify no error, no visual change (graceful handling)
- [ ] Test removal for all entity types
- [ ] Verify red overlay appears on hover when Shift is pressed
- [ ] Take screenshot of removal operation

### US-T009: Test Undo/Redo Functionality
**Description:** As a tester, I want to verify undo/redo works correctly so that users can recover from mistakes.

**Acceptance Criteria:**
- [ ] Start with empty grid
- [ ] Place 3 different entities (obstacle, food, exit)
- [ ] Click "Undo" button
- [ ] Verify last entity (exit) is removed
- [ ] Click "Undo" button again
- [ ] Verify second entity (food) is removed
- [ ] Click "Redo" button
- [ ] Verify food reappears
- [ ] Test keyboard shortcut: Ctrl+Z (or Cmd+Z on Mac)
- [ ] Verify undo happens
- [ ] Test keyboard shortcut: Ctrl+Y (or Cmd+Shift+Z)
- [ ] Verify redo happens
- [ ] Place new entity after undo
- [ ] Verify redo stack is cleared (cannot redo after new action)
- [ ] Test undo/redo with snake placement
- [ ] Test undo/redo with entity removal
- [ ] Verify undo/redo buttons disable when stack is empty
- [ ] Take screenshot showing undo/redo in action

### US-T010: Test Snake Direction Configuration
**Description:** As a tester, I want to verify snake direction dropdown works correctly so that users can set initial snake facing direction.

**Acceptance Criteria:**
- [ ] Locate "Direction" dropdown in toolbar
- [ ] Verify default selection is "East"
- [ ] Click dropdown to open options
- [ ] Verify options: North, South, East, West
- [ ] Select "North"
- [ ] Verify dropdown shows "North" as selected
- [ ] Verify snake direction is updated in level data (check via save)
- [ ] Test all 4 directions
- [ ] Verify selection persists when editing level
- [ ] Take screenshot of dropdown open

### US-T011: Test Level Save Functionality
**Description:** As a tester, I want to verify level saving works correctly so that users can export their levels as JSON.

**Acceptance Criteria:**
- [ ] Create level with: snake (3 segments), 5 obstacles, 3 food, 1 exit, 2 stones
- [ ] Set snake direction to "West"
- [ ] Click "Save Level" button
- [ ] Verify save modal appears with overlay
- [ ] Verify modal has "Level Name" input field
- [ ] Verify modal has "Difficulty" dropdown (easy/medium/hard)
- [ ] Verify modal shows info: "Level ID will be auto-generated"
- [ ] Leave name empty and click "Export JSON"
- [ ] Verify alert/error for missing name
- [ ] Enter name "Test Level Alpha"
- [ ] Leave difficulty empty and click "Export JSON"
- [ ] Verify alert/error for missing difficulty
- [ ] Enter name "Test Level Alpha", select difficulty "medium"
- [ ] Click "Export JSON"
- [ ] Verify JSON file downloads (check browser downloads)
- [ ] Open downloaded JSON file
- [ ] Verify JSON contains: id, name="Test Level Alpha", difficulty="medium"
- [ ] Verify gridSize.width and gridSize.height match created grid
- [ ] Verify snake array has 3 positions
- [ ] Verify snakeDirection="West"
- [ ] Verify obstacles array has 5 positions
- [ ] Verify food array has 3 positions
- [ ] Verify exit object has single position
- [ ] Verify stones array has 2 positions
- [ ] Verify all positions use {x, y} format
- [ ] Verify level ID format matches `timestamp-random` pattern
- [ ] Take screenshots of save modal and downloaded JSON

### US-T012: Test Level Load Functionality
**Description:** As a tester, I want to verify level loading works correctly so that users can edit existing levels.

**Acceptance Criteria:**
- [ ] From landing page, click "Load Existing Level"
- [ ] Verify native file picker dialog appears
- [ ] Cancel file picker
- [ ] Verify returns to landing page (graceful cancel)
- [ ] Click "Load Existing Level" again
- [ ] Select a valid level JSON file from `gsnake-levels/levels/`
- [ ] Verify editor interface loads
- [ ] Verify grid dimensions match loaded level
- [ ] Verify all entities appear at correct positions
- [ ] Verify snake segments appear in correct order
- [ ] Verify snake direction dropdown shows loaded direction
- [ ] Count entities on grid and compare to JSON file
- [ ] Test loading invalid JSON file (create corrupted file)
- [ ] Verify error alert with helpful message
- [ ] Verify returns to landing page on error
- [ ] Test loading JSON with invalid gridSize (width=100)
- [ ] Verify validation error
- [ ] Test loading JSON with missing required field (no snake)
- [ ] Verify validation error
- [ ] Take screenshots of loaded level

### US-T013: Test Level Testing Integration
**Description:** As a tester, I want to verify the test level workflow works end-to-end so that users can test their levels in gsnake-web.

**Acceptance Criteria:**
- [ ] Verify backend server is running: `curl http://localhost:3001/api/test-level` returns 404 (no level saved yet)
- [ ] Create simple test level: snake (2 segments), 1 obstacle, 1 food, 1 exit
- [ ] Click "Test Level" button
- [ ] Verify POST request to `http://localhost:3001/api/test-level` succeeds (check browser network tab)
- [ ] Verify new browser tab/window opens with URL `http://localhost:3000?test=true`
- [ ] Verify gsnake-web loads in new tab
- [ ] Verify test level appears in gsnake-web
- [ ] Verify level is playable (can move snake)
- [ ] Verify level entities match editor (correct positions)
- [ ] Close gsnake-web tab and return to editor
- [ ] Modify level (add 1 more obstacle)
- [ ] Click "Test Level" again
- [ ] Verify updated level loads in gsnake-web
- [ ] Test with backend server NOT running
- [ ] Verify helpful error message
- [ ] Test with gsnake-web NOT running
- [ ] Verify browser opens but shows connection error
- [ ] Take screenshots of test workflow

### US-T014: Test New Level from Editor
**Description:** As a tester, I want to verify "New Level" button works correctly so that users can start fresh without page reload.

**Acceptance Criteria:**
- [ ] Create and edit a level with multiple entities
- [ ] Click "New Level" button in toolbar
- [ ] Verify confirmation (if any) or immediate action
- [ ] Verify returns to landing page
- [ ] Verify can create new level or load existing
- [ ] Verify previous level data is cleared
- [ ] Take screenshot of workflow

### US-T015: Test Load from Editor Toolbar
**Description:** As a tester, I want to verify "Load" button in toolbar works correctly so that users can switch levels without returning to landing page.

**Acceptance Criteria:**
- [ ] Create and edit a level
- [ ] Click "Load" button in toolbar
- [ ] Verify file picker dialog appears
- [ ] Select valid level JSON
- [ ] Verify new level loads, replacing current level
- [ ] Verify no unsaved changes warning (per spec: no validation)
- [ ] Cancel file picker
- [ ] Verify current level remains
- [ ] Take screenshot

### US-T016: Test Grid Responsiveness
**Description:** As a tester, I want to verify the grid displays correctly at different sizes so that the editor is usable on various screens.

**Acceptance Criteria:**
- [ ] Create grid with dimensions 5x5 (minimum)
- [ ] Verify grid renders correctly, cells are 32x32px
- [ ] Verify grid fits in viewport without excessive scrolling
- [ ] Create grid with dimensions 50x50 (maximum)
- [ ] Verify grid renders correctly
- [ ] Verify scrolling is available for large grids
- [ ] Create grid with non-square dimensions (10x30)
- [ ] Verify grid renders correctly
- [ ] Resize browser window to 1024px width
- [ ] Verify layout adapts (sidebar, grid, toolbar)
- [ ] Take screenshots of different grid sizes

### US-T017: Test Exit Entity Uniqueness (Gap Fix Verification)
**Description:** As a tester, I want to verify that only one exit entity can exist on the grid so that levels are valid.

**Acceptance Criteria:**
- [ ] Place exit entity at position (5, 5)
- [ ] Verify exit appears on grid (blue color #2196f3)
- [ ] Place exit entity at position (10, 10)
- [ ] Verify new exit appears at (10, 10)
- [ ] Verify previous exit at (5, 5) is automatically removed
- [ ] Verify console log shows "Removed previous exit at (5, 5)"
- [ ] Test with drag-and-drop
- [ ] Verify same behavior (only one exit exists)
- [ ] Save level and check JSON
- [ ] Verify exit object contains only one position
- [ ] Take screenshot showing single exit enforcement

### US-T018: Test Level ID Uniqueness
**Description:** As a tester, I want to verify level IDs are unique so that levels don't collide.

**Acceptance Criteria:**
- [ ] Create and save level with name "Level A"
- [ ] Note the generated level ID from JSON
- [ ] Immediately create and save another level with name "Level B"
- [ ] Note the generated level ID from JSON
- [ ] Verify IDs are different
- [ ] Verify ID format matches `timestamp-random` pattern (e.g., `1234567890-a1b2c3`)
- [ ] Verify timestamp portion is numeric
- [ ] Verify random portion is 6 alphanumeric characters
- [ ] Take screenshots of both JSON files showing unique IDs

## User Stories - Gap Fixes

### US-F001: Add Level Validation Warnings
**Description:** As a user, I want to receive warnings about potentially invalid levels before saving so that I can create functional levels.

**Acceptance Criteria:**
- [ ] When clicking "Save Level", check for common issues
- [ ] If no snake segments placed, show warning: "No snake placed - level is invalid"
- [ ] If no food placed, show warning: "No food placed - level may not be completable"
- [ ] If no exit placed, show warning: "No exit placed - level cannot be completed"
- [ ] Display warnings in save modal above export button
- [ ] Allow user to export anyway (warnings, not errors)
- [ ] Add "Export Anyway" and "Cancel" buttons when warnings present
- [ ] Typecheck passes
- [ ] Verify in browser using Browser MCP

### US-F002: Improve Error Messages
**Description:** As a user, I want to see helpful, specific error messages when things go wrong so that I can understand and fix issues.

**Acceptance Criteria:**
- [ ] Replace generic `alert()` calls with toast notification system
- [ ] Install toast library: `npm install svelte-french-toast`
- [ ] Add toast container to App.svelte
- [ ] Replace file load error alert with toast error
- [ ] Replace test level error alert with toast error
- [ ] Replace save validation alert with toast warning
- [ ] Add success toast when level saved successfully
- [ ] Add success toast when test level uploaded successfully
- [ ] Style toasts to match editor design (green success, red error, yellow warning)
- [ ] Auto-dismiss toasts after 5 seconds
- [ ] Typecheck passes
- [ ] Verify in browser using Browser MCP

### US-F003: Add Loading States
**Description:** As a user, I want to see loading indicators during async operations so that I know the system is working.

**Acceptance Criteria:**
- [ ] Add loading spinner to "Test Level" button while uploading
- [ ] Change button text to "Uploading..." during upload
- [ ] Disable button during upload to prevent double-clicks
- [ ] Add loading spinner when loading level JSON file
- [ ] Add loading indicator when fetching from backend (if applicable)
- [ ] Use consistent spinner design across all loading states
- [ ] Typecheck passes
- [ ] Verify in browser using Browser MCP

### US-F004: Add Keyboard Shortcuts Help
**Description:** As a user, I want to see available keyboard shortcuts so that I can work more efficiently.

**Acceptance Criteria:**
- [ ] Add "Help" button to toolbar (question mark icon)
- [ ] Clicking Help button opens modal with keyboard shortcuts
- [ ] Display shortcuts:
  - Ctrl+Z / Cmd+Z: Undo
  - Ctrl+Y / Cmd+Shift+Z: Redo
  - Shift+Click: Remove entity
  - Esc: Close modal/cancel
- [ ] Add close button to help modal
- [ ] Style modal consistently with other modals
- [ ] Typecheck passes
- [ ] Verify in browser using Browser MCP

### US-F005: Automate Backend Server Startup
**Description:** As a developer, I want the backend server to start automatically with the dev server so that I don't have to run two commands.

**Acceptance Criteria:**
- [ ] Install concurrently: `npm install --save-dev concurrently`
- [ ] Update package.json dev script to run both vite and server
- [ ] Add separate scripts: `dev:editor` (vite only), `dev:server` (server only)
- [ ] Test `npm run dev` starts both services
- [ ] Update README with new command
- [ ] Verify both servers run simultaneously without conflicts
- [ ] Typecheck passes

## Functional Requirements

### FR-T1: Landing Page
- The landing page must display at `http://localhost:5173` with centered card layout
- Must show "Create New Level" and "Load Existing Level" buttons
- Must match wireframe design from specification

### FR-T2: Grid Configuration
- Grid size modal must accept width and height between 5-50
- Must validate input and show error for invalid values
- Must default to 15x15

### FR-T3: Editor Interface
- Must display toolbar with all specified buttons: New, Load, Undo, Redo, Direction, Test, Save
- Must display entity palette sidebar with all 8 entity types
- Must display grid canvas with 32x32px cells and 1px gaps
- Grid must be scrollable for large dimensions

### FR-T4: Entity Placement
- Must support click-to-select and click-to-place interaction
- Must support drag-and-drop from palette to grid
- Must replace existing entities when placing on occupied cell
- Must support multi-segment snake placement with visual labels

### FR-T5: Entity Removal
- Must support Shift+Click to remove entities
- Must show red overlay on hover when Shift is pressed
- Must update snake segment indices when removing segments

### FR-T6: Undo/Redo
- Must support undo/redo with buttons and keyboard shortcuts
- Must maintain command history with full state restoration
- Must clear redo stack when new action is performed
- Must disable buttons when stacks are empty

### FR-T7: Level Save
- Must open save modal prompting for name and difficulty
- Must validate required fields (name, difficulty)
- Must generate unique level ID with format `timestamp-random`
- Must export JSON with all entities in correct format
- Must download JSON file to browser

### FR-T8: Level Load
- Must open file picker for JSON file selection
- Must parse and validate JSON structure
- Must validate grid dimensions (5-50)
- Must validate required fields (snake, snakeDirection, gridSize)
- Must reconstruct grid with all entities at correct positions
- Must show error and return to landing page on invalid file

### FR-T9: Level Testing
- Must POST level JSON to `http://localhost:3001/api/test-level`
- Must open `http://localhost:3000?test=true` in new tab
- Must show error if backend server not running
- gsnake-web must load level from test endpoint

### FR-T10: Exit Entity Uniqueness
- Must enforce only one exit entity on grid at any time
- Must automatically remove previous exit when placing new exit
- Must work for both click-to-place and drag-and-drop

### FR-T11: Level ID Generation
- Must generate unique level IDs using timestamp + random component
- Must use format `{timestamp}-{6-char-random}`
- Must prevent ID collisions even when saving rapidly

## Non-Goals

- No automated unit tests in this testing phase (manual Browser MCP testing only)
- No cross-browser compatibility testing in initial phase
- No mobile/touch interface testing
- No performance/load testing
- No accessibility (screen reader) testing in initial phase
- No internationalization (i18n) testing

## Technical Considerations

- Testing will use Browser MCP for automated UI interaction
- Screenshots should be saved to `gsnake-specs/test-results/`
- Test results should be documented in `gsnake-specs/test-results/RESULTS.md`
- Backend server must be running on port 3001 for integration tests
- gsnake-web must be running on port 3000 for integration tests
- Tests should be run in Chrome or Chromium browser

## Success Metrics

- 100% of specification requirements verified via testing
- All identified gaps have corresponding fix user stories
- All critical bugs fixed (exit uniqueness, ID generation)
- All user flows from specification work end-to-end
- Complete test results documentation with screenshots
- Zero critical issues remaining before release

## Open Questions

- Should we add automated Playwright/Puppeteer tests after manual testing?
- Should we test with real level files from `gsnake-levels/levels/`?
- Do we need to test with very large grids (40x40, 50x50) for performance?
- Should we verify exported JSON files can be validated by `gsnake-levels` CLI?
- Do we need to test rapid save operations to verify ID uniqueness under stress?

## Test Results Documentation

After completing all testing user stories, create `gsnake-specs/test-results/RESULTS.md` with:

1. **Summary**: Pass/fail for each FR-T requirement
2. **Screenshots**: All captured screenshots organized by test scenario
3. **Identified Gaps**: List of all gaps found with severity (critical/medium/low)
4. **Browser Console Logs**: Any errors or warnings observed
5. **Performance Notes**: Any lag, slowness, or UX friction points
6. **Recommendations**: Prioritized list of fixes and improvements

## Implementation Order

### Phase 1: Core Testing (US-T001 to US-T010)
Test fundamental editor functionality: landing page, grid creation, entity placement, undo/redo

### Phase 2: Save/Load Testing (US-T011 to US-T012)
Test level persistence and JSON format

### Phase 3: Integration Testing (US-T013 to US-T015)
Test backend and gsnake-web integration

### Phase 4: Edge Cases & Fixes (US-T016 to US-T018)
Test edge cases and verify applied fixes

### Phase 5: Gap Fixes (US-F001 to US-F005)
Implement fixes for identified gaps

## Notes

- This PRD focuses on manual testing with Browser MCP for comprehensive UI/UX validation
- Each test user story should produce screenshots and detailed notes
- Gap fix user stories should be implemented after testing phase
- All tests should be repeatable and documented for future regression testing
- Testing should reveal current state: what works, what doesn't, what needs improvement
