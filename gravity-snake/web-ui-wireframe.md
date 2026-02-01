## UI Layout

```wireframe
<!DOCTYPE html>
<html>
<head>
<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
body {
  font-family: Arial, sans-serif;
  background: #f5f5f5;
  padding: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
}
.game-container {
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  max-width: 600px;
  width: 100%;
}
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding-bottom: 15px;
  border-bottom: 2px solid #e0e0e0;
}
.score-info {
  display: flex;
  gap: 20px;
  font-size: 14px;
}
.score-item {
  display: flex;
  flex-direction: column;
}
.score-label {
  color: #666;
  font-size: 11px;
  text-transform: uppercase;
  margin-bottom: 2px;
}
.score-value {
  font-weight: bold;
  font-size: 18px;
  color: #333;
}
.restart-btn {
  padding: 8px 16px;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
}
.game-field {
  width: 100%;
  aspect-ratio: 1;
  background: #fafafa;
  border: 2px solid #ddd;
  display: grid;
  grid-template-columns: repeat(15, 1fr);
  grid-template-rows: repeat(15, 1fr);
  gap: 1px;
  background: #ddd;
}
.cell {
  background: white;
}
.cell.snake-head {
  background: #2196F3;
}
.cell.snake-body {
  background: #64B5F6;
}
.cell.food {
  background: #FF5722;
}
.cell.obstacle {
  background: #424242;
}
.cell.exit {
  background: #4CAF50;
}
.overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.7);
  display: none;
  justify-content: center;
  align-items: center;
}
.overlay.active {
  display: flex;
}
.modal {
  background: white;
  padding: 30px;
  border-radius: 8px;
  text-align: center;
  min-width: 300px;
}
.modal h2 {
  margin-bottom: 20px;
  color: #333;
}
.modal-buttons {
  display: flex;
  gap: 10px;
  justify-content: center;
}
.modal-btn {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
}
.modal-btn.primary {
  background: #4CAF50;
  color: white;
}
.modal-btn.secondary {
  background: #e0e0e0;
  color: #333;
}
</style>
</head>
<body>
<div class="game-container">
  <div class="header">
    <div class="score-info">
      <div class="score-item">
        <span class="score-label">Level</span>
        <span class="score-value" data-element-id="level-display">1</span>
      </div>
      <div class="score-item">
        <span class="score-label">Length</span>
        <span class="score-value" data-element-id="length-display">3</span>
      </div>
      <div class="score-item">
        <span class="score-label">Moves</span>
        <span class="score-value" data-element-id="moves-display">0</span>
      </div>
    </div>
    <button class="restart-btn" data-element-id="restart-button">Restart</button>
  </div>
  
  <div class="game-field" data-element-id="game-field">
    <!-- 15x15 grid cells -->
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell snake-head"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <!-- More cells... (showing sample layout) -->
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell snake-body"></div>
    <div class="cell snake-body"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell food"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <!-- Additional rows with obstacles and exit -->
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell obstacle"></div>
    <div class="cell obstacle"></div>
    <div class="cell obstacle"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell"></div>
    <div class="cell exit"></div>
    <div class="cell"></div>
  </div>
</div>

<!-- Game Over Overlay -->
<div class="overlay" data-element-id="game-over-overlay">
  <div class="modal">
    <h2>Game Over</h2>
    <div class="modal-buttons">
      <button class="modal-btn primary" data-element-id="restart-level-btn">Restart Level</button>
      <button class="modal-btn secondary" data-element-id="back-to-level1-btn">Back to Level 1</button>
    </div>
  </div>
</div>

<!-- Game Complete Overlay -->
<div class="overlay" data-element-id="game-complete-overlay">
  <div class="modal">
    <h2>All Levels Complete!</h2>
    <p style="margin: 15px 0; color: #666;">Congratulations! You've beaten all levels.</p>
    <p style="font-size: 12px; color: #999;">Refresh the page to play again.</p>
  </div>
</div>
</body>
</html>
```
