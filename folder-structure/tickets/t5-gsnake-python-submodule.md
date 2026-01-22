## Objective

Create the `gsnake-python` submodule boundary as an empty repository placeholder.

## Scope

**Included:**

- Add a `gsnake-python` submodule pointer to an empty repo
- Ensure the parent repo reflects the placeholder boundary in the tree

**Excluded:**

- Any Python implementation or packaging
- Python CI or tooling setup

## Spec References

- spec:epics/folder-structure/tech-plan.md - Architectural Approach, "gsnake-python (Placeholder)"
- spec:epics/folder-structure/epic-brief.md - Summary

## Implementation Details

Add the submodule pointer in the parent repository with no implementation content to establish the boundary for future Python work.

## Acceptance Criteria

- [ ] `gsnake-python` is a submodule pointer to an empty repository
- [ ] Parent repository tree reflects the empty submodule boundary

## Dependencies

- Depends on: t1-restructure-repo-layout
