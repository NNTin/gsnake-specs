## Summary

This feature introduces an automated feedback loop for GitHub issues whose titles start with `[n8n]`.

When an issue is opened/edited/labeled (and unlabeled for re-entry), n8n evaluates whether the issue should be analyzed by an LLM. If eligible, n8n calls a deterministic analyzer interface, posts a structured analysis comment, and then applies a lock label so the issue is ignored until human intervention.

## Problem

`[n8n]` issues are not consistently triaged. Analysis quality and response time depend on manual attention.

## Goal

Create a reliable, auditable, low-noise loop:

1. Detect relevant issue events.
1. Analyze issue content with LLM support.
1. Post structured recommendations.
1. Pause further automation until a human explicitly unlocks the issue.

## Scope (MVP)

- Trigger source: GitHub issue webhook -> n8n
- Events: `opened`, `edited`, `labeled`, `unlabeled`
- Filter: title prefix `[n8n]`
- Lock behavior:
  - If `needs-human-review` exists, skip automation.
  - If `llm-analyzed` exists, skip automation (compatibility with current idea).
- Analyzer call: external analyzer endpoint/script (hybrid model)
- Output: issue comment with structured sections + labels

## Label Policy

### Current policy (required)

If either label exists, automation must ignore the issue:

- `llm-analyzed`
- `needs-human-review`

This preserves the explicit “human in the loop” gate.

### Target simplification (recommended)

Converge to one lock label (`needs-human-review`) and keep `llm-analyzed` only for analytics (non-lock), to avoid permanent suppression without clear intent.

## Success Criteria

1. `[n8n]` issues are auto-analyzed on eligible events.
1. Automation never re-analyzes while lock labels are present.
1. Removing lock labels and editing/unlabeling re-enables analysis.
1. Comments are deterministic in structure and include confidence + actions.
1. End-to-end execution is observable in n8n logs and GitHub issue timeline.

## Out of Scope

- Fully autonomous issue closure
- Code changes generated and merged automatically
- Replacing CI workflows

## Dependencies

- Existing n8n environment and workflow sync process in [gsnake-n8n/workflows/infra/n8n-sync.md](../../gsnake-n8n/workflows/infra/n8n-sync.md)
- Existing issue-management patterns in [gsnake-n8n/workflows/n8n-workflow/manage-parent-repo-ci-failure-issue.md](../../gsnake-n8n/workflows/n8n-workflow/manage-parent-repo-ci-failure-issue.md)
- GitHub credential in n8n with issue read/write scope
