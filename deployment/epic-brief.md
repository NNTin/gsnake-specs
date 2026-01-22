## Summary

This Epic establishes a public deployment pipeline for gSnake to GitHub Pages, making the game instantly accessible to anyone with a web browser. The deployment supports two modes: versioned releases for stable gameplay experiences and a development branch for testing. Each tagged release creates a permanent, versioned deployment that showcases the game's evolution over time, while the main branch provides a live testing environment. The root URL automatically redirects users to the latest stable version, and a versions index page allows exploration of the game's development history.

## Context & Problem

### Who's Affected

**Primary Users**: Friends, family, and community members who want to play gSnake but lack technical expertise to clone repositories, install dependencies, and run local development servers.

**Secondary User**: The developer (project owner) who needs a convenient testing environment accessible from any device without local setup.

### Current State

gSnake currently exists only as source code in a GitHub repository at `github.com/nntin/gSnake`. To play the game, users must:

1. Clone the repository
2. Install Node.js and dependencies
3. Run a local development server
4. Navigate to localhost in their browser

This technical barrier prevents non-developers from accessing and enjoying the game.

### The Problem

**Accessibility Barrier**: The game cannot be easily shared with friends, family, or community members. Sending them a GitHub repository link requires them to have development tools and knowledge, which most casual players don't possess.

**No Permanent References**: Without deployed versions, there's no way to reference specific states of the game. As development continues, previous iterations are lost to history unless someone manually checks out old commits.

**Testing Friction**: Testing the game on different devices or browsers requires setting up the development environment on each device, creating unnecessary friction for quick validation.

**Evolution Storytelling**: The game's development journey—how features evolved, how gameplay changed, what experiments were tried—cannot be easily demonstrated or shared with others.

### Desired Outcome

Enable instant, zero-setup access to gSnake through simple URLs that anyone can visit in a browser. Preserve the game's development history through versioned deployments that remain accessible indefinitely, creating a living archive of the game's evolution. Provide the developer with a frictionless testing environment that works across all devices.

### Success Criteria

**Primary Success Metric**: Friends, family, and community members can access and play gSnake by simply visiting a URL, without requiring any technical knowledge, development tools, or assistance.

**Validation**:

- Share the root URL (`nntin.github.io/gSnake`) with non-technical users
- Users can successfully access and play the game without asking for help
- Users can navigate between different versions if desired
- Developer can test on any device by visiting the `/main` URL

### Version Format Requirements

All versioned releases must follow semantic versioning with 'v' prefix:

- Format: `v{major}.{minor}.{patch}` (e.g., v0.1.0, v1.2.3, v2.0.0)
- Git tags must match this format exactly
- URLs will use the same format: `nntin.github.io/gSnake/v0.1.0`

### Out of Scope

The following features are explicitly excluded from this Epic to maintain focused scope:

- **Analytics and Usage Tracking**: No visitor analytics, play statistics, or usage metrics
- **Custom Domain**: Deployment uses GitHub Pages default domain only (no custom domain setup)
- **Deployment Rollback**: No ability to rollback or unpublish a deployed version
- **Version Deletion**: Once deployed, versions remain indefinitely (no deletion mechanism)
- **Changelog Generation**: No automatic changelog or release notes generation
- **Deployment Notifications**: No custom notifications beyond GitHub's standard workflow notifications
- **Version Comparison**: No UI for comparing differences between versions
- **Access Control**: All deployments are public (no authentication or private versions)
- **Performance Monitoring**: No performance metrics or monitoring for deployed versions

