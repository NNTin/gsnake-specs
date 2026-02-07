## Architectural Approach

### Core Architecture Pattern: WAT Framework

The system implements a three-layer architecture that separates concerns between probabilistic AI reasoning and deterministic execution:

**Layer 1: Workflows (Documentation as Code)**

- Markdown SOPs in file:workflows/ serve as the single source of truth
- Human-readable documentation that is also machine-executable
- Version controlled in git alongside generated artifacts

**Layer 2: Agents (AI Orchestration)**

- Claude AI reads markdown SOPs and orchestrates the generation pipeline
- Uses n8n MCP skills to generate valid n8n workflow JSON
- Handles deployment via MCP server execution
- Performs validation and iterative refinement

**Layer 3: Tools (Deterministic Execution)**

- n8n workflows execute the actual automation logic
- n8n CLI (import/export) is used to write/read workflows in the n8n backing store
- n8n MCP server provides programmatic execution/testing and is used by Claude to trigger the internal sync workflow

### Key Architectural Decisions

**1. AI-Powered Workflow Generation**

- **Decision**: Use n8n MCP skills plugin for workflow generation
- **Rationale**: Leverages specialized AI knowledge of n8n's node structure, expression syntax, and workflow patterns
- **Trade-off**: Depends on AI quality, but eliminates manual JSON authoring
- **Constraint**: Generated workflows must conform to n8n's JSON schema

**2. Workflow Sync via n8n CLI (import/export)**

- **Decision**: Use `n8n import:workflow` and `n8n export:workflow` as the mechanism to write/read workflows.
- **Rationale**: Avoids relying on assumptions about n8n auto-loading workflows from files in the data volume.
- **Trade-off**: Requires validating CLI semantics once (especially around IDs and overwrite behavior).
- **Constraint**: CLI execution must be possible from within the n8n runtime (or via a controlled fallback).

**3. Git as Source of Truth**

- **Decision**: Commit both markdown SOPs and generated JSON to version control
- **Rationale**: Full reproducibility, change tracking, and collaboration support
- **Trade-off**: Generated files in git can create noise, but provides audit trail
- **Constraint**: Requires discipline to keep markdown and JSON synchronized

**4. MCP-Based Deployment (internal-only)**

- **Decision**: Claude triggers an internal “workflow-sync” workflow via MCP `execute_workflow`.
- **Rationale**: Keeps the sync mechanism private (no internet-facing webhook) while enabling end-to-end automation.
- **Trade-off**: Requires a one-time manual bootstrap to create the “workflow-sync” workflow.
- **Constraint**: MCP server must be accessible and authenticated.

**5. In-Place Workflow Updates (via canonical export)**

- **Decision**: Preserve workflow IDs by storing *canonical* JSON exported from n8n in git, then re-importing it for updates.
- **Rationale**: CLI import commonly assigns new IDs on first import; exporting after import captures the n8n-assigned IDs so future imports can overwrite the same workflow.
- **Trade-off**: First-time creation becomes a two-step: import → export → commit.
- **Constraint**: The sync workflow must support exporting the affected workflow(s) after import.

**6. Filename-Based Naming Convention**

- **Decision**: Derive workflow names from SOP filenames
- **Rationale**: Clear, predictable mapping between documentation and implementation
- **Trade-off**: Filename changes require workflow recreation
- **Constraint**: Filenames must be valid n8n workflow names

**7. n8n Credential System for Secrets**

- **Decision**: Use n8n's built-in credential management
- **Rationale**: Secure, centralized secret storage with access control
- **Trade-off**: Credentials must be manually configured in n8n UI first
- **Constraint**: Workflows reference credentials by name, not value

### Technical Constraints

**Infrastructure Constraints**:

- Self-hosted n8n instance at `https://n8n.labs.lair.nntin.xyz/`
- MCP server provides read-only access (search/execute/details); workflow *writes* happen via n8n CLI import/export inside the n8n runtime

**Security Constraints**:

- n8n MCP server authentication via Bearer token
- Secrets are managed via the n8n credential system (manual one-time binding step in the POC)

**Operational Constraints**:

- Bootstrap dependency: the internal “workflow-sync” workflow must be manually created first
- No automated rollback mechanism (out of scope for POC)
- Simple linear workflows only (no complex branching/loops)

______________________________________________________________________

## Data Model

### Workflow Definition Schema

