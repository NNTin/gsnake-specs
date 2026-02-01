## Overview

This document defines the user flows and interactions for the gSnake Level Editor. The editor enables visual level creation through an intuitive interface with drag-and-drop entity placement, immediate visual feedback, and seamless testing integration.

## Wireframes

### Landing Page

```wireframe
<!DOCTYPE html>
<html>
<head>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
body { 
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: #f5f5f5;
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  padding: 20px;
}
.container {
  background: white;
  border-radius: 12px;
  padding: 48px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
  max-width: 500px;
  width: 100%;
  text-align: center;
}
h1 {
  font-size: 32px;
  color: #222;
  margin-bottom: 12px;
}
.subtitle {
  color: #666;
  font-size: 16px;
  margin-bottom: 40px;
}
.button-group {
  display: flex;
  flex-direction: column;
  gap: 16px;
}
button {
  padding: 16px 24px;
  font-size: 16px;
  font-weight: 600;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: transform 0.15s ease, box-shadow 0.15s ease;
}
button:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(0,0,0,0.15);
}
.btn-primary {
  background: #4caf50;
  color: white;
}
.btn-secondary {
  background: #2196f3;
  color: white;
}
</style>
</head>
<body>
  <div class="container">
    <h1>gSnake Level Editor</h1>
    <p class="subtitle">Create and edit levels visually</p>
    <div class="button-group">
      <button class="btn-primary" data-element-id="create-new-level-btn">Create New Level</button>
      <button class="btn-secondary" data-element-id="load-existing-level-btn">Load Existing Level</button>
    </div>
  </div>
</body>
</html>
```

### Grid Size Configuration Modal

```wireframe
<!DOCTYPE html>
<html>
<head>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
body { 
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  padding: 20px;
}
.modal {
  background: white;
  border-radius: 12px;
  padding: 32px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.2);
  max-width: 400px;
  width: 100%;
}
h2 {
  font-size: 24px;
  color: #222;
  margin-bottom: 8px;
}
.description {
  color: #666;
  font-size: 14px;
  margin-bottom: 24px;
}
.form-group {
  margin-bottom: 20px;
}
label {
  display: block;
  font-size: 14px;
  font-weight: 600;
  color: #444;
  margin-bottom: 8px;
}
input {
  width: 100%;
  padding: 10px 12px;
  font-size: 16px;
  border: 2px solid #ddd;
  border-radius: 6px;
}
input:focus {
  outline: none;
  border-color: #4caf50;
}
.button-group {
  display: flex;
  gap: 12px;
  margin-top: 24px;
}
button {
  flex: 1;
  padding: 12px 20px;
  font-size: 14px;
  font-weight: 600;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}
.btn-primary {
  background: #4caf50;
  color: white;
}
.btn-secondary {
  background: #e0e0e0;
  color: #444;
}
</style>
</head>
<body>
  <div class="modal">
    <h2>Configure Grid Size</h2>
    <p class="description">Set the dimensions for your level</p>
    <div class="form-group">
      <label for="width">Width (cells)</label>
      <input type="number" id="width" value="15" min="5" max="50" data-element-id="grid-width-input">
    </div>
    <div class="form-group">
      <label for="height">Height (cells)</label>
      <input type="number" id="height" value="15" min="5" max="50" data-element-id="grid-height-input">
    </div>
    <div class="button-group">
      <button class="btn-secondary" data-element-id="cancel-btn">Cancel</button>
      <button class="btn-primary" data-element-id="create-btn">Create</button>
    </div>
  </div>
</body>
</html>
```

### Level Editor Interface

