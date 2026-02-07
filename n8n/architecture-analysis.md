# Architecture Analysis & Issues

## Verified Capabilities

### ✅ n8n CLI Access

- **Location**: `/usr/local/bin/n8n` in container `440742681e58b8049db5f7541c5ce24312bd348662e6a68bab55720f7d16d30e`
- **Version**: 2.4.6
- **Export Command**: Working (tested with `--all --pretty`)
- **Import Command**: Available (not yet tested)
- **Access Method**: `docker exec` from host

### ✅ MCP Server

- **Endpoint**: `https://n8n.labs.lair.nntin.xyz/mcp-server/http`
- **Authentication**: Bearer token configured in `.mcp.json`
- **Purpose**: Read-only access (search, execute, get details)
- **Limitation**: Cannot write workflows directly

### ✅ Repository Structure

```
gsnake-n8n/
├── workflows/
│   ├── service/          # Webhook endpoint specs
│   └── actions/          # Custom node specs
├── tools/
│   └── n8n-flows/        # Generated workflow JSON
└── .mcp.json             # MCP configuration
```

______________________________________________________________________

## Architectural Issues with Current plan.md

### 🚨 Issue 1: Circular Dependency - Workflow Deploys Itself

**Problem**: Plan suggests creating an internal "workflow-sync" n8n workflow that runs n8n CLI to deploy workflows.

**Why This Is Broken**:

1. **Bootstrap Problem**: How do you deploy the sync workflow itself? It can't deploy itself.
1. **Fragility**: If the sync workflow breaks, you can't fix it using the sync mechanism.
1. **Complexity**: Running n8n CLI from within n8n workflow via Execute Command node adds unnecessary indirection.
1. **Permission Issues**: Execute Command node may not have proper permissions.

**Current Plan States** (plan.md:267-278):

> "Internal 'workflow-sync' Workflow (Deployment Mechanism)"
>
> - Import workflow JSON into n8n using `n8n import:workflow`
> - Export workflow JSON from n8n using `n8n export:workflow`
> - Trigger: Internal execution via MCP `execute_workflow`

**The Flaw**: You need a workflow to deploy workflows, but the deployer itself is a workflow that needs deployment.

______________________________________________________________________

### 🚨 Issue 2: Confused Separation of Concerns

**Problem**: Mixing deployment infrastructure (sync workflow) with business logic (actual workflows).

**Why This Is Wrong**:

- n8n workflows should automate business processes, not manage themselves
- Deployment is infrastructure concern, not workflow concern
- Creates tight coupling between workflow runtime and deployment pipeline

**Better Approach**: Deployment should happen OUTSIDE of n8n using host-level tools.

______________________________________________________________________

### 🚨 Issue 3: Unclear File Transfer Mechanism

**Problem**: Plan doesn't specify how JSON files get from host filesystem into container for import.

**Missing Details**:

- How does `n8n import:workflow --input=/path/to/file.json` access files?
- Files are in `gsnake-n8n/tools/n8n-flows/` on host
- Import command runs inside container
- No volume mount for `tools/n8n-flows/` in current docker-compose

**Options to Consider**:

1. Use `docker cp` to copy files into container before import
1. Mount `tools/n8n-flows/` as volume in docker-compose
1. Use `docker exec` with stdin: `cat file.json | docker exec -i <container> n8n import:workflow --input=-`

______________________________________________________________________

### 🚨 Issue 4: Workflow ID Management Unclear

**Problem**: Plan mentions preserving workflow IDs but doesn't specify the exact mechanism.

**Current Plan States** (plan.md:153-162):

> "First creation: import the generated JSON (ID may be assigned/normalized by n8n)"
> "Immediately export the workflow from n8n and commit that exported JSON as the *canonical* JSON"

**Questions**:

1. Does n8n CLI `import` preserve IDs from JSON or assign new ones?
1. When does ID assignment happen - on first import or always?
1. How do we identify which workflow to export if ID changed during import?
1. What happens on reimport - update in place or create duplicate?

**Needs Testing**: Import a workflow with specific ID, see what ID it gets, export it, reimport.

