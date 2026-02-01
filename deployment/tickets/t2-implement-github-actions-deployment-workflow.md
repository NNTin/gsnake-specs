## Objective

Create a complete GitHub Actions workflow that automatically deploys gSnake to GitHub Pages with support for both main branch (development) and tagged version (release) deployments.

## Scope

**Included:**

- Create .github/workflows/deploy.yml with dual trigger support (main push, tag push)
- Implement conditional logic for main vs tag deployments
- Package.json version synchronization for tagged deployments
- Build validation and type checking (`npm run build`, `npm run check`)
- Vite build with dynamic base path configuration
- versions.json management (validation, update, initialization)
- Static page generation (root redirect, versions index)
- Deployment via `peaceiris/actions-gh-pages` action
- Concurrency control to serialize deployments
- Comprehensive error handling (fail-fast strategy)

**Excluded:**

- GitHub Pages repository settings (separate ticket)
- Vite configuration changes (separate ticket)
- Custom 404 page (out of scope per Epic Brief)

## Spec References

- spec:epics/deployment/core-flows.md - Core Flows, Flow 6 & 7 (deployment flows)
- spec:epics/deployment/tech-plan.md - Tech Plan, entire document
- spec:epics/deployment/epic-brief.md - Epic Brief, Version Format Requirements

## Implementation Details

### Workflow Structure

**Triggers:**

- `push` to `main` branch
- `push` of tags matching `v*.*.*` pattern

**Concurrency:**

```yaml
concurrency:
  group: github-pages-deployment
  cancel-in-progress: false
```

**Jobs:**

1. **Validation & Build**

- Checkout code
- Setup Node.js
- Install dependencies
- Run type checking (`npm run check`)
- Conditional: Update package.json version (tags only)
- Conditional: Commit version back to source branch (tags only)
- Set `VITE_BASE_PATH` environment variable (different for main vs tags)
- Build with Vite (`npm run build`)

2. **Deployment**

- Prepare deployment directory
- Conditional: Validate and update versions.json (tags only)
- Conditional: Generate root redirect page (tags only)
- Conditional: Generate versions index page (tags only)
- Deploy to gh-pages using `peaceiris/actions-gh-pages` action

### versions.json Management

**Schema:**

```json
{
  "latestVersion": "v0.1.1",
  "versions": [
    {"version": "v0.1.1", "date": "2026-01-13T10:30:00Z"},
    {"version": "v0.1.0", "date": "2026-01-05T14:20:00Z"}
  ]
}
```

**Operations:**

- Validate JSON structure before updates
- Fail deployment if corrupted
- Initialize on first versioned deployment
- Update only during tag deployments

### Static Page Generation

**Root Redirect (`index.html`):**

- Fetch versions.json
- Redirect to `latestVersion`
- Fallback to `/main` on error

**Versions Index (`versions/index.html`):**

- Fetch versions.json
- Display list with dates
- Show "Latest" badge on current latest
- Clickable links to each version

### Error Handling

- **Package.json commit-back failure**: Fail entire deployment
- **versions.json validation failure**: Fail with clear error message
- **Build failures**: Fail fast, no deployment
- **Concurrent deployments**: Serialized via concurrency groups

## Acceptance Criteria

- [ ] Workflow file created at .github/workflows/deploy.yml
- [ ] Push to main branch triggers deployment to `/main/`
- [ ] Push of tag `v*.*.*` triggers versioned deployment to `/v{version}/`
- [ ] Package.json version updated and committed for tag deployments
- [ ] versions.json created/updated correctly for tag deployments
- [ ] Root redirect page generated with fallback to `/main`
- [ ] Versions index page generated with all versions listed
- [ ] Concurrency groups prevent simultaneous deployments
- [ ] Build failures prevent deployment
- [ ] Invalid versions.json fails deployment with clear error