```wireframe
<!DOCTYPE html>
<html>
<head>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
body { 
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: #f5f5f5;
  display: flex;
  flex-direction: column;
  height: 100vh;
}
.toolbar {
  background: white;
  border-bottom: 1px solid #ddd;
  padding: 12px 20px;
  display: flex;
  align-items: center;
  gap: 16px;
}
.toolbar-section {
  display: flex;
  align-items: center;
  gap: 8px;
}
.toolbar-divider {
  width: 1px;
  height: 24px;
  background: #ddd;
  margin: 0 8px;
}
button {
  padding: 8px 16px;
  font-size: 14px;
  font-weight: 500;
  border: 1px solid #ddd;
  border-radius: 6px;
  background: white;
  cursor: pointer;
}
button:hover {
  background: #f5f5f5;
}
.btn-primary {
  background: #4caf50;
  color: white;
  border-color: #4caf50;
}
.btn-primary:hover {
  background: #45a049;
}
.btn-danger {
  background: #f44336;
  color: white;
  border-color: #f44336;
}
.btn-danger:hover {
  background: #da190b;
}
select {
  padding: 8px 12px;
  font-size: 14px;
  border: 1px solid #ddd;
  border-radius: 6px;
  background: white;
}
.main-content {
  display: flex;
  flex: 1;
  overflow: hidden;
}
.sidebar {
  width: 200px;
  background: white;
  border-right: 1px solid #ddd;
  padding: 16px;
  overflow-y: auto;
}
.sidebar h3 {
  font-size: 14px;
  color: #666;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 12px;
}
.entity-palette {
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.entity-item {
  padding: 12px;
  border: 2px solid #ddd;
  border-radius: 6px;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 8px;
  transition: all 0.15s ease;
}
.entity-item:hover {
  border-color: #999;
  background: #f9f9f9;
}
.entity-item.selected {
  border-color: #4caf50;
  background: #e8f5e9;
}
.entity-icon {
  width: 24px;
  height: 24px;
  border-radius: 4px;
}
.icon-snake { background: #4caf50; }
.icon-obstacle { background: #666; }
.icon-food { background: #ff9800; }
.icon-exit { background: #2196f3; }
.icon-stone { background: #9e9e9e; }
.icon-spike { background: #f44336; }
.icon-floating-food { background: #ffc107; }
.icon-falling-food { background: #ff5722; }
.entity-name {
  font-size: 14px;
  color: #333;
}
.editor-area {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  overflow: auto;
}
.grid-container {
  background: white;
  border: 2px solid #ddd;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}
.grid {
  display: grid;
  gap: 1px;
  background: #ddd;
  border: 1px solid #ddd;
}
.grid-cell {
  width: 32px;
  height: 32px;
  background: white;
  cursor: pointer;
  position: relative;
}
.grid-cell:hover {
  background: #f0f0f0;
}
.grid-cell.has-entity {
  display: flex;
  align-items: center;
  justify-content: center;
}
</style>
</head>
<body>
  <div class="toolbar">
    <div class="toolbar-section">
      <button data-element-id="new-level-btn">New Level</button>
      <button data-element-id="load-level-btn">Load</button>
    </div>
    <div class="toolbar-divider"></div>
    <div class="toolbar-section">
      <button data-element-id="undo-btn">Undo</button>
      <button data-element-id="redo-btn">Redo</button>
    </div>
    <div class="toolbar-divider"></div>
    <div class="toolbar-section">
      <label style="font-size: 14px; color: #666;">Direction:</label>
      <select data-element-id="snake-direction-select">
        <option value="East" selected>East</option>
        <option value="West">West</option>
        <option value="North">North</option>
        <option value="South">South</option>
      </select>
    </div>
    <div style="flex: 1;"></div>
    <div class="toolbar-section">
      <button class="btn-danger" data-element-id="test-level-btn">Test Level</button>
      <button class="btn-primary" data-element-id="save-level-btn">Save Level</button>
    </div>
  </div>
  
  <div class="main-content">
    <div class="sidebar">
      <h3>Entity Palette</h3>
      <div class="entity-palette">
        <div class="entity-item selected" data-element-id="entity-snake">
          <div class="entity-icon icon-snake"></div>
          <span class="entity-name">Snake</span>
        </div>
        <div class="entity-item" data-element-id="entity-obstacle">
          <div class="entity-icon icon-obstacle"></div>
          <span class="entity-name">Obstacle</span>
        </div>
        <div class="entity-item" data-element-id="entity-food">
          <div class="entity-icon icon-food"></div>
          <span class="entity-name">Food</span>
        </div>
        <div class="entity-item" data-element-id="entity-exit">
          <div class="entity-icon icon-exit"></div>
          <span class="entity-name">Exit</span>
        </div>
        <div class="entity-item" data-element-id="entity-stone">
          <div class="entity-icon icon-stone"></div>
          <span class="entity-name">Stone</span>
        </div>
        <div class="entity-item" data-element-id="entity-spike">
          <div class="entity-icon icon-spike"></div>
          <span class="entity-name">Spike</span>
        </div>
        <div class="entity-item" data-element-id="entity-floating-food">
          <div class="entity-icon icon-floating-food"></div>
          <span class="entity-name">Floating Food</span>
        </div>
        <div class="entity-item" data-element-id="entity-falling-food">
          <div class="entity-icon icon-falling-food"></div>
          <span class="entity-name">Falling Food</span>
        </div>
      </div>
    </div>
    
    <div class="editor-area">
      <div class="grid-container">
        <div class="grid" style="grid-template-columns: repeat(15, 32px);" data-element-id="level-grid">
          <!-- Grid cells would be generated dynamically -->
          <div class="grid-cell"></div>
          <div class="grid-cell"></div>
          <div class="grid-cell"></div>
          <!-- ... more cells ... -->
        </div>
      </div>
    </div>
  </div>
</body>
</html>
```

