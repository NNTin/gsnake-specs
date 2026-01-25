# PRD: gSnake Visual Level Editor

## Introduction

The gSnake Level Editor is a standalone web application that enables visual level creation through an intuitive drag-and-drop interface. Currently, level creation requires manual JSON editing in the `gsnake-levels/levels/` directory, which is error-prone, tedious, and lacks visual feedback. This editor will dramatically reduce level creation time by providing immediate visual feedback, a seamless edit-test cycle, and support for all game entities. The tool serves both the core development team and potential future community contributors, enabling faster iteration and higher quality level designs.

## Goals

- Reduce time from concept to playable level by 10x through visual editing
- Enable rapid iteration with immediate visual feedback and undo/redo support
- Lower barrier to entry for level creation (no JSON knowledge required)
- Support all gSnake entity types (snake, obstacles, food, exit, stones, spikes, floating/falling food)
- Provide seamless testing workflow with gsnake-web integration
- Generate valid JSON output compatible with existing gsnake-levels infrastructure

## User Stories

### US-001: Project Setup and Build Configuration
**Description:** As a developer, I need a new gsnake-editor project with Vite + Svelte so the editor can be developed and deployed independently.

**Acceptance Criteria:**
- [ ] Create `gsnake-editor/` directory at repository root
- [ ] Initialize Vite + Svelte project with TypeScript support
- [ ] Configure build output and dev server
- [ ] Add npm scripts for dev, build, preview
- [ ] Create `.gitignore` for node_modules and build artifacts
- [ ] Typecheck passes
- [ ] Verify dev server starts successfully

### US-002: Landing Page with Level Creation Options
**Description:** As a user, I want to choose between creating a new level or loading an existing one when I open the editor.

**Acceptance Criteria:**
- [ ] Implement landing page matching wireframe design
- [ ] "Create New Level" button triggers grid size configuration modal
- [ ] "Load Existing Level" button opens file picker for JSON upload
- [ ] Page centered with proper styling (white card, shadow, rounded corners)
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-003: Grid Size Configuration Modal
**Description:** As a user, I want to specify grid dimensions when creating a new level so I can design levels of different sizes.

**Acceptance Criteria:**
- [ ] Modal appears when "Create New Level" is clicked
- [ ] Width input field (default: 15, min: 5, max: 50)
- [ ] Height input field (default: 15, min: 5, max: 50)
- [ ] Cancel button returns to landing page
- [ ] Create button validates inputs and opens editor
- [ ] Modal styling matches wireframe (overlay, centered, shadow)
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-004: Editor Interface Layout
**Description:** As a user, I need a visual editor interface with toolbar, sidebar, and grid canvas so I can edit levels efficiently.

**Acceptance Criteria:**
- [ ] Top toolbar with New Level, Load, Undo, Redo, Snake Direction, Test, Save buttons
- [ ] Left sidebar (200px) with entity palette
- [ ] Center canvas area with scrollable grid container
- [ ] Layout matches wireframe design exactly
- [ ] Responsive to window resizing
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-005: Entity Palette Implementation
**Description:** As a user, I want to see all available entities in a sidebar palette so I can select which entity to place.

**Acceptance Criteria:**
- [ ] Entity palette shows all 8 entity types: Snake, Obstacle, Food, Exit, Stone, Spike, Floating Food, Falling Food
- [ ] Each entity has colored icon (matches color scheme: snake=green, obstacle=gray, food=orange, exit=blue, stone=gray, spike=red, floating food=yellow, falling food=orange-red)
- [ ] Clicking entity selects it (visual highlight with border/background change)
- [ ] Snake selected by default on load
- [ ] Only one entity can be selected at a time
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-006: Grid Rendering with Configurable Dimensions
**Description:** As a user, I want to see an editable grid that matches my specified dimensions so I can visualize the level layout.

**Acceptance Criteria:**
- [ ] Grid renders with correct number of rows and columns based on configuration
- [ ] Each cell is 32x32 pixels
- [ ] Grid has 1px gap between cells (using background color)
- [ ] Grid container has white background, border, rounded corners, shadow
- [ ] Grid is centered in editor area
- [ ] Grid scrolls if larger than viewport
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-007: Place Entity by Clicking Grid Cells
**Description:** As a user, I want to click grid cells to place the selected entity so I can efficiently place multiple instances.

