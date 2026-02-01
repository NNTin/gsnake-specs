## Summary

This Epic introduces a visual level editor for gSnake, enabling creators to design levels through an intuitive drag-and-drop interface rather than manual JSON editing. The editor will be a standalone web application that allows users to place game entities (snake, obstacles, food, exit, stones, spikes, etc.) on a configurable grid, test levels directly in gsnake-web, and export them as JSON files. By providing immediate visual feedback and a seamless edit-test cycle, the editor will dramatically reduce the time from concept to playable level while lowering the barrier to entry for level creation. This tool serves both the core development team and potential future community contributors, enabling faster iteration, higher quality designs, and ultimately more content for the game.

## Context & Problem

**Who's Affected:**

- **Core development team**: Currently creating official levels through manual JSON editing
- **Potential community contributors**: Future creators who want to contribute custom levels but face technical barriers
- **Level designers**: Anyone designing game content who needs rapid iteration and visual feedback

**Current Pain Points:**

The existing level creation workflow requires manually editing JSON files in the file:gsnake-levels/levels/ directory. This process is:

1. **Error-prone**: No validation during editing; typos and structural errors only surface when testing
1. **Tedious**: Placing entities requires calculating coordinates manually and typing them into JSON arrays
1. **Lacks visual feedback**: Impossible to visualize the level layout without running the game
1. **Slow iteration**: The edit-save-test cycle is cumbersome, requiring file saves and game restarts
1. **High barrier to entry**: Requires understanding JSON structure and coordinate systems, limiting who can create levels

**Impact:**

This friction slows down level creation, reduces the quality of level designs (due to limited iteration), and prevents non-technical contributors from participating in content creation. As the game grows, the need for more diverse and high-quality levels increases, making this manual process a bottleneck.

**Desired Outcome:**

A visual level editor that enables:

- **Faster creation**: Reduce time from idea to playable level by 10x through visual editing
- **Better quality**: Enable rapid iteration and experimentation with immediate visual feedback
- **More content**: Lower the barrier to entry, allowing more people to create levels
- **Seamless testing**: Integrated workflow to test levels in gsnake-web without manual file management