______________________________________________________________________

### 🚨 Issue 5: Generation Method Undefined

**Problem**: Plan assumes "n8n MCP skills" exist for generation but doesn't verify.

**Current Plan States** (plan.md:242-250):

> "n8n MCP Skills (Generation Engine)"
>
> - Generate valid n8n workflow JSON from natural language descriptions

**Questions**:

1. Do these MCP skills actually exist?
1. Are they available in the self-hosted instance?
1. Or should we manually author JSON based on examples?

**Needs Verification**: Query MCP server for available tools/skills.

______________________________________________________________________

## Proposed Revised Architecture

### Principle: Deployment Happens Outside n8n

**Why**: Avoid circular dependency, maintain clean separation, use simpler tools.

### Layer 1: Workflows (Documentation)

- **Location**: `gsnake-n8n/workflows/`
- **Format**: Markdown SOPs
- **Purpose**: Define WHAT workflows should do
- **Examples**: `workflows/service/notify-discord.md`

### Layer 2: Tools (Implementation)

- **Location**: `gsnake-n8n/tools/`
- **Subdirectories**:
  - `tools/n8n-flows/` - Generated n8n workflow JSON files
  - `tools/scripts/` - **NEW** - Deployment/management shell scripts
- **Purpose**: Generated artifacts and deployment tooling

### Layer 3: Agents (Orchestration)

- **Actor**: Claude via MCP
- **Responsibilities**:
  1. Read markdown SOPs
  1. Generate workflow JSON (using MCP skills OR manual authoring)
  1. Save JSON to `tools/n8n-flows/`
  1. Call deployment script to sync to n8n
  1. Test via MCP `execute_workflow`
  1. Report results

______________________________________________________________________

## Proposed Deployment Flow

### Script: `tools/scripts/sync-workflows.sh`

```bash
#!/bin/bash
# Sync workflows between git and n8n instance

CONTAINER_ID="440742681e58b8049db5f7541c5ce24312bd348662e6a68bab55720f7d16d30e"
FLOWS_DIR="$(dirname "$0")/../n8n-flows"
TMP_DIR="/tmp/n8n-sync"

case "$1" in
  import)
    echo "Importing workflows to n8n..."
    # Copy JSON files into container
    docker cp "$FLOWS_DIR" "$CONTAINER_ID:$TMP_DIR"
    # Import all workflows
    docker exec "$CONTAINER_ID" n8n import:workflow --separate --input="$TMP_DIR"
    # Cleanup
    docker exec "$CONTAINER_ID" rm -rf "$TMP_DIR"
    ;;

  export)
    echo "Exporting workflows from n8n..."
    # Create temp directory in container
    docker exec "$CONTAINER_ID" mkdir -p "$TMP_DIR"
    # Export all workflows
    docker exec "$CONTAINER_ID" n8n export:workflow --backup --output="$TMP_DIR"
    # Copy back to host
    docker cp "$CONTAINER_ID:$TMP_DIR/." "$FLOWS_DIR/"
    # Cleanup
    docker exec "$CONTAINER_ID" rm -rf "$TMP_DIR"
    ;;

  sync)
    echo "Full sync: export -> import"
    $0 export
    $0 import
    ;;

  *)
    echo "Usage: $0 {import|export|sync}"
    exit 1
    ;;
esac
```

### Usage Flow:

1. **Claude generates workflow JSON** → saves to `tools/n8n-flows/workflow-name.json`
1. **Claude calls** `tools/scripts/sync-workflows.sh import`
1. **Script copies files into container** via `docker cp`
1. **Script runs** `n8n import:workflow --separate`
1. **Claude calls** `tools/scripts/sync-workflows.sh export` to capture canonical IDs
1. **Claude tests** via MCP `execute_workflow`
1. **User commits** updated JSON with canonical IDs

______________________________________________________________________

## Open Questions Requiring Decisions

### Q1: Volume Mount Strategy

**Question**: Should we mount `tools/n8n-flows/` as a Docker volume to simplify file transfer?

**Option A - Docker CP (Proposed Above)**:

