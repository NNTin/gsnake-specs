# PRD: Automated Level Metadata Synchronization and Validation

## Introduction

Automate the generation and validation of level metadata (levels.toml files) for gSnake levels, ensuring consistency between level JSON files and their registry metadata. This system will migrate existing levels from string IDs to numeric IDs, generate creative level names based on level analysis, create playback solutions, and validate metadata integrity in CI.

The new level files added to gsnake-levels use string-based IDs (e.g., "1769977122223-g36bwe") which are incompatible with the gsnake-core engine that expects u32 IDs. Additionally, the levels.toml files are outdated and reference old level files that no longer exist.

## Goals

- Migrate all level JSON files from string IDs to numeric timestamp-based IDs
- Auto-generate levels.toml files from level JSON files in each difficulty folder
- Generate creative, descriptive level names by analyzing level layout and mechanics
- Create playback solution files for all levels
- Validate levels.toml integrity in CI pipeline
- Integrate metadata sync into the existing generate-levels-json workflow
- Maintain backward compatibility with existing gsnake-core engine (no engine changes)

## User Stories

### US-001: Migrate level IDs from string to numeric format

**Description:** As a developer, I need to convert string-based level IDs to numeric u32 IDs so that levels can be parsed by gsnake-core engine.

**Acceptance Criteria:**

- [ ] Create a migration tool that reads level JSON files with string IDs
- [ ] Extract timestamp portion from string IDs (e.g., "1769977122223-g36bwe" → 1769977122223)
- [ ] Verify extracted timestamp fits within u32 range (0 to 4,294,967,295)
- [ ] Update all level JSON files in levels/{easy,medium,hard}/ directories
- [ ] Preserve all other level fields (name, difficulty, gridSize, snake, obstacles, etc.)
- [ ] Validate that migrated levels parse successfully with gsnake-core's LevelDefinition
- [ ] Typecheck passes

### US-002: Generate creative level names based on analysis

**Description:** As a level designer, I want automatically generated descriptive level names so that levels are identifiable and engaging without manual naming.

**Acceptance Criteria:**

- [ ] Analyze level JSON structure to identify key mechanics:
  - Presence of floating food, falling food, stones, spikes
  - Obstacle patterns (walls, mazes, towers)
  - Grid complexity and layout
  - Food-to-exit distance
- [ ] Generate creative names based on analysis (e.g., "Floating Islands" for levels with floatingFood, "Spike Gauntlet" for spike-heavy levels)
- [ ] Ensure names are unique across all difficulty levels
- [ ] Replace "todoname" placeholders in existing level JSON files
- [ ] Names should be 1-4 words, gameplay-descriptive
- [ ] Typecheck passes

### US-003: Auto-generate levels.toml from level JSON files

**Description:** As a developer, I need levels.toml files to automatically sync with level JSON files so that metadata is always up-to-date.

**Acceptance Criteria:**

- [ ] Scan levels/{easy,medium,hard}/ directories for all .json files
- [ ] Generate levels.toml entries with structure:
  ```toml
  [[level]]
  id = "level-{timestamp}-{suffix}"
  file = "level-{timestamp}-{suffix}.json"
  author = "gsnake"
  solved = true
  difficulty = "{easy|medium|hard}"
  tags = []
  description = "{generated-name}"
  ```
- [ ] Fail with clear error if referenced JSON file doesn't exist
- [ ] Preserve existing manual metadata (author, tags) if already present
- [ ] Write levels.toml to levels/{difficulty}/levels.toml
- [ ] Typecheck passes

### US-004: Generate playback solutions for all levels

**Description:** As a developer, I need playback JSON files for all levels so that levels can be verified and replayed.

**Acceptance Criteria:**

- [ ] Run solve_level binary for each level JSON file
- [ ] Increase max depth to 500 (from current 200) to handle complex levels
- [ ] Generate playback JSON in playbacks/{difficulty}/ directory
- [ ] Playback filename matches level filename (e.g., level-1769977122223-g36bwe.json → playbacks/easy/level-1769977122223-g36bwe.json)
- [ ] Mark level as solved=true in levels.toml if playback generated successfully
- [ ] If solve_level fails, mark solved=false and report level ID
- [ ] Typecheck passes

### US-005: Add CI validation for levels.toml integrity

**Description:** As a developer, I want CI to validate levels.toml files so that invalid metadata is caught before merge.

**Acceptance Criteria:**

- [ ] Add new CI job "validate-levels" to gsnake-levels/.github/workflows/ci.yml
- [ ] Job validates for each difficulty (easy, medium, hard):
  - levels.toml exists and is valid TOML format
  - All referenced JSON files exist
  - All JSON files parse as valid LevelDefinition
- [ ] Fail CI immediately on first validation error with clear error message
- [ ] Run validation job on push and pull_request events
- [ ] Typecheck passes

