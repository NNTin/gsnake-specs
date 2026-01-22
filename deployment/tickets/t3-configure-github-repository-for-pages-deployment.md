## Objective

Configure the GitHub repository settings to enable GitHub Pages deployment from the gh-pages branch and ensure the workflow has necessary permissions.

## Scope

**Included:**

- Enable GitHub Pages in repository settings
- Configure Pages to deploy from gh-pages branch
- Verify workflow has write permissions to repository
- Verify workflow can push to both source and gh-pages branches
- Document any manual setup steps required

**Excluded:**

- Workflow implementation (separate ticket)
- Vite configuration (separate ticket)
- Custom domain setup (explicitly out of scope)

## Spec References

- spec:epics/deployment/epic-brief.md - Epic Brief, Summary and Success Criteria
- spec:epics/deployment/tech-plan.md - Tech Plan, Technical Constraints

## Implementation Details

### GitHub Pages Configuration

1. Navigate to repository Settings → Pages
2. Set Source to "Deploy from a branch"
3. Select branch: `gh-pages`
4. Select folder: `/ (root)`
5. Save configuration

### Workflow Permissions

Ensure GitHub Actions has write permissions:

1. Navigate to repository Settings → Actions → General
2. Under "Workflow permissions", select "Read and write permissions"
3. Check "Allow GitHub Actions to create and approve pull requests" (if needed for commit-back)
4. Save changes

### Verification

- Confirm Pages URL will be: `https://nntin.github.io/gSnake`
- Verify workflow can create gh-pages branch if it doesn't exist
- Verify workflow can push commits to source branch (for package.json updates)

## Acceptance Criteria

- [ ] GitHub Pages enabled in repository settings
- [ ] Pages configured to deploy from gh-pages branch
- [ ] Workflow has write permissions to repository
- [ ] Workflow can push to source branch (for package.json commit-back)
- [ ] Workflow can push to gh-pages branch (for deployment)
- [ ] Pages URL confirmed as `nntin.github.io/gSnake`

## Dependencies

None - this can be done in parallel with other tickets, but must be complete before testing deployments.