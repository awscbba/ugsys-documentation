# ADR-0004: Git Workflow and Branch Protection

**Date**: 2026-02-23
**Status**: Accepted
**Deciders**: AWSUGCBBA Platform Team

## Context

With 7 repositories and multiple contributors, a consistent Git workflow prevents accidental breakage of `main` and ensures code quality gates are enforced.

## Decision

### Branch Strategy

- `main` — protected, always deployable
- `feature/<description>` — all work happens here
- No long-lived `develop` or `staging` branches

### Branch Protection Rules (all 7 repos)

- Require PR before merging to `main`
- Require at least 1 approving review
- Require status checks to pass (CI lint + test)
- No direct pushes to `main` (enforced by branch protection AND local pre-commit hook)
- No force pushes

### Local Git Hooks

Every repo ships hooks in `scripts/hooks/` installed via `just install-hooks`:

**pre-commit**: runs ruff lint + format check on staged files, blocks commits directly to `main`

**pre-push**: runs full test suite, blocks pushes directly to `main`

Hooks use `uv tool run ruff` (not the project venv) to avoid environment resolution issues.

### Commit Convention

Conventional Commits format:
- `feat:` new feature
- `fix:` bug fix
- `chore:` maintenance
- `docs:` documentation only
- `refactor:` code restructure without behavior change
- `test:` test additions/changes

### Release Tagging (shared-libs)

Tag format: `<package>/v<semver>` — e.g. `auth-client/v0.2.0`

```bash
just release auth-client 0.2.0
```

## Consequences

- No one can accidentally push broken code to `main`
- CI is the source of truth for green/red status
- Local hooks catch issues before they reach CI, saving time
- Conventional commits enable automated changelog generation in the future
