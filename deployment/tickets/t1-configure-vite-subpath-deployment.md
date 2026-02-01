## Objective

Modify file:vite.config.ts to support dynamic base path configuration via environment variable, enabling the game to be deployed to GitHub Pages subpaths (`/gSnake/main/`, `/gSnake/v0.1.0/`, etc.).

## Scope

**Included:**

- Update file:vite.config.ts to read `VITE_BASE_PATH` environment variable
- Set default fallback to `/` for local development
- Verify build works with and without environment variable set

**Excluded:**

- GitHub Actions workflow implementation (separate ticket)
- Actual deployment to GitHub Pages (separate ticket)
- Testing on deployed URLs (done after deployment)

## Spec References

- spec:epics/deployment/tech-plan.md - Tech Plan, Section "Base Path Injection"
- spec:epics/deployment/tech-plan.md - Tech Plan, Section "Vite Configuration"

## Implementation Details

Modify file:vite.config.ts to include:

```typescript
base: process.env.VITE_BASE_PATH || '/'
```

This allows the workflow to inject different base paths for different deployment targets:

- Main branch: `VITE_BASE_PATH=/gSnake/main/`
- Tagged versions: `VITE_BASE_PATH=/gSnake/v{version}/`
- Local development: defaults to `/`

## Acceptance Criteria

- [ ] file:vite.config.ts reads `VITE_BASE_PATH` environment variable
- [ ] Local development works without setting environment variable (defaults to `/`)
- [ ] Build succeeds when `VITE_BASE_PATH` is set to `/gSnake/test/`
- [ ] Build output references assets with correct base path when environment variable is set

## Dependencies

None - this is a foundational change that other tickets depend on.