**Acceptance Criteria:**
- [ ] Clicking empty cell places selected entity
- [ ] Clicking occupied cell replaces existing entity (no warning)
- [ ] Selected entity remains active after placement (for multiple placements)
- [ ] Visual feedback on hover (cell background changes)
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-008: Place Entity by Drag-and-Drop
**Description:** As a user, I want to drag entities from the palette to the grid for quick single placements.

**Acceptance Criteria:**
- [ ] Dragging entity from palette shows entity icon on cursor
- [ ] Dropping on grid cell places entity
- [ ] Dropping on occupied cell replaces existing entity
- [ ] Palette selection unchanged after drag-and-drop
- [ ] Visual feedback during drag (cursor shows entity)
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-009: Visual Entity Representation on Grid
**Description:** As a user, I want to see visual representations of entities on the grid so I understand the level layout at a glance.

**Acceptance Criteria:**
- [ ] Entities render with colored backgrounds matching palette icons
- [ ] Snake segments show as connected green cells
- [ ] Each entity type visually distinguishable
- [ ] Entity rendering updates immediately on placement
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-010: Multi-Segment Snake Placement
**Description:** As a user, I want to place snake segments in sequence so I can define the snake's initial position and body.

**Acceptance Criteria:**
- [ ] First click with Snake selected places head
- [ ] Subsequent clicks add body segments to array
- [ ] Visual indication shows connected segments
- [ ] Segments stored in order (head to tail)
- [ ] Selecting different entity ends snake placement
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-011: Remove Entity with Shift+Click
**Description:** As a user, I want to remove entities from the grid by Shift+clicking so I can correct mistakes.

**Acceptance Criteria:**
- [ ] Holding Shift and clicking occupied cell removes entity
- [ ] Cell returns to empty state immediately
- [ ] Visual feedback on Shift hover (different cursor or highlight)
- [ ] Works for all entity types
- [ ] Action added to undo history
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-012: Undo/Redo Implementation
**Description:** As a user, I want to undo and redo editing actions so I can experiment without fear of mistakes.

**Acceptance Criteria:**
- [ ] Undo button reverses last action (placement or removal)
- [ ] Redo button reapplies previously undone action
- [ ] Ctrl+Z (Cmd+Z on Mac) triggers undo
- [ ] Ctrl+Y (Cmd+Shift+Z on Mac) triggers redo
- [ ] Undo/redo buttons disabled when history is empty
- [ ] Grid updates immediately to reflect state changes
- [ ] History preserved across entity placements/removals
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-013: Snake Direction Configuration
**Description:** As a user, I want to set the snake's initial direction so levels can start with different snake orientations.

**Acceptance Criteria:**
- [ ] Direction dropdown in toolbar with options: North, South, East, West
- [ ] Default direction is East
- [ ] Selecting direction updates level data immediately
- [ ] Direction persists when saving level
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-014: Load Existing Level from JSON
**Description:** As a user, I want to load an existing level JSON file so I can edit previously created levels.

**Acceptance Criteria:**
- [ ] "Load Existing Level" button opens browser file picker
- [ ] File picker accepts .json files
- [ ] Successfully parse valid level JSON
- [ ] Grid resizes to match loaded level dimensions
- [ ] All entities render at correct positions
- [ ] Snake direction updates from JSON
- [ ] Error message displays for invalid JSON (with explanation)
- [ ] On error, user returns to landing page
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-015: Save Level Modal with Metadata
**Description:** As a user, I want to provide level name and difficulty when saving so the exported JSON has complete metadata.

**Acceptance Criteria:**
- [ ] "Save Level" button opens save modal
- [ ] Modal prompts for level name (text input)
- [ ] Modal prompts for difficulty (dropdown: easy/medium/hard)
- [ ] Info box states "Level ID will be auto-generated"
- [ ] Cancel button closes modal without saving
- [ ] Export button validates inputs (name required, difficulty required)
- [ ] Modal styling matches wireframe
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-016: Export Level as JSON
**Description:** As a user, I want to download the level as a JSON file so I can add it to the gsnake-levels directory.

**Acceptance Criteria:**
- [ ] Generate unique level ID (timestamp-based or UUID)
- [ ] Include user-provided name and difficulty in JSON
- [ ] Include all entity positions and types
- [ ] Include grid width and height
- [ ] Include snake direction
- [ ] JSON structure matches existing gsnake-levels format
- [ ] Browser downloads file as `level-[id].json`
- [ ] Modal closes after successful export
- [ ] Typecheck passes

### US-017: Backend Service for Test Level Storage
**Description:** As a developer, I need a simple HTTP server to store test levels temporarily so users can test levels seamlessly.