### Save Level Modal

```wireframe
<!DOCTYPE html>
<html>
<head>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
body { 
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  padding: 20px;
}
.modal {
  background: white;
  border-radius: 12px;
  padding: 32px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.2);
  max-width: 450px;
  width: 100%;
}
h2 {
  font-size: 24px;
  color: #222;
  margin-bottom: 8px;
}
.description {
  color: #666;
  font-size: 14px;
  margin-bottom: 24px;
}
.form-group {
  margin-bottom: 20px;
}
label {
  display: block;
  font-size: 14px;
  font-weight: 600;
  color: #444;
  margin-bottom: 8px;
}
input, select {
  width: 100%;
  padding: 10px 12px;
  font-size: 16px;
  border: 2px solid #ddd;
  border-radius: 6px;
}
input:focus, select:focus {
  outline: none;
  border-color: #4caf50;
}
.info-box {
  background: #e3f2fd;
  border-left: 4px solid #2196f3;
  padding: 12px;
  margin-bottom: 20px;
  font-size: 13px;
  color: #1565c0;
}
.button-group {
  display: flex;
  gap: 12px;
  margin-top: 24px;
}
button {
  flex: 1;
  padding: 12px 20px;
  font-size: 14px;
  font-weight: 600;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}
.btn-primary {
  background: #4caf50;
  color: white;
}
.btn-secondary {
  background: #e0e0e0;
  color: #444;
}
</style>
</head>
<body>
  <div class="modal">
    <h2>Save Level</h2>
    <p class="description">Provide level details before exporting</p>
    
    <div class="form-group">
      <label for="level-name">Level Name</label>
      <input type="text" id="level-name" placeholder="e.g., First Steps" data-element-id="level-name-input">
    </div>
    
    <div class="form-group">
      <label for="difficulty">Difficulty</label>
      <select id="difficulty" data-element-id="difficulty-select">
        <option value="">Select difficulty...</option>
        <option value="easy">Easy</option>
        <option value="medium">Medium</option>
        <option value="hard">Hard</option>
      </select>
    </div>
    
    <div class="info-box">
      Level ID will be auto-generated. The JSON file will be downloaded to your device.
    </div>
    
    <div class="button-group">
      <button class="btn-secondary" data-element-id="cancel-save-btn">Cancel</button>
      <button class="btn-primary" data-element-id="export-btn">Export JSON</button>
    </div>
  </div>
</body>
</html>
```