### US-006: Integrate sync into generate-levels-json command

**Description:** As a developer, I want metadata sync to run automatically when generating levels.json so that the output always uses up-to-date metadata.

**Acceptance Criteria:**

- [ ] Add `--sync` flag to generate-levels-json command
- [ ] When `--sync` is passed, run metadata sync before aggregation:
  1. Migrate IDs (if needed)
  1. Generate level names (if needed)
  1. Update levels.toml files
  1. Generate missing playbacks
- [ ] Run sync automatically by default (can be disabled with `--no-sync` flag)
- [ ] Sync process logs progress (migrating IDs, generating names, updating toml, generating playbacks)
- [ ] If sync fails, abort generate-levels-json with error
- [ ] Typecheck passes

### US-007: Add explicit sync-metadata command

**Description:** As a developer, I want to run metadata sync independently so that I can update metadata without generating levels.json.

**Acceptance Criteria:**

- [ ] Add new subcommand: `cargo run -- sync-metadata`
- [ ] Command accepts optional `--difficulty <easy|medium|hard>` to sync specific difficulty
- [ ] Default syncs all difficulties
- [ ] Performs same operations as US-006 sync step
- [ ] Reports summary of changes (e.g., "Migrated 9 IDs, generated 9 names, updated 3 levels.toml files, created 9 playbacks")
- [ ] Exit code 0 on success, non-zero on any failure
- [ ] Typecheck passes

### US-008: Optimize level solver (future enhancement)

**Description:** As a developer, I want to investigate solver optimization so that complex levels can be solved faster.

**Acceptance Criteria:**

- [ ] Document current solve_level performance with max_depth=500
- [ ] Analyze provided playback solutions to understand solution characteristics
- [ ] Identify optimization opportunities:
  - Better heuristics for BFS search
  - A\* pathfinding with appropriate heuristic
  - Pruning strategies for state space reduction
  - Parallel solving for multiple levels
- [ ] Benchmark improvements against current implementation
- [ ] Document findings and recommended approach
- [ ] Typecheck passes

**Note:** This story is marked as future work and can be prioritized separately.

## Functional Requirements

- FR-1: The system must parse level JSON files with either string or numeric IDs
- FR-2: The system must extract timestamp from string IDs (format: "timestamp-suffix")
- FR-3: The system must validate that extracted timestamps fit within u32 range
- FR-4: The system must analyze level structure to identify: obstacles, food types, stones, spikes, grid size
- FR-5: The system must generate unique, descriptive names based on level mechanics
- FR-6: The system must scan levels/{difficulty}/ directories for .json files
- FR-7: The system must generate levels.toml with all required fields (id, file, author, solved, difficulty, tags, description)
- FR-8: The system must fail immediately if a levels.toml entry references a non-existent JSON file
- FR-9: The system must run solve_level binary with max_depth=500 for each level
- FR-10: The system must save playback JSON to playbacks/{difficulty}/ with matching filenames
- FR-11: The system must update solved status in levels.toml based on solve_level success
- FR-12: CI must validate levels.toml format and referenced files exist
- FR-13: CI must validate all level JSON files parse as valid LevelDefinition
- FR-14: CI must fail immediately on first validation error
- FR-15: The generate-levels-json command must run metadata sync by default
- FR-16: The generate-levels-json command must accept --no-sync flag to skip sync
- FR-17: The sync-metadata command must work independently of generate-levels-json

## Non-Goals

- No changes to gsnake-core engine or LevelDefinition struct
- No automatic level difficulty classification
- No level balancing or gameplay tuning
- No UI for level management
- No migration of levels.json in gsnake-core (will be updated via generate-levels-json)
- No multiplayer or online level sharing features

## Design Considerations

### Level Name Generation Strategy

Analyze level characteristics in priority order:

1. **Special mechanics** (highest priority):

   - FloatingFood → "Floating [descriptor]"
   - FallingFood → "Cascade" or "Falling [descriptor]"
   - Stones → "Push" or "Sokoban" prefix
   - Spikes → "Spike" prefix

1. **Obstacle patterns**:

   - Vertical walls → "Tower", "Column"
   - Horizontal walls → "Bridge", "Corridor"
   - Zigzag patterns → "Zigzag", "Maze"
   - Scattered → "Islands", "Platforms"

1. **Complexity indicators**:

   - Food count and distribution
   - Snake starting length
   - Grid utilization (obstacles / total cells)

Example names:

- Level with floating food + obstacles → "Floating Islands"
- Level with spikes + tight corridors → "Spike Gauntlet"
- Level with stones + precision required → "Precision Push"
- Level with falling food + platforms → "Cascade Drop"

### Directory Structure

