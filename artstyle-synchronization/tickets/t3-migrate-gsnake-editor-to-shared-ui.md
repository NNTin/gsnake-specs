## Title

Migrate `gsnake-editor` atomically to consume `gsnake-web-ui` in both root-repo and standalone modes.

## Scope

**In Scope:**

- Add editor-side dependency auto-detection script for mode switching:
  - root-repo mode: local file dependency to `../gsnake-web/packages/gsnake-web-ui`
  - standalone mode: download/vendor `gsnake-web-ui` snapshot from `main` and depend on vendored path
- Wire detection into install flow (preinstall) so setup is automatic.
- Remove existing editor-specific dark theme styles and import shared light-theme styles from `gsnake-web-ui`.
- Rebuild editor UI surfaces to use shared base components (`Modal`, `Overlay`) and shared visual assets.
- Complete migration in one cohesive change set to avoid partial mixed-theme states.

**Out of Scope:**

- Version pinning or npm publishing strategy changes.
- Changes to core game/WASM logic not required for UI integration.
- CI pipeline hardening tasks (covered in T4).

## Spec References

- `gsnake-specs/artstyle-synchronization/brief.md` - `## Summary` (editor must depend on shared UI and adopt synchronized artstyle).
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Flow 2: Initial Setup - Standalone gsnake-editor Mode`.
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Flow 6: gsnake-editor CI - Standalone Build`.
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Flow 8: Handling Breaking Changes in gsnake-web-ui`.
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Flow 9: Atomic Migration of gsnake-editor`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Architectural Approach` > `### Dependency Resolution Strategy`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Data Model` > `### Auto-Detection in gsnake-editor`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Component Architecture` > `### Integration Points`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Component Architecture` > `### Migration Strategy`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Key Technical Constraints` (no version pinning, breaking changes acceptable).

## Dependencies

- T1 - Restructure `gsnake-web` into a workspace monorepo foundation.
- T2 - Build `gsnake-web-ui` as the shared design system package.
