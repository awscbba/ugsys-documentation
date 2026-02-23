# Local Development Guide

## Prerequisites

- [devbox](https://www.jetify.com/devbox) — manages Python, uv, just, AWS CLI, gh
- [uv](https://docs.astral.sh/uv/) — Python package manager (installed by devbox)
- [just](https://just.systems/) — task runner (installed by devbox)
- AWS CLI configured with `awscbba` profile

## Getting Started (any repo)

```bash
# 1. Clone and enter the repo
git clone git@github.com:awscbba/<repo>.git
cd <repo>

# 2. Start devbox shell (installs all tools)
devbox shell

# 3. Install git hooks
just install-hooks

# 4. Sync dependencies
uv sync --extra dev

# 5. Run tests
just test
```

## ugsys-shared-libs

```bash
# Lint all packages
just lint

# Run tests for a specific package
just test-pkg auth-client

# Release a new version (tags + triggers CI publish)
just release auth-client 0.2.0
```

## ugsys-platform-infrastructure

```bash
# Synthesize CDK templates
just synth

# Deploy all stacks
just deploy

# Diff against deployed state
just diff
```

## Consuming shared-libs in a service

Add to `pyproject.toml`:

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

Then run `uv sync`.
