# ADR-0002: Shared Library Distribution via uv Git Sources

**Date**: 2026-02-23
**Status**: Accepted
**Deciders**: AWSUGCBBA Platform Team

## Context

The `ugsys-shared-libs` repo contains 4 Python packages used by all services:

- `ugsys-auth-client` — JWT validation, FastAPI middleware, service-to-service auth
- `ugsys-logging-lib` — structlog + correlation ID middleware
- `ugsys-event-lib` — EventBridge publisher with standard event envelope
- `ugsys-testing-lib` — pytest fixtures, factories, mocks

These packages live in a separate GitHub repo from the services that consume them. A distribution mechanism is needed.

## Decision

Distribute packages as **uv git sources** pinned to version tags.

Consumer repos reference packages in `pyproject.toml`:

```toml
[project]
dependencies = [
    "ugsys-auth-client",
    "ugsys-logging-lib",
    "ugsys-event-lib",
]

[tool.uv.sources]
ugsys-auth-client = { git = "https://github.com/awscbba/ugsys-shared-libs", tag = "auth-client/v0.1.0", subdirectory = "auth-client" }
ugsys-logging-lib  = { git = "https://github.com/awscbba/ugsys-shared-libs", tag = "logging-lib/v0.1.0",  subdirectory = "logging-lib" }
ugsys-event-lib    = { git = "https://github.com/awscbba/ugsys-shared-libs", tag = "event-lib/v0.1.0",    subdirectory = "event-lib" }
```

Tag convention: `<package-name>/v<semver>` (e.g. `auth-client/v0.2.0`)

Releasing a new version:
```bash
just release auth-client 0.2.0   # tags + pushes, triggers CI + GitHub Release
```

## Alternatives Considered

### GitHub Packages (PyPI registry)
Rejected — GitHub Packages does not support a PyPI-compatible package index. Twine uploads return `404 Not Found`. This is a known platform limitation (see [community discussion #8542](https://github.com/orgs/community/discussions/8542)).

### Public PyPI
Rejected — packages contain internal business logic and should remain private.

### Self-hosted pypi-server
Deferred — adds operational overhead. Can be revisited if the number of packages grows significantly.

## Consequences

- No registry infrastructure to maintain
- Version pinning is explicit and auditable in `uv.lock`
- GitHub Actions in consumer repos need read access to `ugsys-shared-libs` (handled via `GITHUB_TOKEN` for same-org repos, or a deploy key for cross-org)
- Each package release creates a tagged GitHub Release with built wheel attached for reference