**Acceptance Criteria:**
- [ ] Create simple Node.js/Express HTTP server in gsnake-editor/
- [ ] POST `/api/test-level` endpoint accepts level JSON
- [ ] Store test level in memory (no database needed)
- [ ] GET `/api/test-level` endpoint returns stored test level
- [ ] Test level expires after 1 hour or on server restart
- [ ] CORS headers configured for localhost development
- [ ] Server runs on separate port from editor (e.g., 3001)
- [ ] Typecheck passes

### US-018: Test Level Integration
**Description:** As a user, I want to test my level in gsnake-web with one click so I can quickly iterate on designs.

**Acceptance Criteria:**
- [ ] "Test Level" button saves current level to backend via POST
- [ ] Backend returns success confirmation
- [ ] Editor opens gsnake-web in new tab/window with test mode parameter
- [ ] gsnake-web detects test mode and fetches level from backend
- [ ] User can play the test level immediately
- [ ] Editor tab remains open for further editing
- [ ] Error message if backend unavailable
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill (test workflow end-to-end)

### US-019: New Level Button in Editor
**Description:** As a user, I want to start a new level while in the editor so I don't have to reload the page.

**Acceptance Criteria:**
- [ ] "New Level" button in toolbar returns to landing page
- [ ] Current work not automatically saved (user must use Save button)
- [ ] Confirmation dialog if unsaved changes exist (optional enhancement)
- [ ] Landing page loads with both creation options available
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-020: Load Button in Editor Toolbar
**Description:** As a user, I want to load a different level while in the editor so I can switch between editing multiple levels.

**Acceptance Criteria:**
- [ ] "Load" button in toolbar opens file picker
- [ ] File picker accepts .json files
- [ ] Successfully loaded level replaces current editor state
- [ ] Grid resizes and re-renders with new level data
- [ ] Undo/redo history resets on load
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

## Functional Requirements

**Grid & Canvas**
- FR-1: Grid must support configurable dimensions from 5x5 to 50x50 cells
- FR-2: Each grid cell must be 32x32 pixels with 1px gaps
- FR-3: Grid must scroll if dimensions exceed viewport
- FR-4: Grid must render centered in editor area

**Entity Management**
- FR-5: System must support all 8 entity types: Snake, Obstacle, Food, Exit, Stone, Spike, Floating Food, Falling Food
- FR-6: Each entity type must have distinct visual representation (colored cell/icon)
- FR-7: Snake must support multi-segment placement with connected visual representation
- FR-8: Placing entity on occupied cell must replace existing entity without confirmation

**Interaction**
- FR-9: Users must be able to place entities by clicking grid cells (click-to-place mode)
- FR-10: Users must be able to place entities by dragging from palette (drag-and-drop mode)
- FR-11: Users must be able to remove entities by Shift+clicking occupied cells
- FR-12: Selected entity in palette must remain active for multiple placements

**Undo/Redo**
- FR-13: System must maintain undo history for all placement and removal actions
- FR-14: Undo must be triggered by Ctrl+Z (Cmd+Z on Mac) or toolbar button
- FR-15: Redo must be triggered by Ctrl+Y (Cmd+Shift+Z on Mac) or toolbar button
- FR-16: Undo/redo buttons must be disabled when respective histories are empty

**File Operations**
- FR-17: System must parse and load existing level JSON files
- FR-18: System must validate JSON structure and show error messages for invalid files
- FR-19: System must generate unique level IDs (timestamp or UUID-based)
- FR-20: System must export levels as JSON matching gsnake-levels format
- FR-21: Exported JSON must include: id, name, difficulty, grid dimensions, entity positions, snake direction

**Testing Integration**
- FR-22: Backend service must provide POST `/api/test-level` endpoint for storing test levels
- FR-23: Backend service must provide GET `/api/test-level` endpoint for retrieving test levels
- FR-24: Test levels must expire after 1 hour or on server restart
- FR-25: Editor must open gsnake-web in new tab with test mode enabled when "Test Level" is clicked
- FR-26: gsnake-web must detect test mode and fetch level from backend automatically

**Configuration**
- FR-27: Users must be able to set initial snake direction (North/South/East/West)
- FR-28: Default snake direction must be East
- FR-29: Grid size defaults must be 15x15 with min 5x5 and max 50x50

## Non-Goals (Out of Scope)

