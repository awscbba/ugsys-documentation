# CI/CD Pipeline Guide

This guide covers the CI and deployment pipeline for `ugsys-*` services. All services follow the same structure.

## Overview

Each service has four GitHub Actions workflows:

| File | Trigger | Purpose |
|---|---|---|
| `ci.yml` | PR → main, push non-main | BFF/backend quality gates |
| `ci-frontend.yml` | PR → main, push non-main | Frontend quality gates (admin-panel only) |
| `deploy-bff.yml` | Push to main (src changes) | Build Lambda image → push ECR → update Lambda |
| `deploy-frontend.yml` | Push to main (frontend changes) | Build SPA → sync S3 → invalidate CloudFront |
| `codeql.yml` | PR → main, push main | Static security analysis |

## CI Pipeline (BFF)

Stages run in order; each blocks the next:

1. **Lint & Format** — `ruff check` + `ruff format --check`
2. **Type Check** — `mypy --strict`
3. **Tests & Coverage** — `pytest` with 80% coverage gate
4. **SAST (Bandit)** — Python security linting
5. **SAST (Semgrep)** — OWASP, secrets, security-audit rules
6. **SBOM + CVE Scan** — CycloneDX → Trivy (CRITICAL/HIGH, unfixed only)
7. **Secret Scan** — TruffleHog (verified secrets only, advisory)
8. **Architecture Guard** — hexagonal layer import rules

## CI Pipeline (Frontend — admin-panel)

1. **Lint & Format** — ESLint + Prettier
2. **Type Check** — `tsc --noEmit`
3. **Tests & Coverage** — vitest with 80% coverage gate
4. **SAST (Semgrep)** — TypeScript, React, OWASP rules
5. **Dependency CVEs** — `npm audit --audit-level=high`
6. **Secret Scan** — TruffleHog (advisory)
7. **Build** — `vite build`
8. **Bundle Analysis** — size report (advisory)

## Deploy Pipeline (BFF)

Triggered on push to `main` when any of these paths change:
- `src/**`, `Dockerfile.lambda`, `pyproject.toml`, `uv.lock`

Steps:
1. **Build** — `docker build -f Dockerfile.lambda`, push to ECR with commit SHA tag + `latest`
2. **Deploy** — `aws lambda update-function-code --image-uri <ecr-uri>`, waits for update
3. **Notify** — Slack message (success or failure)

### Lambda Image Packaging

See [ADR-0005](../adr/0005-lambda-container-image-packaging.md) for the required Dockerfile pattern.
The short version: always use `pip install --target ${LAMBDA_TASK_ROOT}`, never `uv sync`.

## Deploy Pipeline (Frontend)

Triggered on push to `main` when `admin-shell/**` changes.

Steps:
1. **Build** — `npm run build` (Vite SPA)
2. **Deploy**:
   - Hashed assets (`dist/assets/`) → S3 with `Cache-Control: public, max-age=31536000, immutable`
   - Everything else → S3 with `Cache-Control: no-cache, no-store, must-revalidate`
   - CloudFront invalidation `/*`
3. **Notify** — Slack message

## Secrets and Environment Variables

### GitHub Secrets

Secrets must be set at **both** the repo level AND the `prod` environment level.
Jobs that declare `environment: prod` only see environment-scoped secrets — repo-level secrets are not injected.

| Secret | Scope | Value |
|---|---|---|
| `AWS_ROLE_ARN` | repo + prod env | OIDC role ARN for GitHub Actions |
| `ADMIN_SHELL_BUCKET` | repo + prod env | S3 bucket name for the SPA |
| `CLOUDFRONT_DISTRIBUTION_ID` | repo + prod env | CloudFront distribution ID |
| `SLACK_BOT_TOKEN` | org-level | Slack bot token |
| `SLACK_CHANNEL_ID` | org-level | Slack channel for notifications |

To set a secret on the `prod` environment:
```bash
printf 'value' | gh secret set SECRET_NAME --repo awscbba/<repo> --env prod
```

### Lambda Environment Variables

Set directly on the Lambda function (not via GitHub secrets):

| Variable | Description |
|---|---|
| `JWT_PUBLIC_KEY` | RS256 public key PEM for JWT validation |
| `JWT_AUDIENCE` | Expected `aud` claim (e.g. `admin-panel`) |
| `JWT_ISSUER` | Expected `iss` claim (e.g. `ugsys-identity-manager`) |
| `IDENTITY_MANAGER_BASE_URL` | Base URL of the Identity Manager service |
| `USER_PROFILE_SERVICE_BASE_URL` | Base URL of the User Profile Service |
| `SERVICE_REGISTRY_TABLE_NAME` | DynamoDB table for service registry |
| `AUDIT_LOG_TABLE_NAME` | DynamoDB table for audit logs |
| `EVENT_BUS_NAME` | EventBridge bus name |

## Branch Protection

`main` requires:
- 1 approving review
- All required status checks passing (10 checks for admin-panel)
- No direct pushes

See [ADR-0004](../adr/0004-git-workflow.md) for the full Git workflow.

## Troubleshooting

### Lambda returns 500 on every request
Check CloudWatch logs first:
```bash
aws logs filter-log-events \
  --log-group-name "/aws/lambda/<function-name>" \
  --region us-east-1 \
  --start-time $(($(date +%s) * 1000 - 3600000)) \
  --limit 20 \
  --query 'events[].message' --output json
```

Common causes:
- `No module named 'mangum'` — Dockerfile uses `uv sync` instead of `pip install --target`. See ADR-0005.
- Missing env var — check Lambda configuration for required variables above.

### Frontend deploy succeeds but S3 bucket name is empty
The `deploy` job uses `environment: prod`. Secrets must be set on the `prod` environment, not just at repo level. See the Secrets section above.

### DNS_PROBE_FINISHED_NXDOMAIN on your machine but site works elsewhere
Your router or local DNS resolver has a stale negative cache entry. Fix:
```bash
# Override DNS on macOS to bypass the router
networksetup -setdnsservers Wi-Fi 8.8.8.8 1.1.1.1
```
Or flush the OS DNS cache:
```bash
sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder
```
