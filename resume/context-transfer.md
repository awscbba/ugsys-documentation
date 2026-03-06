# Context Transfer — ugsys-admin-panel Development

**Date**: 2026-03-06
**From**: Previous dev session (macOS)
**To**: New device

---

## Repos and Org

- GitHub org: `awscbba`
- Repos: `ugsys-admin-panel`, `ugsys-documentation`, `ugsys-platform-infrastructure`, `ugsys-projects-registry`
- DNS domain: `apps.cloud.org.bo` — NEVER use `cbba.cloud.org.bo`
- Admin panel frontend URL: `https://admin.apps.cloud.org.bo`
- Lambda function name: `ugsys-admin-panel-prod`, region: `us-east-1`

---

## Tooling Rules

- Python package manager: `uv` — never bare `pip install` for project deps
- `pip install --target` is only acceptable inside `Dockerfile.lambda`
- GitHub CLI: use `/usr/local/bin/gh` (not `gh` — it's aliased to `git push` in the shell)
- AWS CLI commands: run with `cwd: ugsys-platform-infrastructure` so direnv loads credentials
- Long-running commands (`npm run dev`, `uv sync` interactively, etc.) must NOT be run — tell user to run manually
- All CLI operations go through the justfile where possible
- `prod` GitHub environment has secrets set separately from repo-level secrets

---

## Lambda Packaging Pattern (ADR-0006)

Every `Dockerfile.lambda` must use this pattern — `uv sync` silently breaks Lambda:

```dockerfile
RUN uv export --no-dev --no-hashes --frozen -o /tmp/requirements.txt \
    && pip install -r /tmp/requirements.txt --target "${LAMBDA_TASK_ROOT}" --quiet
```

---

## Completed Tasks (all done, merged to main)

| # | Task | PR | Notes |
|---|------|----|-------|
| 1 | Fix CI pipeline failures | PR merged | PyJWT migration, CodeQL v4, job name fixes |
| 2 | Split deploy into BFF + frontend workflows | PR merged | `deploy-bff.yml`, `deploy-frontend.yml` |
| 3 | Fix BFF 500 `No module named 'mangum'` + frontend TypeError | PR #6 merged | Dockerfile.lambda packaging, HttpClient.ts, TopBar.tsx |
| 4 | Fix `ulid.new()` → `ULID()` (python-ulid v3 API) | PR #7 merged | `dynamodb_audit_log_repository.py` |
| 5 | Document Lambda packaging pattern + CI/CD guide | Merged to main | `ugsys-documentation` repo |
| 6 | Fix DNS `*.cbba.cloud.org.bo` → `*.apps.cloud.org.bo` + ASCII → Mermaid diagrams | PR #2 merged | `platform-contract.md` |
| 7 | Update ADR-0005, renumber ADR-0006, rewrite contract Section 14 | PR #3 merged | See below |

---

## Task 7 Detail — Documentation Updates (PR #3, `ugsys-documentation`)

### ADR-0005 (`adr/0005-admin-panel-plugin-architecture.md`) — rewritten
- Frontend: Plugin Manifest system (not FSD monorepo slices)
- Section 2: hybrid seed + runtime API registration (not push-at-startup)
- Section 3: SameSite=Lax (matches actual `auth.py` code — was incorrectly `Strict`)
- Architecture Flow: Mermaid diagrams for BFF startup, login, proxy flow
- BFF API Surface table: all actual endpoints from implementation
- Alternatives Considered: fixed (was corrupted/truncated from previous session)
- References: ADR-0006 for Lambda packaging

### ADR-0006 (`adr/0006-lambda-container-image-packaging.md`) — renumbered
- Was `0005-lambda-container-image-packaging.md` (numbering conflict with ADR-0005)
- Renamed via `git mv`, internal heading updated from ADR-0005 → ADR-0006

### `specs/platform-contract.md` Section 14 — rewritten
- 14.1: Plugin Manifest system overview with Mermaid diagram
- 14.2: Hybrid seed registration contract (seed JSON + runtime API) — removed push-at-startup to identity-manager
- 14.3: BFF registry endpoints (was incorrectly pointing to identity-manager endpoints)
- 14.4: Actual repo structure (`admin-shell/` + `src/`)
- 14.5: Correct auth flow — SameSite=Lax, actual `/api/v1/` paths, Mermaid login + proxy sequence diagrams
- 14.6: RBAC table with actual roles (`super_admin`, `admin`, `moderator`, `auditor`, `member`/`guest`/`system` → 403)
- 14.7: BFF startup sequence Mermaid diagram (seed_loader → health polling)
- 14.8: Full BFF API surface table (all 16 endpoints)
- 14.9: Gap tracker updated with actual implementation status

---

## Current State of `ugsys-admin-panel`

### BFF (Python/FastAPI Lambda) — COMPLETE
All tasks 1–7 and 13–14 from the spec are done. The BFF is deployed and running.

Key files:
- `src/main.py` — FastAPI app, full middleware stack, DI wiring, seed loader, health polling
- `src/presentation/api/v1/auth.py` — login/logout/refresh/me, httpOnly SameSite=Lax cookies
- `src/presentation/api/v1/registry.py` — service registration, list, deregister, config-schema
- `src/presentation/api/v1/proxy.py` — `ANY /api/v1/proxy/{service_name}/{path:path}`
- `src/presentation/api/v1/config.py` — `POST /api/v1/proxy/{service_name}/config`
- `src/presentation/api/v1/health.py` — aggregated health status
- `src/presentation/api/v1/users.py` — user list, role change, status change
- `src/presentation/api/v1/audit.py` — audit log query
- `src/infrastructure/seed/seed_loader.py` — reads `config/seed_services.json`, upserts DynamoDB
- `src/infrastructure/adapters/in_memory_circuit_breaker.py` — CLOSED/OPEN/HALF_OPEN states
- `src/infrastructure/adapters/identity_manager_client.py` — HTTP adapter for identity-manager
- `Dockerfile.lambda` — uses ADR-0006 pattern (`uv export` + `pip install --target`)
- `main.py` — Lambda handler entry point (Mangum wrapper)

### Admin Shell (React/TypeScript SPA) — COMPLETE (tasks 8–12)
Deployed to `https://admin.apps.cloud.org.bo`.

Key files:
- `admin-shell/src/infrastructure/http/HttpClient.ts` — singleton, auto-refresh on 401, CSRF header
- `admin-shell/src/presentation/components/layout/TopBar.tsx` — displayName guard fixed
- `admin-shell/src/presentation/components/layout/AppShell.tsx`
- `admin-shell/src/presentation/components/layout/Sidebar.tsx`
- `admin-shell/src/presentation/components/ErrorBoundary.tsx`
- `admin-shell/src/presentation/components/SessionMonitor.tsx`
- `admin-shell/src/presentation/components/views/HealthDashboard.tsx`
- `admin-shell/src/presentation/components/views/UserManagement.tsx`
- `admin-shell/src/presentation/components/views/AuditLog.tsx`

### CI/CD Workflows
- `.github/workflows/ci.yml` — BFF: ruff, mypy, pytest, bandit, semgrep, trivy, gitleaks
- `.github/workflows/ci-frontend.yml` — Shell: eslint, tsc, vitest, semgrep, npm audit
- `.github/workflows/codeql.yml` — weekly + on PR, Python security-extended
- `.github/workflows/deploy-bff.yml` — builds Docker image, pushes to ECR, updates Lambda
- `.github/workflows/deploy-frontend.yml` — builds Vite SPA, deploys to S3/CloudFront

---

## Spec Files Location

```
ugsys-admin-panel/.kiro/specs/admin-panel/
├── requirements.md   — 13 requirements, all implemented
├── design.md         — architecture, 41 correctness properties
└── tasks.md          — 15 task groups, all required tasks [x] complete
```

### Optional tasks still pending (marked `*` in tasks.md — can skip for MVP)
All property-based tests (hypothesis/fast-check) and integration tests are optional (`*`).
Required tasks are 100% complete.

---

## BFF API Surface (actual endpoints)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/auth/login` | None | Authenticate, set httpOnly cookies |
| POST | `/api/v1/auth/logout` | Cookie | Clear cookies, call identity-manager logout |
| POST | `/api/v1/auth/refresh` | Cookie | Transparent token refresh |
| GET | `/api/v1/auth/me` | Cookie | Current user info (JWT + profile enrichment) |
| POST | `/api/v1/registry/services` | S2S or super_admin | Register/update service |
| GET | `/api/v1/registry/services` | admin+ | List services (role-filtered) |
| DELETE | `/api/v1/registry/services/{name}` | super_admin | Deregister service |
| GET | `/api/v1/registry/services/{name}/config-schema` | admin+ | Get config JSON Schema |
| ANY | `/api/v1/proxy/{service}/{path}` | admin+ | Forward to downstream service |
| GET | `/api/v1/health/services` | admin+ | Aggregated health status |
| GET | `/api/v1/users` | admin+ | Paginated user list (enriched) |
| PATCH | `/api/v1/users/{id}/roles` | super_admin | Change user roles |
| PATCH | `/api/v1/users/{id}/status` | admin+ | Activate/deactivate user |
| GET | `/api/v1/audit/logs` | auditor+ | Paginated audit log |
| GET | `/health` | None | BFF own health check |
| POST | `/internal/events` | Infra-level | Receive EventBridge events |

---

## Key Dependencies (`pyproject.toml`)

- `fastapi`, `mangum` — Lambda ASGI handler
- `python-ulid` — v3 API: `ULID()` not `ulid.new()`
- `python-jose[cryptography]` — RS256 JWT validation
- `structlog` — structured logging
- `boto3` — DynamoDB, EventBridge
- `httpx` — async HTTP client for downstream calls
- `ugsys-shared-libs` — git source dependency (requires `--no-hashes` in Dockerfile)

---

## What to Work on Next

The BFF and Admin Shell are functionally complete and deployed. Likely next steps:

1. **Connect real services** — add actual platform services to `config/seed_services.json` as they come online (projects-registry, identity-manager, etc.)
2. **Plugin Manifest endpoints** — each service needs to expose `/api/v1/admin/manifest.json`
3. **Optional property tests** — hypothesis (BFF) and fast-check (frontend) tests marked `*` in tasks.md
4. **Integration tests** — moto-based DynamoDB tests and FastAPI TestClient end-to-end tests (tasks 14.3, 14.4)
5. **super_admin bootstrap** — first admin creation flow (gap tracker item)

---

## Documentation Repo (`ugsys-documentation`) — Current State

All on `main` branch:
- `adr/0005-admin-panel-plugin-architecture.md` — accurate, reflects implementation
- `adr/0006-lambda-container-image-packaging.md` — canonical Lambda packaging pattern
- `specs/platform-contract.md` — Section 14 updated, all DNS fixed, Mermaid diagrams
- `guides/ci-cd-pipeline.md` — CI/CD pipeline documentation
