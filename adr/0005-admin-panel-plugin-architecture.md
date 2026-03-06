# ADR-0005: Admin Panel Plugin Architecture

**Date**: 2026-02-24
**Status**: Accepted
**Deciders**: AWSUGCBBA Platform Team

## Context

The platform has 5 independently deployed microservices. Each service has its own admin surface
(user management, config, RBAC, dashboards). We need a unified admin UI so operators don't have
to navigate 5 separate interfaces.

### Constraints

- Single small team — not multiple teams shipping independently
- Phase 4 work — admin panel is the last service to be built
- Security is non-negotiable — admin panel has elevated privileges
- Services are independently deployed Lambdas behind API Gateway
- All services already validate RS256 JWTs via `ugsys-auth-client`

### What the admin panel must do

- Provide a single login and unified nav for all services
- Let each service own its full admin UI (not just generic CRUD forms)
- Allow new services to plug in without changing the panel's code
- Keep auth and RBAC centralized in identity-manager
- Let operators configure service-specific parameters per service

---

## Decision

### 1. Frontend architecture — React SPA (monorepo, not micro-frontends)

**Decision**: The admin panel is a single React + Vite SPA organized as a monorepo with
Feature-Sliced Design (FSD) internal boundaries. Each service's admin UI is a *slice* inside
the monorepo, not a separately deployed remote module.

**Why not micro-frontends (Module Federation)**:

The research is clear: MFEs are worth the complexity when the bottleneck is *organizational
scale* — many teams shipping independently. We have one team. The costs of MFE are real:

- Remote entry waterfall: N services × N network hops before first render
- Version skew: compatibility matrix grows as O(N²) across service versions
- Build complexity: each service needs a federation-aware build pipeline
- XSS surface: passing JWT as a prop to dynamically loaded remote code is a security risk —
  any compromised remote module gets the token
- Vite Module Federation is still maturing; Webpack 5 MF is production-proven but adds
  significant build overhead

FSD inside a monorepo gives us the same domain boundaries and independent ownership without
the distributed deployment complexity. When the team grows to multiple independent teams,
extracting slices into true MFEs is straightforward because the contracts are already clean.

**Fallback trigger**: If a service team genuinely needs independent frontend deployment cadence
(e.g. a third-party contributor), that slice can be extracted to a true MFE at that point.
The FSD public API contract makes extraction safe.

### 2. Service plugin registration — push at startup (backend only)

Every `ugsys-*` service registers itself with identity-manager at startup via S2S token.
The registration declares:

- Service identity and display metadata
- Config schema (JSON Schema) — what parameters the operator can configure
- Service-specific roles — what RBAC roles this service defines

The admin panel frontend fetches the service registry from identity-manager on load and
renders the appropriate nav entries and config screens. No frontend remote loading.

### 3. Auth — HttpOnly cookie, never JS-accessible token

**Decision**: The admin panel authenticates via identity-manager. The JWT is stored in an
HttpOnly, Secure, SameSite=Strict cookie — never in localStorage, sessionStorage, or a
React prop/context.

**Why**: Passing a JWT as a React prop or storing it in JS-accessible memory means any XSS
vulnerability (including in a dynamically loaded remote module) can exfiltrate the token.
HttpOnly cookies are inaccessible to JavaScript by design. The browser sends the cookie
automatically on every same-origin request.

For cross-origin requests to service APIs (different subdomains), the panel uses a
**BFF (Backend for Frontend)** proxy endpoint on the admin panel's own API Gateway. The
panel backend reads the HttpOnly cookie, validates it, and forwards the request to the
target service with the JWT in the `Authorization` header. The frontend never touches the
raw token.

```
Browser (admin panel SPA)
  → POST /api/v1/proxy/projects/admin/users   (admin panel BFF)
    Cookie: session=<HttpOnly JWT>
  → admin panel BFF validates cookie, extracts JWT
  → GET https://api.cbba.cloud.org.bo/projects/api/v1/admin/users
    Authorization: Bearer <JWT>
  → projects-registry validates JWT via ugsys-auth-client
```

### 4. Service config — stored and served by identity-manager

Each service registers a JSON Schema describing its configurable parameters. Identity-manager
stores both the schema and the operator-set values. Services fetch their own config from
identity-manager at startup (after registering), overriding env var defaults.

This makes identity-manager the single source of truth for:
- Which services exist on the platform
- What roles/permissions each service defines
- What configurable options each service exposes and their current values

### 5. RBAC — centralized in identity-manager, service roles declared at registration

Service-specific roles (e.g. `projects:admin`) are declared by each service at registration
and stored in identity-manager. The admin panel manages all role assignments through
identity-manager's RBAC endpoints. Services never manage their own role assignments.

---

## Architecture Flow

```
Service cold start
  → POST /api/v1/services/register  (identity-manager, S2S token)
    { service_id, config_schema, roles[] }
  → GET  /api/v1/services/{id}/config  (identity-manager, S2S token)
    → apply operator-set config values over env defaults

Admin opens panel (browser)
  → GET /api/v1/auth/login  (identity-manager)
  → JWT stored in HttpOnly cookie by admin panel BFF
  → GET /api/v1/registry/services  (admin panel BFF → identity-manager)
  → React SPA renders nav from service list

Admin configures a service
  → React SPA calls admin panel BFF proxy endpoint
  → BFF reads HttpOnly cookie, forwards JWT to target service
  → Target service validates JWT via ugsys-auth-client

Admin manages users/RBAC
  → All calls go to identity-manager via BFF proxy
  → Service-specific roles visible because registered at startup
```

---

## Consequences

**Positive**:
- No XSS token exfiltration risk — JWT never in JS memory
- No remote entry waterfall — single SPA bundle, fast initial load
- No version skew problem — all admin UI ships together
- Adding a new service = register at startup + add a FSD slice to the monorepo
- FSD boundaries make future MFE extraction safe if team scales
- BFF proxy is a natural place to add audit logging for all admin actions

**Negative / accepted trade-offs**:
- Admin panel frontend must be updated when a new service adds admin UI
  (mitigated: same team, monorepo, low coordination cost)
- BFF proxy adds one network hop for cross-service admin calls
  (mitigated: admin panel is low-traffic, latency is acceptable)
- HttpOnly cookie requires CSRF protection (mitigated: SameSite=Strict + CSRF token)

---

## Alternatives Considered

- **Module Federation (micro-frontends)**: Rejected for Phase 4. Complexity cost exceeds
  benefit for a single team. Revisit if team grows to 3+ independent squads.
- **JWT as React prop / context**: Rejected — XSS risk. Any compromised dependency or
  dynamic remote gets the token.
- **localStorage for JWT**: Rejected — industry consensus is clear: XSS = game over.
- **Pull config (static service list in panel)**: Rejected — requires panel code changes
  every time a service is added.
- **Generic form rendering only (no custom UI per service)**: Available as per-service
  fallback for simple services, not the primary approach.

---

## References

- `specs/platform-contract.md` Section 14 — Admin Panel Plugin Contract
- ADR-0001 — Microservices Architecture
- [Micro-Frontends: Are They Still Worth It in 2025?](https://feature-sliced.design/blog/micro-frontend-architecture)
- [Stop Storing JWT in LocalStorage: HttpOnly Cookie Strategy](https://openillumi.com/en/en-react-rest-jwt-httponly-security/)
