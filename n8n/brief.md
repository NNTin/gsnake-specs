## Summary

This Epic validates and documents the implemented WAT workflow model for n8n in `gsnake-n8n/`: markdown SOPs define behavior, workflow JSON files in git define executable artifacts, and deployment is handled by a host-level sync script using n8n CLI import/export. The prior design that used an internal n8n "workflow-sync" workflow has been replaced.

The system now supports a full loop: define/update SOP -> generate/update workflow JSON -> import into n8n -> test execution -> export/canonicalize changes -> commit.

## Context & Problem

**Who is affected**\
nntin, while building and maintaining automation workflows on a self-hosted n8n instance.

**The pain being solved**\
Manual workflow authoring and edits in the n8n UI are slow, error-prone, and drift from markdown documentation.

**Current architecture in production repo (`gsnake-n8n/`)**

- `workflows/` holds SOPs (for example `workflows/infra/`, `workflows/n8n-webhook/`)
- `tools/n8n-flows/` holds git-tracked n8n JSON workflow definitions
- `tools/scripts/sync-workflows.sh` deploys/syncs workflows using `n8n import:workflow` and `n8n export:workflow`
- MCP server is used for workflow discovery/details/execution testing, not direct workflow writes

## Confirmed Technical Reality

- n8n is self-hosted at `https://n8n.labs.lair.nntin.xyz/`
- n8n CLI import/export is available in container `n8n`
- Sync uses `docker cp` + n8n CLI (outside-n8n deployment path)
- Workflow IDs are preserved on import; re-import with same ID updates in place
- Workflow files are stored by ID (`tools/n8n-flows/{workflow-id}.json`)
- Imported workflows are typically inactive by default and activated after validation
- Credentials/secrets are managed in n8n/environment, not committed into git

## Success Criteria

1. `brief.md`, `flows.md`, and `plan.md` reflect the implemented external sync architecture.
1. Workflow lifecycle is documented as git-first with `sync-workflows.sh` (`import`, `export`, `sync`).
1. Specs document ID-based file naming and in-place workflow updates via stable IDs.
1. Specs describe MCP correctly as read/details/execute support for testing and validation.
1. Deprecated internal sync workflow approach is no longer part of the primary design.

## Explicitly Out Of Scope

- Reintroducing an internal n8n workflow to deploy workflows
- Direct workflow writes through MCP server
- Fully automated credential provisioning/rotation in this POC
- Production-grade rollback orchestration and CI/CD enforcement
- Complex orchestration patterns beyond current implemented workflows
