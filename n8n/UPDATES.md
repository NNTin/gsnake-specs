# Documentation Updates - 2026-02-07

## Files Updated

### ✅ gsnake-n8n/CLAUDE.md

**Changes:**

- Updated Layer 2 example to reference `tools/scripts/sync-workflows.sh` instead of deprecated docker volume sync
- Enhanced Layer 3 description to include both n8n workflows and shell scripts
- Fixed typo: "transofmrations" → "transformations"
- Expanded file structure with detailed directory layout
- Added comprehensive sync-workflows.sh documentation
- Added new section: "SOP to Tools Mapping" with status table
- Added new section: "n8n Workflow Management" with technical details
- Updated core principles with sync strategy and ID preservation info

**Key Additions:**

- Sync script usage examples (import/export/sync)
- Workflow ID preservation mechanics
- File format specifications
- Credential management approach
- Testing guidelines
- Clear deprecation notices for old approaches

### ✅ gsnake-n8n/workflows/actions/node-docker-volume-sync.md

**Changes:**

- Added prominent deprecation warning at top
- Marked as historical context
- Points to replacement (sync-workflows.sh)
- References architecture-decision.md for explanation

### ✅ gsnake-n8n/workflows/service/docker-volume-sync.md

**Changes:**

- Added prominent deprecation warning at top
- Marked as historical context
- Points to replacement (sync-workflows.sh)
- References architecture-decision.md for explanation

### ✅ Created Tools

- `gsnake-n8n/tools/scripts/sync-workflows.sh` - Working sync script (tested export)
- Exported 5 existing workflows to `gsnake-n8n/tools/n8n-flows/`

### ✅ Created Documentation (gsnake-specs/n8n/)

- `architecture-analysis.md` - Detailed issue analysis
- `test-findings.md` - Test results and verified behaviors
- `architecture-decision.md` - Revised architecture and decisions

______________________________________________________________________

## SOP to Tools Mapping Status

| SOP File | Implementation | Status | Notes |
|----------|---------------|--------|-------|
| `workflows/service/notify-discord.md` | Not implemented | ⏳ TODO | First POC workflow candidate |
| `workflows/service/docker-volume-sync.md` | Deprecated | ⚠️ Historical | Replaced by sync-workflows.sh |
| `workflows/actions/node-docker-volume-sync.md` | Deprecated | ⚠️ Historical | Replaced by sync-workflows.sh |

______________________________________________________________________

## What Changed Architecturally

### OLD Approach (Deprecated)

```
User → Claude → n8n workflow "volume-sync" → Docker volume mount → n8n data dir
                      ↑______________|
                  (circular dependency)
```

**Problems:**

- Circular dependency (workflow deploys workflows)
- Assumed n8n auto-loads from volume (unreliable)
- Complex setup with webhook triggers
- Bootstrap problem (how to deploy the sync workflow itself?)

### NEW Approach (Implemented)

```
User → Claude → sync-workflows.sh → n8n CLI → n8n instance
          ↓
    Update Git
```

**Benefits:**

- No circular dependency
- Uses official n8n CLI (reliable)
- Simple shell script (easy to debug)
- Git is source of truth
- ID preservation works perfectly

______________________________________________________________________

## Key Technical Learnings

### ✅ Verified Facts

1. **n8n CLI is fully functional**

   - Export: `n8n export:workflow --backup --output=/dir/` creates one file per workflow
   - Import: `n8n import:workflow --separate --input=/dir/` imports all files
   - Files named by workflow ID (e.g., `test-id-12345.json`)

1. **ID preservation works perfectly**

   - IDs from JSON are preserved on import (no regeneration)
   - Reimporting with same ID updates in place (idempotent)
   - No need for two-step "import then export" flow

1. **File transfer via docker cp works**

   - Tested bidirectionally
   - No docker-compose changes needed
   - Safe and explicit

1. **Sync script is functional**

   - Export tested successfully (5 workflows)
   - Import expected to work (symmetric design)
   - Cleanup handles temp directories properly