```
gsnake-levels/
├── levels/
│   ├── easy/
│   │   ├── level-1769977122223-g36bwe.json
│   │   ├── level-1769978263873-eupaj5.json
│   │   └── levels.toml
│   ├── medium/
│   │   ├── level-1769975926647-nv3aqb.json
│   │   ├── level-1769976243963-g777pk.json
│   │   ├── level-1769976545160-dvt1ot.json
│   │   └── levels.toml
│   └── hard/
│       ├── level-1769977459516-uoh6te.json
│       ├── level-1769977886852-17yfz6.json
│       ├── level-1769978661806-v473pq.json
│       └── levels.toml
└── playbacks/
    ├── easy/
    │   ├── level-1769977122223-g36bwe.json
    │   └── level-1769978263873-eupaj5.json
    ├── medium/
    │   ├── level-1769975926647-nv3aqb.json
    │   ├── level-1769976243963-g777pk.json
    │   └── level-1769976545160-dvt1ot.json
    └── hard/
        ├── level-1769977459516-uoh6te.json
        ├── level-1769977886852-17yfz6.json
        └── level-1769978661806-v473pq.json
```

## Technical Considerations

### ID Migration

- Current format: `"id": "1769977122223-g36bwe"` (string)
- Target format: `"id": 1769977122223` (u32)
- Timestamp range: 0 to 4,294,967,295 (max u32)
- Current timestamps (~1769977122223) fit within u32 range

### Solver Performance

- Current max_depth: 200 moves
- New max_depth: 500 moves
- BFS explores all possible moves per state
- State includes: snake position, direction, food, floating_food, falling_food, stones, spikes, exit_is_solid
- Complex levels may have large state spaces
- Consider future optimization if 500 is insufficient

### CI Pipeline Integration

```yaml
validate-levels:
  name: Validate Levels
  runs-on: ubuntu-latest
  steps:
    - name: Checkout submodule
      uses: actions/checkout@v4
    - name: Setup Rust
      uses: dtolnay/rust-toolchain@stable
    - name: Run validation
      run: cargo run -- validate-levels-toml
      working-directory: ${{ env.LEVELS_DIR }}
```

### Error Handling

- Fail fast on any validation error
- Clear error messages with file paths and line numbers
- Exit codes:
  - 0: Success
  - 1: Validation failure
  - 2: IO error
  - 3: Parse error

## Success Metrics

- All 9 new level files successfully migrated to numeric IDs
- All levels parse without errors using gsnake-core LevelDefinition
- All levels have creative, unique names
- All levels have generated playback solutions
- CI validation passes on all branches
- `cargo run -- generate-levels-json` successfully updates gsnake-core/engine/core/data/levels.json
- Zero manual intervention required for metadata management
- Level addition workflow: add JSON → run sync-metadata → commit

## Open Questions

- Should we add a dry-run mode for sync-metadata to preview changes?
- Should we validate that playback solutions actually complete the level (not just that solve_level succeeded)?
- Should we add a maximum solve time limit (e.g., 60 seconds) to prevent hanging on unsolvable levels?
- Should we track level statistics (move count, solution length) in levels.toml?
- Should the solver use parallel processing for multiple levels?

## Implementation Order

Recommended implementation sequence:

1. **US-001** (ID migration) - Unblocks all other work
1. **US-002** (Name generation) - Can run in parallel with US-003
1. **US-003** (levels.toml generation) - Depends on US-001, US-002
1. **US-004** (Playback generation) - Depends on US-001
1. **US-007** (sync-metadata command) - Integrates US-001 through US-004
1. **US-006** (integrate with generate-levels-json) - Depends on US-007
1. **US-005** (CI validation) - Can be done anytime after US-003
1. **US-008** (Solver optimization) - Future work, not blocking

## Dependencies

- gsnake-core engine (no changes allowed, must remain compatible)
- Rust toolchain (stable)
- Existing solve_level binary
- Existing generate-levels-json command
- GitHub Actions CI pipeline

## Risk Mitigation

**Risk:** Solver fails to find solutions for complex levels within max_depth=500

- **Mitigation:** User has existing solutions that can be manually provided; implement US-008 for optimization

**Risk:** ID migration breaks existing references

- **Mitigation:** Validate all references after migration; maintain backup of original files

**Risk:** Generated names are not descriptive enough

- **Mitigation:** Implement name analysis heuristics; allow manual override in levels.toml

**Risk:** CI validation adds significant build time

- **Mitigation:** Only validate changed files; cache validation results; run in parallel

## Timeline Estimate

- US-001: High priority (blocks other work)
- US-002: Medium priority
- US-003: High priority
- US-004: High priority
- US-005: Medium priority
- US-006: Medium priority
- US-007: High priority
- US-008: Low priority (future work)

Total estimated complexity: Medium to Large (involves multiple systems and careful data migration)
