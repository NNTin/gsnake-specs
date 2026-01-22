## Objective

Migrate the existing Svelte web client into a fresh `gsnake-web` repository and attach it to the parent as a git submodule.

## Scope

**Included:**

- Create the `gsnake-web` repository at `git@github.com:NNTin/gsnake-web.git` with a new history
- Move the existing Svelte app into the new repo and commit
- Add `gsnake-web` as a submodule in the parent repo
- Ensure all Svelte-related configs live inside `gsnake-web`
- Update parent references (paths/scripts/docs) to the new submodule path

**Excluded:**

- Playwright e2e tests (remain in the parent repo)
- CI workflow changes
- Web feature work beyond the move

## Spec References

- spec:epics/folder-structure/epic-brief.md - Requirements, "Web App Migration" and "Web Submodule Boundary"
- spec:epics/folder-structure/core-flows.md - Flow 0: Web Submodule Migration
- spec:epics/folder-structure/tech-plan.md - gsnake-web (Svelte Frontend), "Migration Note"

## Implementation Details

Create the `gsnake-web` repo as a fresh history, migrate the current Svelte app files into it, and add it to the parent repository as a submodule. Update any parent references to point at `gsnake-web`, keeping Playwright configs/tests at the root.

## Acceptance Criteria

- [ ] `gsnake-web` exists at `git@github.com:NNTin/gsnake-web.git` with a new history
- [ ] The Svelte app source and configs live only under `gsnake-web`
- [ ] The parent repo includes `gsnake-web` as a git submodule pointer
- [ ] Root references/scripts/docs point to the `gsnake-web` path
- [ ] Playwright e2e tests remain in the parent repo
- [ ] Parent repo build and e2e testing continue to work

## Dependencies

- Depends on: t1-restructure-repo-layout

## Validation Notes

- Build from the parent repo using the existing integration build command/script.
- Run Playwright e2e tests from the parent repo and confirm they pass with the submodule checked out.
