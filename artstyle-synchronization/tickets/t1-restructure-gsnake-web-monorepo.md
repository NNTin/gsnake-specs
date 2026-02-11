## Title

Restructure `gsnake-web` into an npm workspace monorepo foundation.

## Scope

**In Scope:**

- Convert `gsnake-web` root to a private npm workspace with:
  - `packages/gsnake-web-ui`
  - `packages/gsnake-web-app`
- Move/split current web app sources into `packages/gsnake-web-app` without changing gameplay behavior.
- Establish package manifests, scripts, and workspace wiring so `npm install` and package-local builds run from repo root.
- Align `gsnake-web-app` to Svelte 5 as required by the architecture.
- Preserve root-repo local WASM dependency behavior for `gsnake-web-app`.

**Out of Scope:**

- Implementing final shared styles/assets/components (`gsnake-web-ui` contents are covered in T2).
- Migrating `gsnake-editor` to shared UI (covered in T3).
- CI workflow and documentation changes (covered in T4).

## Spec References

- `gsnake-specs/artstyle-synchronization/brief.md` - `## Summary` (success criteria for monorepo split and shared UI package).
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Flow 1: Initial Setup - Root Repository Mode`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Architectural Approach` > `### Monorepo Strategy`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Architectural Approach` > `### Svelte Version Alignment`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Data Model` > `### Package Structure`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Data Model` > `### npm Workspaces Configuration`.

## Dependencies

None. This is the foundation ticket.