- No built-in level validation (validation handled by existing gsnake-levels CLI)
- No automatic file system integration (users manually place JSON in levels/ directory)
- No multiplayer/collaborative editing
- No version control or level history within the editor
- No in-editor gameplay simulation (testing uses actual gsnake-web)
- No level templates or pre-built assets library
- No authentication or user accounts
- No cloud storage or level sharing features
- No undo/redo for metadata changes (only entity placement/removal)
- No custom entity creation or modification
- No physics simulation or gravity preview
- No automated testing or validation of level solvability

## Design Considerations

**UI/UX Requirements**
- Follow wireframe designs exactly for production-ready polish
- Use modern, clean aesthetic with rounded corners, shadows, and smooth transitions
- Implement hover states for all interactive elements (buttons, cells, entities)
- Use color scheme:
  - Snake: #4caf50 (green)
  - Obstacle: #666 (dark gray)
  - Food: #ff9800 (orange)
  - Exit: #2196f3 (blue)
  - Stone: #9e9e9e (gray)
  - Spike: #f44336 (red)
  - Floating Food: #ffc107 (yellow)
  - Falling Food: #ff5722 (deep orange)
- Primary action buttons: green (#4caf50)
- Destructive actions: red (#f44336)
- Secondary actions: gray (#e0e0e0)

**Interaction Principles**
- Immediate visual feedback for all actions (no delays)
- Support both drag-and-drop and click-to-place workflows
- Forgiving editing with undo/redo support
- No validation warnings during editing (users can create any configuration)
- Clear state communication (selected entity always highlighted)

**Component Reusability**
- Create reusable Svelte components for: Modal, Button, EntityPalette, GridCell, Toolbar
- Use consistent styling system (CSS variables or Tailwind)
- Separate business logic from presentation components

## Technical Considerations

**Architecture**
- Separate web application in `gsnake-editor/` directory
- Independent build and deployment from gsnake-web
- Technology stack: Vite + Svelte + TypeScript (consistent with gsnake-web)
- Backend: Simple Node.js/Express server for test level storage
- Client-side focused: Minimal backend dependency

**State Management**
- Level state stored in Svelte store or reactive variables
- Undo/redo implemented with command pattern (action history stack)
- File operations entirely client-side (FileReader API for load, Blob/download for save)

**Backend Service**
- Simple HTTP server running on port 3001
- In-memory storage (Map or object) for test levels
- CORS enabled for localhost development
- Single test level per session (overwrites on each test)
- No persistence required (test levels are temporary)

**Integration Points**
- gsnake-web must support test mode query parameter (e.g., `?test=true`)
- gsnake-web must fetch test level from `http://localhost:3001/api/test-level` when in test mode
- JSON format must match existing gsnake-levels schema exactly

**Performance Requirements**
- Grid rendering must support up to 50x50 (2500 cells) without lag
- Entity placement must be instant (< 16ms for 60fps)
- Undo/redo must be instant regardless of history size
- File parsing must complete in < 500ms for typical level files

**Browser Compatibility**
- Target modern browsers (Chrome, Firefox, Safari, Edge - latest 2 versions)
- Use standard File API for load/save (no polyfills required)
- Drag-and-drop using HTML5 drag events

## Success Metrics

**Efficiency**
- Reduce level creation time from 30+ minutes (manual JSON) to 3-5 minutes (visual editor)
- Enable complete edit-test-refine cycle in under 1 minute

**Quality**
- Zero JSON syntax errors in exported levels (validated by gsnake-levels CLI)
- 100% entity type coverage (all 8 types supported)
- Support 100% of existing level configurations (no feature regression)

**Usability**
- User can create first level without documentation
- Entity placement requires < 2 actions (select + click or drag + drop)
- Testing workflow requires < 3 clicks (Test button → play in gsnake-web → return)

**Technical**
- No performance degradation up to 50x50 grid size
- Undo/redo history supports minimum 50 actions
- JSON export matches gsnake-levels schema 100%

## Open Questions

1. Should we add keyboard shortcuts for entity selection (1-8 keys)?
2. Should the editor validate required entities (e.g., at least one snake, one exit)?
3. Should we add a "recent levels" list on the landing page?
4. Should there be a visual preview mode (read-only view without grid lines)?
5. Should we support copy/paste for entity groups or level sections?
6. Should the backend support multiple concurrent test levels (per user session)?
7. Should we add autosave functionality (localStorage) to prevent accidental data loss?
8. Should gsnake-web modifications be part of this PRD or a separate dependency?