## User Flows

### Flow 1: Create New Level

**Description:** User creates a new level from scratch with custom grid dimensions.

**Trigger:** User clicks "Create New Level" button on landing page.

**Steps:**

1. User lands on the editor landing page
1. User clicks "Create New Level" button
1. Grid size configuration modal appears
1. User enters desired width and height (default: 15x15)
1. User clicks "Create" button
1. Editor interface loads with empty grid at specified dimensions
1. Entity palette shows on left sidebar with "Snake" selected by default
1. Toolbar displays with undo/redo, snake direction (default: East), test, and save buttons
1. User is ready to place entities on the grid

**Exit:** User is in the editor interface, ready to edit the level.

______________________________________________________________________

### Flow 2: Load Existing Level

**Description:** User loads an existing level JSON file for editing.

**Trigger:** User clicks "Load Existing Level" button on landing page.

**Steps:**

1. User lands on the editor landing page
1. User clicks "Load Existing Level" button
1. File upload dialog appears (browser native file picker)
1. User selects a level JSON file from their device
1. Editor parses the JSON file
1. If parsing succeeds:

- Editor interface loads with the level data
- Grid automatically resizes to match the loaded level's dimensions
- All entities from the JSON appear on the grid with visual representations
- Snake direction updates to match the loaded level

7. If parsing fails:

- Error message displays explaining the issue
- User returns to landing page

**Exit:** User is in the editor interface with the loaded level, ready to edit.

______________________________________________________________________

### Flow 3: Edit Level - Place Entities

**Description:** User places entities on the grid using drag-and-drop or click-to-place.

**Trigger:** User is in the editor interface.

**Steps:**

**Option A: Drag-and-Drop (Single Placement)**

1. User drags an entity from the palette
1. Cursor shows the entity icon while dragging
1. User drops the entity on a grid cell
1. Entity appears on the grid with visual representation (colored cell/icon)
1. If cell already has an entity, it's replaced without warning
1. Palette selection remains unchanged

**Option B: Click-to-Select (Multiple Placements)**

1. User clicks an entity in the palette
1. Selected entity highlights in the palette (border/background change)
1. User clicks grid cells to place the entity
1. Each click places the entity on that cell
1. Entity remains selected for multiple placements
1. If cell already has an entity, it's replaced without warning
1. User can click a different entity in the palette to switch

**Special Case: Snake Placement**

1. User selects "Snake" from palette
1. User clicks first grid cell (this becomes the snake head)
1. User clicks subsequent cells to add body segments
1. Each click adds a segment to the snake array
1. Visual feedback shows connected snake segments
1. User clicks a different entity to finish snake placement

**Exit:** Entities are placed on the grid, level state is updated.

______________________________________________________________________

### Flow 4: Edit Level - Remove Entities

**Description:** User removes entities from the grid.

**Trigger:** User wants to delete an entity from the grid.

**Steps:**

1. User holds Shift key
1. User clicks on a grid cell containing an entity
1. Entity is immediately removed from the grid
1. Grid cell returns to empty state
1. Level state is updated
1. Action is added to undo history

**Exit:** Entity is removed, grid cell is empty.

______________________________________________________________________

### Flow 5: Edit Level - Undo/Redo

**Description:** User undoes or redoes editing actions.

**Trigger:** User clicks undo/redo buttons or uses keyboard shortcuts.

**Steps:**

**Undo:**

1. User clicks "Undo" button or presses Ctrl+Z (Cmd+Z on Mac)
1. Last action is reversed (entity placement or removal)
1. Grid updates to reflect previous state
1. Action moves to redo history

**Redo:**

1. User clicks "Redo" button or presses Ctrl+Y (Cmd+Shift+Z on Mac)
1. Previously undone action is reapplied
1. Grid updates to reflect redone state
1. Action moves back to undo history

