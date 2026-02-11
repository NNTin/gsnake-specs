## Summary

This Epic establishes gsnake-web as the single source of truth for visual design across the gSnake project by extracting a shared design system library (gsnake-web-ui) that both gsnake-web and gsnake-editor depend on. Currently, the web game and level editor maintain separate, divergent styling and visual assets, requiring duplicate maintenance and creating inconsistent user experiences. By restructuring gsnake-web into a monorepo with a shared UI library and making gsnake-editor depend on it, any artstyle updates in gsnake-web will automatically propagate to the editor. This approach must work seamlessly in both root repository mode (using local file paths) and standalone mode (using git dependencies), ensuring CI passes in all scenarios without breaking existing E2E integration tests.

## Context & Problem

**Who's Affected:**

- **Developers** maintaining gsnake-web and gsnake-editor face duplicate work when updating visual styles, components, or assets
- **End users** experience inconsistent visual design between the game interface and level editor
- **Contributors** working on either project independently need clear dependency management

**Current State:**

- file:gsnake-web/app.css defines a light theme with minimal styling
- file:gsnake-editor/src/app.css defines a dark theme with elaborate styling
- Visual assets (SVG sprites) exist only in file:gsnake-web/assets/
- No shared component library - modals, overlays, and UI elements are duplicated
- Both projects are git submodules that can be checked out independently or as part of the root repo
- gsnake-web uses auto-detection (file:gsnake-web/scripts/detect-local-deps.js) to switch between local WASM and git dependencies
- gsnake-editor has no dependency on gsnake-web

**The Pain:**

1. **Design Drift**: When visual updates happen in gsnake-web, they don't propagate to gsnake-editor, causing the two interfaces to diverge over time
1. **Duplicate Maintenance**: Developers must manually sync styling changes across both projects
1. **No Single Source of Truth**: There's no authoritative design system - each project maintains its own styles
1. **Inconsistent User Experience**: Users see different visual languages in the game vs. the editor
1. **Complex Dependency Management**: Need to support both root repo mode (local paths) and standalone mode (git dependencies) while keeping CI green

**Success Criteria:**

- gsnake-web is restructured into a monorepo with `packages/gsnake-web-ui` (shared library) and `packages/gsnake-web-app` (game application)
- gsnake-web-ui contains styles, assets, and base UI components (Modal, Overlay)
- gsnake-editor depends on gsnake-web-ui and adopts its light theme
- Auto-detection script in gsnake-editor switches between local and git dependencies
- CI validation ensures game object elements in gsnake-web-ui match WASM definitions
- All existing E2E tests in root repo continue to pass
- Both gsnake-web and gsnake-editor standalone CI workflows pass
- Documentation is updated to reflect the new architecture

**Out of Scope:**

- Publishing gsnake-web-ui to npm (using git dependencies only)
- Version pinning (always using latest from main branch)
- Migrating game-specific components to the shared library (only styles, assets, and base components)

&#160;
