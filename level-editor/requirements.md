Level Editor Requirements Summary
Architecture & Platform
Separate web application in new gsnake-editor/ directory with its own build
Technology stack: Vite + Svelte (consistent with gsnake-web)
Backend service: Simple HTTP server for temporary level storage (enables testing)
Client-side focused: Purely browser-based editing with minimal backend dependency
Core Editing Features
Grid Configuration: Configurable width and height (not fixed to 15x15)
Entity Palette: Support all entity types
Snake (initial position and segments)
Obstacles (walls/barriers)
Food (collectible items)
Exit (level completion point)
Stones (pushable objects with gravity)
Spikes (hazards)
FloatingFood (food that doesn't fall)
FallingFood (food affected by gravity)
Interaction Model: Drag-and-drop from palette to grid
Initial State: Empty grid (user builds from scratch)
File Operations
Load Levels: Upload JSON file via standard file input (client-side)
Save Levels: Download JSON file (user manually places in levels/ directory)
Metadata Handling:
Auto-generate level ID
Prompt for name and difficulty (easy/medium/hard) on save
No Validation: Editor allows any configuration; validation happens separately via existing gsnake-levels CLI
Testing Integration
Test Button: Saves level to backend temporary storage
Launch gsnake-web: Opens in new browser tab/window
Level Loading: gsnake-web loads the test level from temporary storage
Seamless Workflow: Edit → Test → Refine cycle
Key Design Decisions
No built-in validation (keeps editor simple, leverages existing CLI tools)
Manual file placement (avoids complex filesystem permissions)
Backend only for testing (minimal infrastructure)
Separate application (clean separation of concerns)