### 📋 Implementation Details

**Workflow File Format:**

```json
{
  "id": "workflow-id-here",           // Preserved on import
  "name": "Workflow Name",
  "versionId": "uuid-generated-by-n8n", // Changes on each update
  "nodes": [...],
  "connections": {...},
  "shared": [{                        // Project assignment
    "projectId": "...",
    "project": {...}
  }]
}
```

**Sync Flow:**

1. Export: n8n → container temp dir → host git repo
1. Import: host git repo → container temp dir → n8n
1. Sync: export (backup) then import (restore)

______________________________________________________________________

## Next Steps

### Immediate Tasks

1. ⏳ Test import functionality with sync script
1. ⏳ Clean up test workflow from n8n UI
1. ⏳ Answer remaining open questions:
   - Q: What MCP server tools are available?
   - Q: How do credential references work?
   - Q: Project assignment behavior?

### Documentation Tasks

4. ⏳ Rewrite `gsnake-specs/n8n/plan.md` with corrected architecture
1. ⏳ Rewrite `gsnake-specs/n8n/flows.md` to remove workflow-sync bootstrap
1. ⏳ Update `gsnake-specs/n8n/brief.md` if needed

### POC Implementation

7. ⏳ Create first real workflow (notify-discord)

   - Write SOP in `workflows/service/notify-discord.md`
   - Generate/author JSON
   - Deploy via sync script
   - Test via MCP or UI
   - Commit to git

1. ⏳ Document credential workflow

1. ⏳ Test end-to-end generation flow

______________________________________________________________________

## Files Needing Updates (Not Yet Done)

### gsnake-specs/n8n/

- [ ] `plan.md` - Complete rewrite with new architecture
- [ ] `flows.md` - Remove workflow-sync workflow, update flows
- [ ] `brief.md` - Update if success criteria changed

### gsnake-n8n/workflows/

- [ ] `service/notify-discord.md` - Needs implementation
- [ ] Create more SOPs as needed

### gsnake-n8n/tools/n8n-flows/

- [ ] Clean up test workflow (test-id-12345.json)
- [ ] Add real workflows as they're created

______________________________________________________________________

## Success Metrics Progress

1. ✅ Sync script successfully exports workflows
1. ⏳ Sync script successfully imports workflows (assumed working, not yet tested)
1. ⏳ Generate new workflow from markdown SOP
1. ⏳ Workflow deploys to n8n via sync script
1. ⏳ Workflow executes successfully via MCP
1. ⏳ Changes committed to git with canonical JSON
1. ⏳ Documentation accurately reflects working system

**Current Status: 1/7 complete** (export verified)

______________________________________________________________________

## Command Reference

### Sync Script

```bash
# Export workflows from n8n to git
cd /home/nntin/git/gSnake/gsnake-n8n
./tools/scripts/sync-workflows.sh export

# Import workflows from git to n8n
./tools/scripts/sync-workflows.sh import

# Bidirectional sync
./tools/scripts/sync-workflows.sh sync
```

### n8n CLI (inside container)

```bash
CONTAINER_ID="440742681e58b8049db5f7541c5ce24312bd348662e6a68bab55720f7d16d30e"

# Export all workflows
docker exec $CONTAINER_ID n8n export:workflow --backup --output=/tmp/flows/

# Import workflows from directory
docker exec $CONTAINER_ID n8n import:workflow --separate --input=/tmp/flows/

# Get help
docker exec $CONTAINER_ID n8n export:workflow --help
docker exec $CONTAINER_ID n8n import:workflow --help
```

### Browser Access (for manual testing)

```bash
# Open n8n UI
agent-browser open https://n8n.labs.lair.nntin.xyz/

# Credentials in .env file
cat /home/nntin/git/gSnake/gsnake-n8n/.env
```

______________________________________________________________________

## Questions for User

None currently - documentation updates are complete. Ready for next phase:

1. Test import functionality
1. Rewrite gsnake-specs/n8n/\*.md files
1. Create first POC workflow

User can proceed with any of these next steps.
