# Git Hooks for gsnake-specs

This directory contains shared git hooks for the gsnake-specs repository.

## Enabling Hooks

To enable these hooks, run from the gsnake-specs directory:

```bash
git config core.hooksPath .github/hooks
```

## Verification

Verify that hooks are enabled:

```bash
git config core.hooksPath
```

This should output: `.github/hooks`

## Disabling Hooks

To disable the hooks and revert to default behavior:

```bash
git config --unset core.hooksPath
```

## Available Hooks

### pre-commit

The pre-commit hook runs the following checks:

1. **Format Check**: `mdformat --check .`
2. **Lint Check**: `pymarkdown scan .`

**Note: Tests are NOT run in pre-commit hooks for speed.** This hook focuses on Markdown formatting and linting only.
