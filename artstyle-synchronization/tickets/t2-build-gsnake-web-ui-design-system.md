## Title

Build `gsnake-web-ui` as the shared design system package for styles, assets, and base components.

## Scope

**In Scope:**

- Implement `packages/gsnake-web-ui` with:
  - base light-theme styles
  - shared sprite assets (`sprites.svg`)
  - base `Modal` and `Overlay` components
  - single public entrypoint (`index.js`) for CSS/assets/components exports
- Add the missing `FallingFood` sprite and enforce sprite completeness.
- Implement bidirectional sprite/type validation script and integrate it into `gsnake-web-ui` build (prebuild hook).
- Configure `gsnake-web-ui` precompiled output pattern so consumers import built JS/CSS/assets rather than raw `.svelte`.
- Integrate `gsnake-web-app` consumption of `gsnake-web-ui` exports and verify local dev workflow keeps updates visible quickly.

**Out of Scope:**

- `gsnake-editor` dependency and UI migration work (covered in T3).
- CI workflow updates across repositories (covered in T4).
- Publishing to npm or introducing version pinning.

## Spec References

- `gsnake-specs/artstyle-synchronization/brief.md` - `## Summary` (success criteria for shared UI package contents and validation).
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Flow 3: Updating Artstyle in Root Repository`.
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Flow 4: Adding New Game Object with Visual Element`.
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Key Design Decisions` > `### Hot-Reload Development Experience`.
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Key Design Decisions` > `### Bidirectional Validation`.
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Key Design Decisions` > `### Precompiled Consumption`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Component Architecture` > `### gsnake-web-ui Exports`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Component Architecture` > `### Base Component Design`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Component Architecture` > `### Validation Script Architecture`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Component Architecture` > `### Missing Sprite Addition`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Architectural Approach` > `### Precompiled Consumption Pattern`.

## Dependencies

- T1 - Restructure `gsnake-web` into a workspace monorepo foundation.
