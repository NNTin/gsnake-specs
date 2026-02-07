## Summary

This Epic validates the feasibility of the WAT Framework (Workflows, Agents, Tools) by proving that markdown Standard Operating Procedures can be automatically translated into working n8n workflows using AI. The goal is to eliminate the tedious manual process of building n8n workflows through the UI by defining workflows as markdown documents that serve as both documentation and executable specifications. Success means demonstrating one complete end-to-end example: translating the file:workflows/service/docker-volume-sync.md SOP into a functional n8n workflow that can be deployed and executed on a self-hosted n8n instance. This proof of concept will also refine the WAT Framework documentation (file:CLAUDE.md) to eliminate ambiguities and establish clear patterns for future workflow development.

## Context & Problem

**Who's Affected**\
nntin, who is building automation pipelines for high-level orchestration (triggering jobs, coordinating workflows, making decisions) using a self-hosted n8n instance.

**The Current Pain**\
Creating n8n workflows manually through the UI is tedious and time-consuming. Each workflow requires clicking through the interface to add nodes, configure parameters, connect components, and test execution. This manual process doesn't scale well as the number of workflows grows, and it creates a disconnect between documentation (markdown SOPs) and implementation (n8n workflows). Starting fresh with n8n, there's an opportunity to establish a better workflow creation process from the beginning.

**Where This Happens**\
The gsnake-n8n repository, which implements the WAT Framework architecture:

- **Workflows Layer**: Markdown SOPs in file:workflows/ defining what should be done
- **Agents Layer**: Claude AI orchestrating and making decisions based on workflows
- **Tools Layer**: n8n workflows in file:tools/n8n-flows/ that execute deterministic operations

**The Vision**\
Markdown SOPs become the single source of truth. An AI agent reads the SOP, uses n8n MCP skills to generate the corresponding n8n workflow JSON, and deploys it to the self-hosted instance. Workflows stay synchronized with their documentation because they're generated from the same source. Changes to SOPs automatically propagate to n8n workflows, eliminating documentation drift.

**Technical Constraints**

- Self-hosted n8n instance at `https://n8n.labs.lair.nntin.xyz/`
- n8n MCP server provides read-only access (search, execute, get details)
- Writing workflows requires syncing the Docker volume `nntin-labs-n8n-data`
- n8n MCP skills plugin enabled for AI-powered workflow generation

**Success Criteria**\
The proof of concept is successful when:

1. The file:workflows/service/docker-volume-sync.md SOP is translated to a working n8n workflow using AI
1. The generated workflow is deployed to the self-hosted n8n instance via volume sync
1. The workflow executes successfully when triggered by a GitHub webhook
1. The WAT Framework documentation is refined based on learnings from the POC

**Explicitly Out of Scope**

- Complex workflows with branching logic, loops, or conditional paths
- Sophisticated error recovery and retry mechanisms
- UI or web interface for workflow generation (CLI/manual process is acceptable)
- Production-grade hardening and reliability features
- Multiple workflow types beyond webhook-triggered service workflows
