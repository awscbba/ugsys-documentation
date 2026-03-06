# ADR-0005: Lambda Container Image Packaging with uv

**Date**: 2026-03-06
**Status**: Accepted
**Deciders**: AWSUGCBBA Platform Team

## Context

All `ugsys-*` services deploy as AWS Lambda container images (not zip packages). Each service uses `uv` for dependency management. The Lambda base image (`public.ecr.aws/lambda/python:3.13`) uses a non-standard Python interpreter at `/var/lang/bin/python3`, not `/usr/bin/python3`.

When `uv sync` is used inside the Dockerfile it installs packages into a `.venv/` directory. The Lambda runtime does not add `.venv/` to `sys.path`, so all imports fail at cold start with `Runtime.ImportModuleError: No module named '<package>'`.

## Decision

Use the following two-step pattern in every `Dockerfile.lambda`:

```dockerfile
# 1. Export locked, pinned requirements (no hashes — incompatible with git sources)
RUN uv export --no-dev --no-hashes --frozen -o /tmp/requirements.txt \
    && pip install -r /tmp/requirements.txt --target "${LAMBDA_TASK_ROOT}" --quiet
```

Key points:
- `uv export` resolves the lockfile to a flat `requirements.txt` — no venv involved
- `pip install --target ${LAMBDA_TASK_ROOT}` installs directly into `/var/task/`, which is always on `sys.path` in the Lambda runtime
- `--no-hashes` is required when the lockfile contains git-sourced packages (e.g. `ugsys-shared-libs`)
- `--frozen` ensures the lockfile is not modified during the build

## Alternatives Considered

| Approach | Why rejected |
|---|---|
| `uv sync --no-dev` | Installs into `.venv/` — invisible to Lambda runtime |
| `UV_SYSTEM_PYTHON=1 uv sync` | Installs into the wrong system Python (`/usr/bin/python3`), not `/var/lang/bin/python3` |
| `uv pip install --python /var/lang/bin/python3` | Works but fragile — depends on the internal path of the base image |
| Zip deployment | Rejected in ADR-0001 (serverless-first, container images preferred) |

## Consequences

- All services must use this pattern — `uv sync` in a Dockerfile targeting Lambda will silently produce a broken image
- The `ugsys-projects-registry` repo established this pattern first; all other services must follow it
- Build times are slightly longer than `uv sync` because `pip` is slower than `uv`, but the difference is negligible for Lambda image sizes

## References

- `ugsys-projects-registry/Dockerfile.lambda` — canonical reference implementation
- `ugsys-admin-panel/Dockerfile.lambda` — fixed in PR #6 (2026-03-06)