- ✅ No changes to docker-compose
- ✅ Works with existing setup
- ❌ Requires copy step (slower)

**Option B - Volume Mount**:

- ✅ Faster (no copy needed)
- ✅ Simpler script
- ❌ Requires modifying docker-compose.yml (not in our repo)

**Recommendation**: Start with Option A (docker cp), upgrade to Option B if needed.

______________________________________________________________________

### Q2: Workflow Generation Method

**Question**: How do we generate n8n workflow JSON from markdown SOPs?

**Option A - n8n MCP Skills**:

- ✅ Automated generation
- ✅ AI understands n8n patterns
- ❌ Requires MCP skills plugin (unverified availability)

**Option B - Manual JSON Authoring**:

- ✅ Full control
- ✅ No dependencies
- ❌ Labor intensive
- ❌ Error prone

**Option C - Export + Modify**:

- ✅ Start from working examples
- ✅ Learn n8n structure
- ❌ Still manual work

**Recommendation**: Verify if MCP skills exist. If not, use Option C for POC.

______________________________________________________________________

### Q3: Workflow ID Behavior

**Question**: How does `n8n import:workflow` handle IDs?

**Needs Testing**:

1. Create workflow with ID "abc123" in JSON
1. Import it
1. Export it
1. Check if ID preserved or changed
1. Reimport the exported JSON
1. Verify update-in-place vs duplicate creation

**Impact**: Determines if we need two-step "import then export" flow or if single import preserves IDs.

______________________________________________________________________

### Q4: Credential Binding Workflow

**Question**: When/how are credentials bound to workflows?

**Known**:

- Workflows reference credentials by name (e.g., `$credentials.github_token`)
- Credentials must exist in n8n before workflow can execute
- Credential values stored in n8n, not in JSON

**Unknown**:

1. Can workflow JSON specify credential NAME without value?
1. Or must credentials be manually bound in UI after import?
1. Does export capture credential bindings?

**Impact**: Determines if credential setup is one-time manual or part of sync flow.

______________________________________________________________________

### Q5: MCP Server Tool Inventory

**Question**: What tools does the n8n MCP server actually provide?

**Need to Check**:

- `search_workflows`
- `get_workflow_details`
- `execute_workflow`
- Any workflow generation/authoring tools?

**Why It Matters**: Determines our generation strategy and testing capabilities.

______________________________________________________________________

## Next Steps

### 1. Test & Verify (Do Now)

- [ ] Test workflow ID preservation on import/export
- [ ] List available MCP server tools
- [ ] Test credential reference in imported workflows
- [ ] Verify `--separate` import behavior (one file = one workflow?)

### 2. Create Tools (After Verification)

- [ ] Write `tools/scripts/sync-workflows.sh` with correct file transfer method
- [ ] Test import of one example workflow
- [ ] Test export and verify ID preservation

### 3. Rewrite Documentation (After Tools Work)

- [ ] Update `gsnake-specs/n8n/plan.md` with revised architecture
- [ ] Update `gsnake-specs/n8n/flows.md` to remove circular workflow-sync dependency
- [ ] Update `gsnake-specs/n8n/brief.md` if needed
- [ ] Deprecate `workflows/actions/node-docker-volume-sync.md` (old approach)

### 4. POC Implementation (After Docs Updated)

- [ ] Choose one simple workflow to implement (e.g., notify-discord)
- [ ] Write markdown SOP
- [ ] Generate/author JSON
- [ ] Deploy via script
- [ ] Test via MCP
- [ ] Validate end-to-end flow

______________________________________________________________________

## Decision Points for User

**Please confirm or adjust these key decisions:**

1. **Deployment Method**: Use shell script with `docker cp` instead of internal n8n workflow?
1. **Generation Method**: Manual JSON authoring for POC, or wait until we verify MCP skills?
1. **File Transfer**: Start with `docker cp`, or modify docker-compose to add volume mount?
1. **Testing Priority**: Which question should we answer first via testing?

Once we align on these, I'll rewrite the plan.md/flows.md/brief.md with the corrected architecture.
