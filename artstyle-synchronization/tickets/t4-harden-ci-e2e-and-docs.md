## Title

Harden cross-repo CI/E2E validation and update architecture documentation for shared UI workflows.

## Scope

**In Scope:**

- Update `gsnake-web` CI to build workspace packages in order and fail on sprite/type validation mismatches.
- Update `gsnake-editor` CI to validate standalone install/build/test flow using vendored `gsnake-web-ui` snapshot.
- Update root repository CI to run integrated build/test/E2E flow with shared UI dependency wiring.
- Ensure existing Playwright integration coverage still passes with synchronized artstyle architecture.
- Update docs in impacted repos for:
  - root-repo setup and standalone setup behavior
  - expected handling of breaking shared-UI changes
  - developer workflow for updating shared styles/assets/components

**Out of Scope:**

- New product features unrelated to artstyle synchronization.
- Alternative dependency management models (e.g., version pinning, npm publishing).

## Spec References

- `gsnake-specs/artstyle-synchronization/brief.md` - `## Summary` (success criteria for standalone CI, root E2E, and documentation updates).
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Flow 5: gsnake-web CI - Standalone Build`.
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Flow 6: gsnake-editor CI - Standalone Build`.
- `gsnake-specs/artstyle-synchronization/flows.md` - `## Flow 7: Root Repository E2E Integration Tests`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Component Architecture` > `### CI Integration`.
- `gsnake-specs/artstyle-synchronization/plan.md` - `## Key Technical Constraints` (E2E must pass, standalone CI must pass).

## Dependencies

- T2 - Build `gsnake-web-ui` as the shared design system package.
- T3 - Migrate `gsnake-editor` atomically to shared UI consumption.