**Exit:** Level state is updated to previous/next state.

______________________________________________________________________

### Flow 6: Configure Snake Direction

**Description:** User sets the initial direction the snake faces.

**Trigger:** User wants to change snake direction from default (East).

**Steps:**

1. User locates the "Direction" dropdown in the toolbar
1. User clicks the dropdown
1. Options appear: North, South, East, West
1. User selects desired direction
1. Snake direction is updated in the level data
1. Visual feedback may show direction indicator on snake (optional)

**Exit:** Snake direction is set for the level.

______________________________________________________________________

### Flow 7: Test Level

**Description:** User tests the level by playing it in gsnake-web.

**Trigger:** User clicks "Test Level" button in toolbar.

**Steps:**

1. User clicks "Test Level" button
1. Editor generates level JSON from current state
1. Browser downloads the JSON file (e.g., `test-level.json`)
1. User manually opens gsnake-web in a new tab/window
1. In gsnake-web, user uses the file upload feature to load the test level
1. User plays the level to test gameplay
1. User identifies issues or confirms level works as intended
1. User closes gsnake-web tab and returns to editor tab
1. User makes adjustments based on testing feedback
1. User can repeat the test cycle as needed

**Exit:** User returns to editor to continue editing or save the level.

______________________________________________________________________

### Flow 8: Save/Export Level

**Description:** User exports the level as a JSON file with metadata.

**Trigger:** User clicks "Save Level" button in toolbar.

**Steps:**

1. User clicks "Save Level" button
1. Save modal appears prompting for metadata
1. User enters level name (e.g., "First Steps")
1. User selects difficulty from dropdown (easy/medium/hard)
1. Modal shows info: "Level ID will be auto-generated"
1. User clicks "Export JSON" button
1. Editor generates level JSON with:

- Auto-generated unique ID
- User-provided name and difficulty
- All entity positions and grid configuration
- Snake direction

8. Browser downloads the JSON file (e.g., `level-001.json`)
1. Modal closes, user returns to editor
1. User can manually place the file in the appropriate directory

**Exit:** Level JSON is downloaded, user can continue editing or create a new level.

______________________________________________________________________

### Flow 9: Start New Level from Editor

**Description:** User starts a new level while already in the editor.

**Trigger:** User clicks "New Level" button in toolbar.

**Steps:**

1. User clicks "New Level" button in toolbar
1. Editor returns to landing page
1. User can choose "Create New Level" or "Load Existing Level"
1. Flow continues as per Flow 1 or Flow 2

**Exit:** User is back at landing page or in a new editor session.

______________________________________________________________________

## Interaction Patterns Summary

| Action | Interaction | Visual Feedback |
| ----------------------- | ------------------------------- | ----------------------------------- |
| Select entity | Click entity in palette | Highlight with border/background |
| Place entity (single) | Drag from palette, drop on grid | Entity appears on cell |
| Place entity (multiple) | Click entity, then click cells | Entity appears on each clicked cell |
| Place snake | Click cells in sequence | Connected segments appear |
| Remove entity | Shift + click on cell | Entity disappears |
| Undo | Ctrl+Z or button | Grid reverts to previous state |
| Redo | Ctrl+Y or button | Grid advances to next state |
| Change direction | Select from dropdown | Dropdown updates |
| Test level | Click button | JSON downloads |
| Save level | Click button, fill modal | JSON downloads with metadata |

## Key UX Principles

1. **Immediate Visual Feedback:** All entity placements show instantly with colored cells/icons matching game appearance
1. **Flexible Interaction:** Support both drag-and-drop (quick single placements) and click-to-select (efficient multiple placements)
1. **Forgiving Editing:** Undo/redo support allows experimentation without fear of mistakes
1. **Minimal Friction:** No validation warnings during editing; users can create any configuration
1. **Clear State Communication:** Selected entity always highlighted in palette
1. **Seamless Testing:** Simple download-and-upload workflow for testing in gsnake-web