**Markdown SOP Structure** (workflows/service/*.md or workflows/actions/*.md):

```markdown
# Workflow Purpose

## Objective
What this workflow accomplishes

## Inputs
- Parameter 1: description, type, required/optional
- Parameter 2: description, type, required/optional

## Steps
1. Step description
2. Step description

## Outputs
- Output 1: description, type
- Output 2: description, type

## Error Handling
Expected failures and recovery strategies
```

**Generated n8n Workflow JSON** (tools/n8n-flows/\*.json):

```json
{
  "name": "workflow-name",
  "nodes": [...],
  "connections": {...},
  "settings": {...},
  "staticData": null,
  "tags": [],
  "triggerCount": 0,
  "updatedAt": "timestamp",
  "versionId": "uuid"
}
```

### Workflow Metadata Mapping

| Markdown SOP | n8n Workflow JSON | Derivation Rule |
| ----------------- | ------------------------------- | ------------------------------------------------ |
| Filename | `name` field | `docker-volume-sync.md` → `"docker-volume-sync"` |
| Objective section | Workflow description/notes | Copied verbatim |
| Inputs section | Webhook/trigger node parameters | Mapped to node configuration |
| Steps section | Node sequence | Translated to nodes + connections |
| Outputs section | Final node outputs | Mapped to response/action nodes |

### Workflow Identification Strategy

**Primary Key**: Workflow name (derived from filename)\
**Secondary Key**: n8n-assigned workflow ID (captured via `n8n export:workflow` after first import)

**ID Preservation for Updates (POC)**:

1. First creation: import the generated JSON (ID may be assigned/normalized by n8n)
1. Immediately export the workflow from n8n and commit that exported JSON as the *canonical* JSON in git
1. Subsequent updates: regenerate by starting from the canonical JSON, preserving the workflow ID, then re-import
1. Re-export after import when you need to confirm/canonicalize (and after manual credential binding)

### Credential References

Workflows reference credentials by name, not value:

```javascript
// In n8n Code node
const apiKey = $credentials.github_token;
const secret = $env.N8N_WEBHOOK_SECRET;
```

**Credential Naming Convention**:

- Service-based: `github_token`, `discord_webhook`, `n8n_api_key`
- Environment-based: `production_db`, `staging_api`

### Workflow Sync Data Flow (CLI import/export)

**Authority / conflict rule (POC)**:

- Git is the source of truth
- Sync primarily applies git → n8n (overwrite n8n state when possible)

**Import (git → n8n)**:

- Input: workflow JSON from git (generated or canonical)
- Action: `n8n import:workflow`
- Post-step: export the imported workflow to capture/confirm IDs and metadata

**Export (n8n → git)**:

- Purpose: backup + normalization (capture n8n-assigned IDs and any UI-applied metadata, e.g., credential bindings)
- Action: `n8n export:workflow`

**Conflict handling (POC)**:

- If git JSON differs from n8n export, the git version is re-imported; the export is used only to update git after import and to capture IDs/metadata.

______________________________________________________________________

## Component Architecture

```mermaid
graph TD
    User[User] -->|Chat| Claude[Claude Agent]
    Claude -->|Read| SOPs[Markdown SOPs<br/>workflows/]
    Claude -->|Use| Skills[n8n MCP Skills]
    Skills -->|Generate| JSON[Workflow JSON<br/>tools/n8n-flows/]
    Claude -->|Execute| MCP[n8n MCP Server]
    MCP -->|Trigger| SyncWF[Internal "workflow-sync" workflow]
    SyncWF -->|Run| CLI[n8n CLI import/export]
    CLI -->|Read/Write| Store[n8n backing store]
    Claude -->|Commit| Git[Git Repository]
    JSON -->|Tracked in| Git
    SOPs -->|Tracked in| Git
    MCP -->|Test/Execute| n8n
    n8n -->|Results| Claude
    Claude -->|Feedback| User
```

### Component Responsibilities

**1. Claude Agent (Orchestrator)**

- **Responsibilities**:
  - Parse user requests and clarify requirements
  - Read markdown SOPs from file:workflows/
  - Invoke n8n MCP skills to generate workflow JSON
  - Save generated JSON to file:tools/n8n-flows/
  - Execute the internal “workflow-sync” workflow via MCP server (import/export)
  - Test deployed workflows via MCP server
  - Report results and errors to user
  - Commit changes to git
- **Interfaces**:
  - Input: User chat messages
  - Output: Generated workflows, test results, status updates
  - Dependencies: n8n MCP skills, n8n MCP server, filesystem, git

**2. n8n MCP Skills (Generation Engine)**

- **Responsibilities**:
  - Generate valid n8n workflow JSON from natural language descriptions
  - Ensure correct node configuration and connections
  - Apply n8n best practices and patterns
  - Validate expression syntax and code snippets
- **Interfaces**:
  - Input: Workflow requirements from Claude
  - Output: n8n workflow JSON structure
  - Skills used: `n8n-workflow-patterns`, `n8n-node-configuration`, `n8n-code-javascript`, `n8n-expression-syntax`

**3. n8n MCP Server (Execution Interface)**

- **Responsibilities**:
  - Provide programmatic access to n8n instance
  - Execute workflows on demand
  - Search for existing workflows
  - Retrieve workflow details and execution results
- **Interfaces**:
  - Endpoint: `https://n8n.labs.lair.nntin.xyz/mcp-server/http`
  - Authentication: Bearer token
  - Tools: `search_workflows`, `get_workflow_details`, `execute_workflow`

**4. Internal “workflow-sync” Workflow (Deployment Mechanism)**

- **Responsibilities**:
  - Import workflow JSON into n8n using `n8n import:workflow`
  - Export workflow JSON from n8n using `n8n export:workflow` (backup + canonicalization)
  - Return a structured summary (what was imported/exported)
- **Interfaces**:
  - Trigger: Internal execution via MCP `execute_workflow` (no internet-facing webhook in POC)
  - Input: action (import/export), target workflow(s)
  - Output: status + references to exported artifacts
- **Node Structure (high-level)**:
  - Trigger node (manual/execute)
  - Command execution step to run n8n CLI
  - Response step with structured summary

**(Optional fallback) Docker socket access (avoid for POC)**

- **Status**: Not used in the primary CLI-based approach. Only consider if CLI import/export cannot be executed from within n8n.

**5. Git Repository (Version Control)**

- **Responsibilities**:
  - Track markdown SOPs and generated JSON
  - Provide change history and audit trail
  - Enable collaboration and rollback
- **Structure**:
  - file:workflows/service/ - Service workflow SOPs (webhook endpoints)
  - file:workflows/actions/ - Action workflow SOPs (custom nodes)
  - file:tools/n8n-flows/ - Generated n8n workflow JSON files
  - file:CLAUDE.md - WAT Framework documentation

### Integration Points

**Claude ↔ n8n MCP Skills**:

- Claude provides workflow requirements in natural language
- Skills return structured n8n workflow JSON
- Iterative refinement if validation fails

**Claude ↔ n8n MCP Server**:

- Claude calls `execute_workflow` to trigger the internal “workflow-sync” workflow (import/export)
- Claude calls `execute_workflow` to test generated workflows
- Claude calls `search_workflows` to find existing workflows for updates
- Claude calls `get_workflow_details` to inspect workflow configuration

**Sync workflow ↔ n8n CLI**:

- The internal “workflow-sync” workflow executes `n8n import:workflow` and `n8n export:workflow`
- CLI import/export is the authoritative mechanism for writing/reading workflows
- Export output is used for backup + canonicalization into git

**n8n CLI ↔ n8n backing store**:

- CLI reads/writes workflows in n8n’s backing store (DB)
- This avoids relying on file-copy semantics in the n8n data directory
- After import, workflows are available in n8n UI and can be executed via MCP

**Claude ↔ Git**:

- Claude commits both markdown SOPs and generated JSON
- Commit messages reference the workflow name and change type
- Git provides version history for troubleshooting

### Failure Modes and Recovery

**Workflow Generation Failure**:

- Cause: Invalid SOP, unclear requirements, AI hallucination
- Detection: JSON validation fails, n8n rejects workflow
- Recovery: Claude analyzes error, refines SOP, regenerates

**Sync Workflow Failure (CLI execution)**:

- Cause: CLI not available/allowed, missing permissions, incorrect paths, unexpected CLI semantics
- Detection: Command execution step returns error
- Recovery: Manual CLI run on host/container to unblock; adjust sync workflow accordingly

**Deployment Failure**:

- Cause: n8n can't parse JSON, missing credentials, invalid node configuration
- Detection: Workflow doesn't appear in n8n UI or shows errors
- Recovery: Claude inspects workflow details, fixes JSON, re-syncs

**Test Execution Failure**:

- Cause: Workflow logic error, missing inputs, credential issues
- Detection: MCP `execute_workflow` returns error or unexpected output
- Recovery: Claude analyzes execution logs, updates workflow, re-deploys

### Bootstrap Sequence

**Initial Setup** (One-time, manual):

1. User manually creates the internal “workflow-sync” workflow in n8n UI
1. User verifies the workflow can run n8n CLI import/export commands in the n8n runtime
1. User sets up any required n8n credentials (manual binding step happens after first deploy)
1. User configures MCP server authentication

**First Workflow Generation** (Automated):

1. User requests workflow creation via Claude chat
1. Claude reads markdown SOP
1. Claude generates workflow JSON using MCP skills
1. Claude saves JSON to file:tools/n8n-flows/
1. Claude executes the internal “workflow-sync” workflow via MCP (Import)
1. Claude executes the internal “workflow-sync” workflow via MCP (Export) to capture canonical JSON with IDs
1. User performs any one-time credential binding in n8n UI (if required)
1. Claude re-exports if credential bindings changed
1. Claude tests workflow via MCP
1. Claude reports results to user
1. User commits markdown + canonical JSON to git
