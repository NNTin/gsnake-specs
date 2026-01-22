## Summary

The gSnake project currently manages all game levels in a single monolithic `levels.json` file embedded in gsnake-core, with no structured workflow for creating, testing, verifying, or documenting individual levels. This Epic introduces a dedicated level management system through a new `gsnake-levels` git submodule that provides level organization by difficulty, automated verification of solvability, visual documentation via SVG renders, and CLI-driven workflows for level designers. The system extends gsnake-cli to support individual level selection and asciinema recording, extends gsnake-core to support external level loading while maintaining backward compatibility, and extends the contract interface to enable gsnake-web to load custom levels and select specific levels to play.

## Context & Problem

### Who's Affected

**Level Designers & Contributors**

- Currently have no structured way to create, test, and document individual levels
- Cannot verify that levels are solvable before committing changes
- Have no visual documentation to showcase level designs
- Must manually edit the monolithic `levels.json` file, risking merge conflicts

**Players & Users**

- Cannot practice or replay specific levels in CLI mode
- Cannot load custom level sets in the web interface
- Have no way to see visual previews of levels before playing

**CI/CD & Quality Assurance**

- No automated verification that levels remain solvable after code changes
- No way to detect if level changes break existing solutions
- Cannot generate documentation artifacts automatically

### Current Pain Points

1. **Monolithic Level Storage**: All 12 levels are stored in file:gsnake-core/engine/core/data/levels.json with no organization by difficulty or metadata tracking
2. **No Level Verification**: There's no automated way to verify that levels are solvable with provided solutions, leading to potential broken levels in production
3. **Limited CLI Capabilities**: The CLI in file:gsnake-core/engine/bindings/cli/src/main.rs loads all levels at once and cycles through them sequentially, with no way to select or test individual levels
4. **No Visual Documentation**: Levels exist only as JSON data with no visual representations for documentation or preview purposes
5. **No Level Metadata**: No tracking of level authorship, difficulty, tags, or solvability status
6. **Web Interface Limitations**: The contract interface (file:e2e/contract.spec.ts) supports level selection via `?level=2` query parameter but cannot load custom level sets

### What Success Looks Like

- Level designers can create individual level files organized by difficulty (easy/medium/hard)
- Automated verification ensures all levels with playback solutions are solvable
- Visual SVG renders provide documentation for each level
- CLI supports selecting and playing individual levels via `--level-file` or `--level-id`
- CLI can record gameplay sessions with built-in asciinema integration
- Web interface can load custom levels.json files and select specific levels
- CI pipeline automatically verifies level integrity and generates documentation
- The aggregated `levels.json` can be generated from individual level files with difficulty filtering
- All changes maintain backward compatibility with existing gsnake-web functionality

