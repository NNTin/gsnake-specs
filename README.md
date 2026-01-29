# gsnake-specs

Documentation and specifications for the gSnake project.

## Overview

This repository contains:
- Epic briefs and technical plans
- Task definitions and PRDs (Product Requirements Documents)
- Architecture documentation
- Core flows and requirements
- Findings and research notes

## Repository Structure

```
gsnake-specs/
├── deployment/          # Deployment specifications
├── e2e-testing/        # End-to-end testing specs
├── findings/           # Research findings and notes
├── folder-structure/   # Repository architecture
├── gravity-snake/      # Core game specifications
├── level-editor/       # Level editor requirements
├── level-management/   # Level management flows
├── navigation/         # Navigation specifications
├── rust-core/          # Rust core engine specs
├── tasks/              # PRDs and architecture docs
└── wasm-tsrs-integration/ # WASM integration specs
```

## Standalone Build Status

**gsnake-specs is standalone-ready.**

This repository:
- ✅ Contains only Markdown documentation files
- ✅ Has no build dependencies on sibling repositories
- ✅ Can be cloned and read independently
- ✅ Requires no build tools (pure documentation)

### Documentation References

Some documentation files reference sibling repositories (e.g., `../gsnake-core`) in the context of:
- Describing the overall architecture
- Explaining build processes for other submodules
- Documenting dependency resolution strategies

These are **documentation references** (describing how other parts of the system work), not **build dependencies** (required to build this repository).

## Usage

### Viewing Documentation

All documentation is in Markdown format and can be:
- Read directly on GitHub
- Viewed in any Markdown editor
- Browsed locally with any text editor

### Key Documentation Files

- [Repository Architecture](tasks/repo-architecture.md) - Central documentation for dependency management
- [Standalone Submodules Build PRD](tasks/prd-standalone-submodules-build.md) - PRD for standalone build strategy
- [CI/CD Workflows PRD](tasks/prd-ci-tests-and-workflows.md) - CI/CD strategy documentation

## Contributing

When adding new documentation:
1. Use relative paths for internal links (within gsnake-specs)
2. Use absolute GitHub URLs for external references
3. Ensure Markdown syntax is valid
4. Keep documentation up-to-date with code changes

## No Build Required

Unlike other gSnake submodules, gsnake-specs:
- Has no npm install step
- Has no cargo build step
- Has no compilation or bundling process
- Is immediately usable after clone

This is intentional - documentation should be accessible without build tools.
