# ugsys Platform — Exhaustive Service Contract

**Version**: 1.0.0
**Status**: Authoritative — all implementation must match this document
**Last updated**: 2026-02-24
**Source of truth**: Derived from Registry (registry-api), devsecops-poc, ugsys-identity-manager, ugsys-user-profile-service

> This is the single source of truth for all 6 ugsys microservices.
> No endpoint, field, event, or business rule should be implemented that is not here.
> No feature that IS here should be skipped.

---

## Table of Contents

1. [Platform Overview](#1-platform-overview)
2. [Shared Conventions](#2-shared-conventions)
3. [Identity Manager](#3-identity-manager)
4. [User Profile Service](#4-user-profile-service)
5. [Projects Registry](#5-projects-registry)
6. [Omnichannel Service](#6-omnichannel-service)
7. [Mass Messaging](#7-mass-messaging)
8. [EventBridge Event Catalog](#8-eventbridge-event-catalog)
9. [Cross-Cutting Concerns](#9-cross-cutting-concerns)
10. [Implementation Gap Tracker](#10-implementation-gap-tracker)
11. [Phase Roadmap](#11-phase-roadmap)
12. [Verified Infrastructure & Shared Libs Reference](#12-verified-infrastructure--shared-libs-reference)
13. [Registry Reference Appendix](#section-13--registry-reference-appendix)
14. [Admin Panel Plugin Architecture](#section-14--admin-panel-plugin-architecture)

---

## 1. Platform Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         API Gateway (per service)                            │
└──┬──────────┬──────────┬──────────┬──────────┬──────────┬───────────────────┘
   │          │          │          │          │          │
┌──▼───┐ ┌───▼───┐ ┌────▼───┐ ┌────▼──────┐ ┌▼────────┐ ┌▼──────────┐
│iden- │ │user-  │ │proj-   │ │omni-      │ │mass-    │ │admin-     │
│tity  │ │profile│ │ects    │ │channel    │ │messaging│ │panel (BFF)│
└──┬───┘ └───┬───┘ └────┬───┘ └────┬──────┘ └┬────────┘ └┬──────────┘
   │          │          │          │          │           │
   └──────────┴──────────┴──────────┴──────────┴───────────┘
                              │
                    EventBridge Bus (ugsys-platform-bus)
```

| Service | Repo | Status |
|---------|------|--------|
| Identity Manager | `ugsys-identity-manager` | Phase 1 — partial |
| User Profile | `ugsys-user-profile-service` | Phase 1 — partial |
| Projects Registry | `ugsys-projects-registry` | Phase 2 — pending |
| Omnichannel | `ugsys-omnichannel-service` | Phase 3 — pending |
| Mass Messaging | `ugsys-mass-messaging` | Phase 4 — pending |
| Admin Panel | `ugsys-admin-panel` | Phase 4 — pending |

**Infrastructure**: Lambda + API Gateway, DynamoDB, EventBridge, SES, SNS, S3, CloudFront
**Region**: us-east-1
**Auth**: JWT RS256 — identity-manager issues tokens, all services validate via `ugsys-auth-client`
**IDs**: UUIDs (identity-manager uses UUID4; projects-registry will use ULIDs)

---

## 2. Shared Conventions

### 2.1 HTTP

- All routes versioned: `/api/v1/`
- Content-Type: `application/json`
- Dates: ISO 8601 UTC (`2026-01-15T10:30:00Z`)
- IDs: UUID4 (identity-manager, user-profile); ULID (projects-registry, omnichannel, mass-messaging)
- Pagination: `?page=1&page_size=20` (default 20, max 100)

### 2.2 Standard Response Envelopes

**Success (single)**:
```json
{ "data": { ... }, "meta": { "request_id": "..." } }
```

**Success (list)**:
```json
{
  "data": [ ... ],
  "meta": { "request_id": "...", "total": 100, "page": 1, "page_size": 20, "total_pages": 5 }
}
```

**Error**:
```json
{
  "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [ { "field": "email", "message": "Invalid format" } ] },
  "meta": { "request_id": "..." }
}
```

### 2.3 HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | OK |
| 201 | Created |
| 204 | No Content (DELETE) |
| 400 | Validation error |
| 401 | Not authenticated |
| 403 | Authenticated but not authorized (IDOR, wrong role) |
| 404 | Not found |
| 409 | Conflict (duplicate) |
| 422 | Unprocessable entity |
| 423 | Account locked |
| 429 | Rate limited |
| 500 | Internal server error |

### 2.4 JWT Claims (access token, RS256)

**ugsys target** (RS256 — all new services):
```json
{
  "sub": "<user_uuid>",
  "email": "user@example.com",
  "roles": ["member"],
  "status": "active",
  "type": "access",
  "iat": 1700000000,
  "exp": 1700003600,
  "jti": "<token_uuid>"
}
```

**Registry current** (HS256 — until migrated):
```json
{
  "sub": "<user_id>",
  "email": "user@example.com",
  "isAdmin": false,
  "exp": 1700003600,
  "iat": 1700000000,
  "type": "access"
}
```

> Roles are NOT embedded in the Registry JWT — they are fetched from the RBAC service on each request. The `isAdmin` flag is the only authorization claim in the token. ugsys identity-manager MUST embed `roles` in the JWT (RS256).

Service tokens additionally carry `"type": "service"` and `"client_id": "ugsys-projects-registry"`.

**Token TTLs**:
- Registry access token: 48h (configurable via `access_token_expire_hours`)
- Registry refresh token: 30 days
- ugsys target: access token 1h, refresh token 7d
- Password reset token: 1 hour (JWT `type: password_reset`)

### 2.5 Roles & Permissions

**Roles** (stored in JWT `roles` claim):

| Role | Level | Description |
|------|-------|-------------|
| `guest` | 0 | Unauthenticated — public read only |
| `member` | 1 | Authenticated user — default on register |
| `moderator` | 2 | Can manage all projects and subscriptions |
| `auditor` | 3 | Read-only access to all resources |
| `admin` | 4 | Full user + content management |
| `super_admin` | 5 | All permissions |
| `system` | 6 | Service accounts only |

**Granular permissions** (Phase 2 — added to JWT `permissions` claim):

```
user:read:own        user:read:all        user:create
user:update:own      user:update:all      user:delete:own      user:delete:all
user:admin
project:read:public  project:read:all     project:create
project:update:own   project:update:all   project:delete:own   project:delete:all
project:admin
subscription:read:own  subscription:read:all  subscription:create
subscription:update:own  subscription:update:all
subscription:delete:own  subscription:delete:all
subscription:admin
role:read  role:assign  role:revoke  role:admin
system:config  system:audit  system:monitor  system:backup
security:audit  security:admin
```

**Role → Permission matrix** (from `rbac.py` `DEFAULT_ROLES`):

| Permission | GUEST | USER | MODERATOR | ADMIN | SUPER_ADMIN | AUDITOR | SYSTEM |
|------------|-------|------|-----------|-------|-------------|---------|--------|
| `user:read:own` | | ✅ | ✅ | | ✅ | ✅ | |
| `user:update:own` | | ✅ | ✅ | | ✅ | | |
| `user:read:all` | | | | ✅ | ✅ | ✅ | |
| `user:create` | | | | ✅ | ✅ | | |
| `user:update:all` | | | | ✅ | ✅ | | |
| `user:admin` | | | | ✅ | ✅ | | |
| `project:read:public` | ✅ | ✅ | | | ✅ | | |
| `project:read:all` | | ✅ | ✅ | ✅ | ✅ | ✅ | |
| `project:create` | | ✅ | ✅ | ✅ | ✅ | | |
| `project:update:own` | | ✅ | | | ✅ | | |
| `project:update:all` | | | ✅ | ✅ | ✅ | | |
| `project:delete:own` | | ✅ | ✅ | | ✅ | | |
| `project:delete:all` | | | | ✅ | ✅ | | |
| `project:admin` | | | | ✅ | ✅ | | |
| `subscription:read:own` | | ✅ | | | ✅ | ✅ | |
| `subscription:read:all` | | | ✅ | ✅ | ✅ | ✅ | |
| `subscription:create` | | ✅ | ✅ | ✅ | ✅ | | |
| `subscription:update:own` | | ✅ | | | ✅ | | |
| `subscription:update:all` | | | ✅ | ✅ | ✅ | | |
| `subscription:delete:own` | | ✅ | ✅ | | ✅ | | |
| `subscription:delete:all` | | | | ✅ | ✅ | | |
| `subscription:admin` | | | | ✅ | ✅ | | |
| `role:read` | | | | ✅ | ✅ | ✅ | |
| `role:assign` | | | | ✅ | ✅ | | |
| `system:config` | | | ✅ | ✅ | ✅ | ✅ | ✅ |
| `system:audit` | | | | ✅ | ✅ | ✅ | |
| `system:monitor` | | | | ✅ | ✅ | | |
| `system:backup` | | | | | ✅ | | ✅ |
| `security:audit` | | | | ✅ | ✅ | ✅ | |
| `security:admin` | | | | | ✅ | | ✅ |

### 2.6 Middleware Stack (all services, in this order)

1. `CorrelationIdMiddleware` — reads `X-Request-ID` header or generates UUID4; binds to structlog context; echoes back in response header
2. `SecurityHeadersMiddleware` — sets `X-Content-Type-Options`, `X-Frame-Options: DENY`, `X-XSS-Protection`, `Strict-Transport-Security`, `Content-Security-Policy: default-src 'self'`, `Referrer-Policy`
3. `RateLimitMiddleware` — per-IP, 60 req/min default; returns 429 with `Retry-After` header

### 2.7 Public Endpoints (no auth required)

```
GET  /health
GET  /
POST /api/v1/auth/login
POST /api/v1/auth/register
POST /api/v1/auth/forgot-password
POST /api/v1/auth/reset-password
POST /api/v1/auth/verify-email
POST /api/v1/auth/resend-verification
POST /api/v1/auth/refresh
GET  /api/v1/projects/public          (projects-registry only)
```

### 2.8 Service-to-Service Auth

All S2S calls use `client_credentials` grant:
1. Caller POSTs to `identity-manager /api/v1/auth/service-token` with `client_id` + `client_secret`
2. Receives short-lived access token (`type: service`)
3. Passes token as `Authorization: Bearer <token>` on all S2S requests
4. Receiving service validates via `ugsys-auth-client` (checks `type: service` claim)

---

## 3. Identity Manager

**Repo**: `ugsys-identity-manager`
**Base URL**: `https://api.cbba.cloud.org.bo/identity` (or Lambda function URL in dev)
**DynamoDB table**: `ugsys-identity-{env}-users`

### 3.1 Domain Entity: User

```python
@dataclass
class User:
    id: UUID                              # UUID4, generated on creation
    email: str                            # lowercase, unique
    hashed_password: str                  # bcrypt 12 rounds
    full_name: str                        # html.escape()'d on input

    # Registry uses camelCase: firstName, lastName (separate fields)
    # ugsys identity-manager uses full_name (single field)

    status: UserStatus                    # pending_verification | active | inactive
    roles: list[UserRole]                 # default: [member]
    is_admin: bool                        # legacy flag; superseded by roles

    # Security
    failed_login_attempts: int            # default 0; reset on success
    account_locked_until: datetime | None # set after 5 failures; 30-min lock
    last_login_at: datetime | None        # updated on every successful login
    last_password_change: datetime | None
    require_password_change: bool         # default False; forces change on next login

    # Verification
    email_verified: bool                  # default False
    email_verification_token: str | None  # UUID4; cleared after verify
    email_verified_at: datetime | None

    # Password history — Phase 2
    password_history: list[str]           # last 5 bcrypt hashes

    created_at: datetime
    updated_at: datetime
```

**UserStatus**: `pending_verification` | `active` | `inactive`
**UserRole**: `super_admin` | `admin` | `moderator` | `auditor` | `member`

**API response shape** (never expose sensitive fields):
```json
{
  "id": "550e...",
  "email": "user@example.com",
  "full_name": "John Doe",
  "status": "active",
  "roles": ["member"],
  "is_admin": false,
  "email_verified": true,
  "require_password_change": false,
  "last_login_at": "2026-02-24T10:00:00Z",
  "last_password_change": "2026-01-01T00:00:00Z",
  "created_at": "2026-01-01T00:00:00Z",
  "updated_at": "2026-02-24T10:00:00Z"
}
```

**NEVER return**: `hashed_password`, `failed_login_attempts`, `account_locked_until`, `password_history`, `email_verification_token`

### 3.2 DynamoDB Schema

Table: `ugsys-identity-{env}-users`

| Attribute | Type | Key |
|-----------|------|-----|
| `PK` | String | Partition key — `USER#{id}` |
| `SK` | String | Sort key — `PROFILE` |
| `email` | String | GSI-1 PK (`email-index`) |
| `status` | String | GSI-2 PK (`status-index`), SK = `created_at` |
| All User fields | Various | — |

Token blacklist table: `ugsys-identity-{env}-token-blacklist`

| Attribute | Type | Key |
|-----------|------|-----|
| `jti` | String | Partition key |
| `ttl` | Number | DynamoDB TTL (auto-expire) |

### 3.3 Business Rules

**Account lockout**:
- Lock after 5 consecutive failed login attempts
- Lock duration: 30 minutes (`account_locked_until = now + 30min`)
- Reset `failed_login_attempts = 0` and `account_locked_until = None` on successful login
- Return `423 Locked` with `{ "retry_after_seconds": N }` when locked

**Password policy**:
- Minimum 8 characters
- At least 1 uppercase, 1 lowercase, 1 digit, 1 special character (`!@#$%^&*()_+-=[]{}|;:,.<>?`)
- Cannot reuse last 5 passwords (Phase 2)
- `require_password_change = True` forces change on next login — login returns `403` with `{ "require_password_change": true }`

**Password reset token**:
- TTL: 1 hour (JWT `type: password_reset`, `exp = now + 1h`)
- On reset: sets new hash, sets `last_password_change`, clears `require_password_change`

**Email verification**:
- New users start as `pending_verification`
- Token is UUID4, expires in 24 hours (stored in user record)
- On verify: `status → active`, `email_verified_at = now`, token cleared
- Login blocked for `pending_verification` users — returns `403` with `{ "email_not_verified": true }`

**Anti-enumeration**:
- `forgot-password` always returns 200 regardless of email existence
- `resend-verification` always returns 200 regardless of email existence
- Login error message: always `"Invalid credentials"` — never `"email not found"`

**Service accounts**:
- Stored in `SERVICE_ACCOUNTS_JSON` env var (JSON array of `{client_id, hashed_secret}`)
- `client_credentials` grant only — no user login
- Token carries `"type": "service"`, `"client_id": "..."` — no `roles` or `sub`

### 3.4 API Endpoints

#### Auth — `/api/v1/auth`

| Method | Path | Auth | Status |
|--------|------|------|--------|
| POST | `/register` | None | ✅ Done |
| POST | `/login` | None | ✅ Done |
| POST | `/refresh` | None | ✅ Done |
| POST | `/logout` | Bearer | ✅ Done (client-side only — no blacklist yet) |
| POST | `/forgot-password` | None | ✅ Done |
| POST | `/reset-password` | None | ✅ Done |
| GET | `/validate-reset-token/{token}` | None | ✅ Done |
| GET | `/validate` | Bearer | ✅ Done |
| POST | `/verify-email` | None | ❌ Missing |
| POST | `/resend-verification` | None | ❌ Missing |
| POST | `/password/change` | Bearer | ✅ Done |
| PUT | `/profile` | Bearer | ✅ Done |
| GET | `/me` | Bearer | ✅ Done |
| GET | `/subscriptions` | Bearer | ✅ Done |
| POST | `/validate-token` | Service | ❌ Missing (needed by auth-client remote validation) |
| POST | `/service-token` | None | ✅ Done |

**POST /register**
```json
// Request
{ "email": "user@example.com", "password": "SecurePass1!", "full_name": "John Doe" }
// Response 201
{ "message": "User registered", "user_id": "550e8400-e29b-41d4-a716-446655440000" }
// Side effects: publishes identity.user.registered; sends verification email via omnichannel
// Errors: 409 email already exists, 400 password too weak
```

**POST /login**
```json
// Request
{ "email": "user@example.com", "password": "SecurePass1!" }
// Response 200
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer",
  "expires_in": 3600,
  "user": { "id": "...", "email": "...", "firstName": "...", "lastName": "...", "roles": ["member"], "isAdmin": false },
  "require_password_change": false
}
// Side effects:
//   failure → increments failed_login_attempts; at 5 → locks account (423)
//   success → resets failed_login_attempts, sets last_login_at
//   if require_password_change → 403 { "require_password_change": true }
//   if pending_verification → 403 { "email_not_verified": true }
//   if account locked → 423 { "retry_after_seconds": N }
// Errors: 401 invalid credentials, 403 not verified / must change password, 423 locked
```

**POST /refresh**
```json
// Request
{ "refresh_token": "eyJ..." }
// Response 200
{ "access_token": "eyJ...", "refresh_token": "eyJ...", "token_type": "bearer" }
// Errors: 401 invalid or expired token
```

**POST /logout** (Bearer)
```json
// Request (Bearer token in header)
{ }  // body optional
// Response 200
{ "message": "Logged out successfully", "userId": "550e..." }
// Note: current impl is client-side only — token blacklist (jti → DynamoDB TTL) is ❌ Missing
```

**GET /validate** (Bearer)
```json
// Response 200
{ "valid": true, "user": { "id": "...", "email": "...", "isAdmin": false, "roles": [...] } }
// Errors: 401 invalid/expired token
```

**GET /validate-reset-token/{token}**
```json
// Response 200
{ "valid": true, "token": "<token>" }
// Errors: 200 with valid: false if expired/invalid
```

**PUT /profile** (Bearer)
```json
// Request — allowed fields only (email changes NOT allowed here)
{ "firstName": "John", "lastName": "Doe", "phone": "+591XXXXXXXX", "dateOfBirth": "1990-05-15", "address": { "street": "...", "city": "...", "state": "...", "postalCode": "...", "country": "BO" } }
// Response 200 — updated PersonResponse
// Errors: 400 if email change attempted, 403 if isAdmin change attempted
```

**GET /me** (Bearer)
```json
// Response 200 — current user object (PersonResponse shape)
{ "id": "...", "email": "...", "firstName": "...", "lastName": "...", "isAdmin": false, "isActive": true, "roles": [...] }
```

**GET /subscriptions** (Bearer)
```json
// Response 200
{ "subscriptions": [ { ...EnrichedSubscriptionResponse... } ], "count": 3 }
```

**POST /forgot-password**
```json
// Request
{ "email": "user@example.com" }
// Response 200 (always — anti-enumeration)
{ "message": "If that email is registered, a reset link has been sent" }
// Side effect (if user exists): publishes identity.auth.password_reset_requested
```

**POST /reset-password**
```json
// Request
{ "token": "<reset_token>", "new_password": "NewPass1!", "confirm_password": "NewPass1!" }
// Response 200
{ "message": "Password has been reset successfully", "userId": "...", "email": "..." }
// Side effects: sets new hash, sets last_password_change, clears require_password_change
// Errors: 400 invalid/expired token, 400 password too weak, 400 passwords don't match
```

**POST /password/change** (Bearer) ✅ Done
```json
// Request
{ "currentPassword": "OldPass1!", "newPassword": "NewPass1!", "confirmPassword": "NewPass1!" }
// Response 200
{ "message": "Password changed successfully", "userId": "..." }
// Side effects: validates current, sets new hash, sets last_password_change
// Errors: 400 wrong current password, 400 too weak, 400 passwords don't match
```

**POST /validate-token** (S2S only) ❌ Missing
```json
// Request (Service Bearer)
{ "token": "eyJ..." }
// Response 200
{ "valid": true, "sub": "550e...", "roles": ["member"], "type": "access" }
// Errors: 401 invalid token
// Note: GET /validate serves the same purpose for user tokens; this S2S endpoint is needed
//   by ugsys-auth-client's remote validation fallback path
```

**POST /verify-email** ❌ Missing
```json
// Request
{ "token": "<verification_token>" }
// Response 200
{ "message": "Email verified successfully" }
// Side effects: status → active, email_verified_at = now, token cleared
// Errors: 400 invalid/expired token, 409 already verified
```

**POST /resend-verification** ❌ Missing
```json
// Request
{ "email": "user@example.com" }
// Response 200 (always — anti-enumeration)
{ "message": "If the account exists and is unverified, a new email has been sent" }
// Side effect (if user pending): publishes identity.auth.verification_requested
```

**POST /service-token**
```json
// Request
{ "client_id": "ugsys-projects-registry", "client_secret": "<secret>" }
// Response 200
{ "access_token": "eyJ...", "token_type": "bearer", "expires_in": 3600 }
// Errors: 401 invalid credentials
```

#### Users — `/api/v1/users`

| Method | Path | Auth | Status |
|--------|------|------|--------|
| GET | `/` | admin | ✅ Done |
| GET | `/me` | Bearer | ✅ Done |
| GET | `/{id}` | own or admin | ✅ Done |
| PATCH | `/{id}` | own | ✅ Done (full_name only) |
| DELETE | `/{id}` | admin | ✅ Done (deactivates) |
| GET | `/{id}/roles` | own or admin | ✅ Done |
| PUT | `/{id}/roles/{role}` | admin | ✅ Done |
| DELETE | `/{id}/roles/{role}` | admin | ✅ Done |
| POST | `/{id}/suspend` | admin | ❌ Missing |
| POST | `/{id}/activate` | admin | ❌ Missing |
| POST | `/{id}/require-password-change` | admin | ❌ Missing |

**GET /users** (admin)
```json
// Query: ?page=1&page_size=20&status=active&role=member
// Response 200
[{ "id": "550e...", "email": "...", "full_name": "...", "status": "active", "roles": ["member"] }]
// Note: current impl returns list directly — should be wrapped in envelope
```

**GET /users/me** and **GET /users/{id}**
```json
// Response 200
{ "id": "550e...", "email": "...", "full_name": "...", "status": "active", "roles": ["member"] }
// NEVER return: hashed_password, failed_login_attempts, account_locked_until, password_history
// IDOR check: non-admin can only fetch own id
```

**PATCH /users/{id}**
```json
// Request (own user only)
{ "full_name": "New Name" }
// Response 200 — updated user object
// Note: email changes NOT allowed here — managed separately (Phase 2)
```

**PUT /users/{id}/roles/{role}** (admin)
```json
// Path param role: member | admin | super_admin | moderator | auditor
// Response 200 — updated user object
// Errors: 404 user not found
```

**DELETE /users/{id}** (admin)
```json
// Response 200 — deactivated user object (status → inactive)
// Note: soft delete only — record is never removed from DynamoDB
```

#### Roles — `/api/v1/roles`

| Method | Path | Auth | Status |
|--------|------|------|--------|
| GET | `/` | admin | ✅ Done |

```json
// Response 200
[
  { "name": "member", "description": "Authenticated user with basic permissions" },
  { "name": "admin", "description": "System administrator" },
  { "name": "super_admin", "description": "Full system access" },
  { "name": "moderator", "description": "Content moderation" },
  { "name": "auditor", "description": "Read-only compliance access" }
]
```

### 3.5 Events Published

| Event | Trigger | Key Payload Fields |
|-------|---------|-------------------|
| `identity.user.registered` | POST /register | `user_id`, `email`, `verification_token`, `expires_at` |
| `identity.user.activated` | Email verified or password reset | `user_id`, `email` |
| `identity.user.deactivated` | DELETE /users/{id} | `user_id`, `email` |
| `identity.user.role_changed` | PUT/DELETE /users/{id}/roles/{role} | `user_id`, `email`, `role`, `action` (assigned/removed) |
| `identity.auth.login_success` | POST /login success | `user_id`, `ip_address` |
| `identity.auth.login_failed` | POST /login failure | `email`, `ip_address`, `attempt_count` |
| `identity.auth.account_locked` | 5th failed attempt | `user_id`, `email`, `locked_until` |
| `identity.auth.password_reset_requested` | POST /forgot-password | `user_id`, `email`, `reset_token`, `expires_at` |
| `identity.auth.password_changed` | POST /change-password or /reset-password | `user_id`, `email` |
| `identity.auth.verification_requested` | POST /resend-verification | `user_id`, `email`, `verification_token`, `expires_at` |

---

## 4. User Profile Service

**Repo**: `ugsys-user-profile-service`
**Base URL**: `https://api.cbba.cloud.org.bo/profiles`
**DynamoDB table**: `ugsys-profiles-{env}`

### 4.1 Domain Entity: UserProfile

```python
@dataclass
class Address:
    street: str = ""
    city: str = ""
    state: str = ""
    postal_code: str = ""
    country: str = ""          # ISO 3166-1 alpha-2 recommended

@dataclass
class UserProfile:
    user_id: UUID              # same UUID as identity-manager
    email: str                 # denormalized from identity-manager; read-only here
    full_name: str             # synced from identity-manager on register
    phone: str = ""            # E.164 format: +591XXXXXXXX
    date_of_birth: str = ""    # YYYY-MM-DD
    address: Address = field(default_factory=Address)
    email_verified: bool = False
    require_password_change: bool = False  # synced from identity-manager

    # Migration audit
    migrated_from: str | None = None   # "registry" if migrated
    migrated_at: datetime | None = None

    created_at: datetime
    updated_at: datetime
```

**Missing fields to add (from Registry PersonResponse)**:
- `date_of_birth` ✅ present
- `phone` ✅ present
- `address` ✅ present
- `is_active` — covered by identity-manager `status`
- `last_login_at` — owned by identity-manager, not profile
- `notification_preferences: dict` — `{ email: bool, sms: bool, whatsapp: bool }` ❌ Missing
- `language: str` — ISO 639-1, default `"es"` ❌ Missing
- `timezone: str` — IANA, default `"America/La_Paz"` ❌ Missing
- `avatar_url: str | None` — S3/CloudFront URL ❌ Missing
- `bio: str | None` — max 500 chars ❌ Missing
- `display_name: str | None` ❌ Missing

### 4.2 DynamoDB Schema

Table: `ugsys-profiles-{env}`

| Attribute | Type | Key |
|-----------|------|-----|
| `PK` | String | Partition key — `PROFILE#{user_id}` |
| `SK` | String | Sort key — `PROFILE` |
| `email` | String | GSI-1 PK (`email-index`) |

### 4.3 API Endpoints

| Method | Path | Auth | Status |
|--------|------|------|--------|
| GET | `/api/v1/profiles/me` | Bearer | ✅ Done |
| GET | `/api/v1/profiles/{user_id}` | Bearer (own or admin) | ✅ Done |
| PATCH | `/api/v1/profiles/{user_id}/contact` | Bearer (own or admin) | ✅ Done |
| PATCH | `/api/v1/profiles/{user_id}/personal` | Bearer (own or admin) | ✅ Done |
| POST | `/api/v1/profiles/{user_id}/avatar` | Bearer (own or admin) | ❌ Missing |
| DELETE | `/api/v1/profiles/{user_id}/avatar` | Bearer (own or admin) | ❌ Missing |
| GET | `/api/v1/profiles` | admin | ❌ Missing |
| DELETE | `/api/v1/profiles/{user_id}` | admin | ❌ Missing |

**GET /profiles/me** and **GET /profiles/{user_id}**
```json
// Response 200
{
  "user_id": "550e...",
  "email": "user@example.com",
  "full_name": "John Doe",
  "phone": "+591XXXXXXXX",
  "date_of_birth": "1990-05-15",
  "address": { "street": "", "city": "Cochabamba", "state": "", "postal_code": "", "country": "BO" },
  "email_verified": true,
  "avatar_url": null,
  "bio": null,
  "language": "es",
  "timezone": "America/La_Paz",
  "notification_preferences": { "email": true, "sms": false, "whatsapp": false }
}
// IDOR: non-admin can only fetch own profile
```

**PATCH /profiles/{user_id}/contact**
```json
// Request (all optional)
{ "phone": "+591XXXXXXXX", "street": "Av. Heroinas", "city": "Cochabamba", "state": "Cochabamba", "postal_code": "", "country": "BO" }
// Response 200 — updated profile
```

**PATCH /profiles/{user_id}/personal**
```json
// Request (all optional)
{ "full_name": "John Doe", "date_of_birth": "1990-05-15" }
// Response 200 — updated profile
// Note: email is read-only — managed by identity-manager
```

**POST /profiles/{user_id}/avatar** ❌ Missing
```
// Request: multipart/form-data, field: "file"
// Constraints: max 5MB, JPEG/PNG/WebP only
// Response 200
{ "avatar_url": "https://cdn.cbba.cloud.org.bo/avatars/550e....jpg" }
// Side effect: uploads to S3 bucket ugsys-avatars-{env}, sets CloudFront URL
```

### 4.4 Events Consumed

| Event | Action |
|-------|--------|
| `identity.user.registered` | Create profile with `email`, `full_name` from payload |
| `identity.user.deactivated` | Soft-delete profile (set `deleted_at`) |
| `identity.auth.password_changed` | Clear `require_password_change = False` |
| `identity.user.role_changed` | No action needed (roles live in identity-manager) |

### 4.5 Events Published

| Event | Trigger |
|-------|---------|
| `profile.updated` | PATCH /contact or /personal |
| `profile.avatar_updated` | POST /avatar |

### 4.6 Migration from Registry

Script: `scripts/migrate_from_registry.py`

Transforms Registry `Person` → `UserProfile`:

| Registry field | Profile field |
|----------------|---------------|
| `id` | `user_id` |
| `email` | `email` |
| `firstName + " " + lastName` | `full_name` |
| `phone` | `phone` |
| `dateOfBirth` | `date_of_birth` |
| `address.street` | `address.street` |
| `address.city` | `address.city` |
| `address.state` | `address.state` |
| `address.postalCode` | `address.postal_code` |
| `address.country` | `address.country` |
| `emailVerified` | `email_verified` |
| `requirePasswordChange` | `require_password_change` |

---

## 5. Projects Registry

**Repo**: `ugsys-projects-registry`
**Base URL**: `https://api.cbba.cloud.org.bo/projects`
**DynamoDB tables**: `ugsys-projects-{env}`, `ugsys-subscriptions-{env}`, `ugsys-form-submissions-{env}`
**Source**: Extracted from `Registry/registry-api` — all business logic preserved

### 5.1 Domain Entity: Project

```python
@dataclass
class Project:
    id: str                              # ULID
    name: str                            # 1–200 chars
    description: str                     # 1–5000 chars
    start_date: str                      # YYYY-MM-DD
    end_date: str                        # YYYY-MM-DD; must be >= start_date
    max_participants: int                # 1–1000
    status: ProjectStatus                # pending | active | completed | cancelled
    created_by: str                      # user_id of creator
    current_participants: int            # default 0; incremented on active subscription

    # Optional fields
    category: str | None                 # max 100 chars
    location: str | None                 # max 200 chars
    requirements: str | None             # max 5000 chars
    registration_end_date: str | None    # YYYY-MM-DD
    is_enabled: bool                     # default True
    rich_text: str | None                # HTML/Markdown, max 10000 chars / 15KB

    # Dynamic forms
    form_schema: FormSchema | None       # max 50KB JSON; up to 20 custom fields
    images: list[ProjectImage]           # S3/CloudFront URLs

    # Notifications
    enable_subscription_notifications: bool  # default True
    notification_emails: list[str]           # additional admin emails

    created_at: str
    updated_at: str
```

**ProjectStatus**: `pending` | `active` | `completed` | `cancelled`

### 5.2 Domain Entity: Subscription

```python
@dataclass
class Subscription:
    id: str                  # ULID
    person_id: str           # user_id from identity-manager
    project_id: str          # project ULID
    status: str              # pending | active | cancelled | rejected
    subscription_date: str   # ISO date
    is_active: bool          # default True
    notes: str | None        # max 1000 chars
    created_at: str
    updated_at: str
```

**Subscription status flow**:
- `pending` → `active` (admin approves) or `rejected` (admin rejects)
- `active` → `cancelled` (user or admin cancels)
- Super admins are auto-approved on subscribe (status starts as `active`)
- Regular users start as `pending` and require admin approval

### 5.3 Domain Entity: FormSchema

```python
@dataclass
class CustomField:
    id: str                              # unique within form
    type: Literal["text", "poll_single", "poll_multiple"]
    question: str                        # 1–500 chars
    options: list[str]                   # 2–10 unique options (required for poll types; ignored for text)
    required: bool                       # default False

@dataclass
class FormSchema:
    version: str                         # default "1.0"
    fields: list[CustomField]            # max 20 fields; unique IDs
    rich_text_description: str           # Markdown, max 10000 chars
```

**Validation rules** (from `form_submission_service.py`):
- `poll_single`: response must be a string matching one of `field.options`
- `poll_multiple`: response must be a list; each item must be in `field.options`
- `text`: no additional validation beyond required check
- Required fields: if `field.required == true` and field_id absent from submission → `ValueError`

### 5.4 Domain Entity: FormSubmission

```python
@dataclass
class FormSubmission:
    id: str                  # ULID
    project_id: str
    person_id: str
    responses: dict          # field_id → answer value
    created_at: datetime
    updated_at: datetime
```

### 5.5 Domain Entity: ProjectImage

```python
@dataclass
class ProjectImage:
    url: str          # S3 or CloudFront HTTPS URL
    filename: str     # original filename, max 255 chars
    size: int         # bytes, max 10MB
```

### 5.6 DynamoDB Schema

**Projects table** `ugsys-projects-{env}`:

| Attribute | Type | Key |
|-----------|------|-----|
| `PK` | String | `PROJECT#{id}` |
| `SK` | String | `PROJECT` |
| `status` | String | GSI-1 PK (`status-index`), SK = `created_at` |
| `created_by` | String | GSI-2 PK (`creator-index`) |

**Subscriptions table** `ugsys-subscriptions-{env}`:

| Attribute | Type | Key |
|-----------|------|-----|
| `PK` | String | `SUB#{id}` |
| `SK` | String | `SUB` |
| `person_id` | String | GSI-1 PK (`person-index`), SK = `created_at` |
| `project_id` | String | GSI-2 PK (`project-index`), SK = `created_at` |
| `person_id + "#" + project_id` | String | GSI-3 PK (`unique-check-index`) — for duplicate check |

**Form submissions table** `ugsys-form-submissions-{env}`:

| Attribute | Type | Key |
|-----------|------|-----|
| `PK` | String | `SUBMISSION#{id}` |
| `SK` | String | `SUBMISSION` |
| `project_id` | String | GSI-1 PK (`project-index`) |
| `person_id` | String | GSI-2 PK (`person-index`) |

### 5.7 API Endpoints

#### Projects — `/api/v1/projects`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | Bearer | List all projects (admin/moderator) |
| GET | `/public` | None | List active/enabled projects |
| GET | `/{id}` | Bearer | Get project by ID |
| POST | `/` | Bearer | Create project |
| PUT | `/{id}` | Bearer (owner or admin) | Update project |
| DELETE | `/{id}` | admin | Delete project |
| GET | `/{id}/subscriptions` | Bearer (admin/moderator) | List project subscriptions |
| POST | `/{id}/subscriptions` | Bearer | Subscribe to project |
| PUT | `/{id}/subscribers/{sub_id}` | admin | Update subscription (approve/reject) |
| DELETE | `/{id}/subscribers/{sub_id}` | Bearer (own or admin) | Cancel subscription |
| GET | `/{id}/enhanced` | Bearer | Get project with form schema |
| PUT | `/{id}/form-schema` | Bearer (owner or admin) | Update form schema |
| POST | `/enhanced` | Bearer | Create project with dynamic fields |

**GET /projects** (admin/moderator)
```json
// Query: ?limit=50&status=active&category=tech
// Response 200
{ "data": [ { "id": "01JXXX", "name": "...", "status": "active", "current_participants": 5, "max_participants": 20, ... } ], "meta": { "total": 12 } }
```

**GET /projects/public** (no auth)
```json
// Returns only projects where status=active AND is_enabled=true
// Response 200 — same envelope, excludes notification_emails and internal fields
```

**POST /projects** (Bearer)
```json
// Request
{
  "name": "AWS Workshop",
  "description": "Hands-on CDK workshop",
  "start_date": "2026-03-01",
  "end_date": "2026-03-31",
  "max_participants": 30,
  "status": "pending",
  "category": "cloud",
  "location": "Cochabamba",
  "requirements": "Basic AWS knowledge",
  "registration_end_date": "2026-02-28",
  "enable_subscription_notifications": true,
  "notification_emails": ["admin@cbba.cloud.org.bo"]
}
// Response 201 — full project object
// Side effect: publishes projects.project.created
// Errors: 400 end_date before start_date, 400 form_schema too large
```

**PUT /projects/{id}** (owner or admin)
```json
// Request — all fields optional
{ "status": "active", "max_participants": 50 }
// Response 200 — updated project
// Side effect: publishes projects.project.updated
```

**POST /projects/{id}/subscriptions** (Bearer)
```json
// Request
{ "person_id": "550e...", "notes": "Interested in CDK track" }
// Response 201
{ "data": { "id": "01JYYY", "person_id": "550e...", "project_id": "01JXXX", "status": "pending", ... } }
// Business rules:
//   - Duplicate check: 409 if subscription already exists for person+project
//   - super_admin → status = active (auto-approved)
//   - others → status = pending
//   - if enable_subscription_notifications → sends email to project notification_emails
// Side effect: publishes projects.subscription.created
```

**PUT /projects/{id}/subscribers/{sub_id}** (admin)
```json
// Request
{ "status": "active", "notes": "Approved" }
// Response 200 — updated subscription
// Side effect: if status → active, publishes projects.subscription.approved
//              if status → rejected, publishes projects.subscription.rejected
```

**DELETE /projects/{id}/subscribers/{sub_id}** (own or admin)
```json
// Response 200
{ "data": { "deleted": true, "subscription_id": "01JYYY" } }
// Side effect: publishes projects.subscription.cancelled
```

**PUT /projects/{id}/form-schema** (owner or admin)
```json
// Request
{
  "version": "1.0",
  "fields": [
    { "id": "q1", "type": "poll_single", "question": "Experience level?", "options": ["Beginner", "Intermediate", "Advanced"], "required": true }
  ],
  "rich_text_description": "## Details\n\nMarkdown content here"
}
// Response 200 — updated project
// Validation: max 20 fields, unique field IDs, 2–10 options per poll, schema max 50KB
```

#### Subscriptions — `/api/v1/subscriptions`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | admin | List all subscriptions |
| GET | `/{id}` | Bearer | Get subscription by ID |
| POST | `/` | Bearer | Create subscription (direct) |
| PUT | `/{id}` | admin | Update subscription |
| DELETE | `/{id}` | Bearer (own or admin) | Delete subscription |
| POST | `/check` | Bearer | Check if subscription exists |
| GET | `/person/{person_id}` | Bearer (own or admin) | Get person's subscriptions (enriched) |
| GET | `/project/{project_id}` | admin | Get project's subscriptions (enriched) |

**GET /subscriptions/person/{person_id}** — returns `EnrichedSubscriptionResponse`:
```json
{
  "id": "01JYYY",
  "person_id": "550e...",
  "project_id": "01JXXX",
  "status": "active",
  "subscription_date": "2026-02-01",
  "is_active": true,
  "project_name": "AWS Workshop",
  "project_description": "...",
  "project_status": "active",
  "person_name": "John Doe",
  "person_email": "user@example.com"
}
```

**POST /subscriptions/check**
```json
// Request — takes personId + projectId (NOT email)
{ "personId": "550e...", "projectId": "01JXXX" }
// Response 200
{ "data": { "exists": true } }
// Note: the frontend workflow first calls POST /public/check-email to resolve email → personId,
//   then passes personId here. See subscription workflow below.
```

#### Public — `/api/v1/public` (no auth required)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/check-email` | None | Check if email is registered |
| POST | `/register` | None | Create account without subscribing |
| POST | `/subscribe` | None | Register + subscribe in one call |

**POST /public/check-email**
```json
// Request
{ "email": "user@example.com" }
// Response 200
{ "exists": true }
```

**POST /public/register**
```json
// Request
{
  "email": "user@example.com", "firstName": "John", "lastName": "Doe",
  "password": "SecurePass123!", "phone": "+591 70000000",
  "dateOfBirth": "1990-01-15",
  "address": { "street": "", "city": "", "state": "", "postalCode": "", "country": "" }
}
// Response 201
{ "personId": "...", "email": "...", "firstName": "...", "lastName": "...", "emailSent": true, "message": "Account created successfully!" }
// Errors: 400 if email already exists
```

**POST /public/subscribe**
```json
// Request
{
  "email": "user@example.com", "projectId": "01JXXX",
  "firstName": "John", "lastName": "Doe", "phone": "+591 70000000",
  "dateOfBirth": "1990-01-15", "address": { ... }
}
// Response 201
{ "subscription": { ... }, "personCreated": true, "emailSent": true, "message": "Successfully subscribed!" }
// Business rules:
//   - If person exists: check for duplicate subscription (409 if exists), create PENDING subscription
//   - If person does not exist: create person with temp password, create PENDING subscription
//   - ALL public subscriptions start as PENDING (admin approval required)
//   - Sends pending-approval email to subscriber
//   - Sends admin notification email (configurable via env var — NOT hardcoded)
```

#### Subscription Workflow (frontend decision tree)

```
POST /public/check-email  { "email": "..." }
         │
         ├─ exists: false ──► POST /public/subscribe  → account created + subscription pending
         │
         └─ exists: true
                  │
                  └─► POST /subscriptions/check  { "personId": "...", "projectId": "..." }
                               │
                               ├─ exists: false ──► Show "please login to subscribe"
                               │
                               └─ exists: true  ──► Show "already subscribed — login to view status"
```

#### Form Submissions — `/api/v1/form-submissions`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/` | Bearer | Submit form responses |
| GET | `/project/{project_id}` | admin | Get all submissions for project |
| GET | `/person/{person_id}/project/{project_id}` | Bearer (own or admin) | Get specific submission |

**POST /form-submissions**
```json
// Request
{
  "project_id": "01JXXX",
  "person_id": "550e...",
  "responses": { "q1": "Intermediate", "q2": ["Option A", "Option B"] }
}
// Response 201 — full submission object
// Validation: validates responses against project's form_schema
// Errors: 400 invalid responses, 400 project has no form schema
```

#### Images — `/api/v1/images`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/upload-url` | Bearer | Get presigned S3 upload URL |

**POST /images/upload-url**
```json
// Request
{ "filename": "workshop.jpg", "content_type": "image/jpeg", "file_size": 2048000 }
// Response 200
{ "data": { "upload_url": "https://s3.amazonaws.com/...", "image_id": "01JZZZ", "cloud_front_url": "https://cdn.cbba.cloud.org.bo/..." } }
// Constraints: max 10MB, JPEG/PNG/GIF/WebP only
```

### 5.8 Events Published

| Event | Trigger |
|-------|---------|
| `projects.project.created` | POST /projects |
| `projects.project.updated` | PUT /projects/{id} |
| `projects.project.published` | PUT /projects/{id} with status → active |
| `projects.project.deleted` | DELETE /projects/{id} |
| `projects.subscription.created` | POST /projects/{id}/subscriptions |
| `projects.subscription.approved` | PUT subscription status → active |
| `projects.subscription.rejected` | PUT subscription status → rejected |
| `projects.subscription.cancelled` | DELETE subscription |

### 5.9 Events Consumed

| Event | Action |
|-------|--------|
| `identity.user.deactivated` | Cancel all active subscriptions for user |

---

## 6. Omnichannel Service

**Repo**: `ugsys-omnichannel-service`
**Base URL**: `https://api.cbba.cloud.org.bo/omnichannel`
**DynamoDB tables**: `ugsys-messages-{env}`, `ugsys-templates-{env}`
**Phase**: 3 — pending implementation

### 6.1 Purpose

Single entry point for all outbound notifications across channels. Other services publish events to EventBridge; omnichannel consumes them and routes to the appropriate channel. Direct API calls also accepted for transactional messages.

### 6.2 Supported Channels

| Channel | Provider | Use case |
|---------|----------|----------|
| Email | AWS SES | Verification, password reset, subscription notifications |
| SMS | AWS SNS | OTP, urgent alerts |
| WhatsApp | Meta Cloud API | Community updates, event reminders |
| Slack | Slack API | Internal team notifications, CI/CD alerts |
| Telegram | Telegram Bot API | Community channel broadcasts |

### 6.3 Domain Entity: Message

```python
@dataclass
class Message:
    id: str                      # ULID
    channel: Channel             # email | sms | whatsapp | slack | telegram
    recipient: str               # email address, phone E.164, channel-specific ID
    template_id: str | None      # reference to MessageTemplate
    subject: str | None          # email only
    body: str                    # rendered content
    status: MessageStatus        # queued | sending | sent | delivered | failed
    metadata: dict               # channel-specific metadata
    retry_count: int             # default 0; max 3
    error_message: str | None
    sent_at: datetime | None
    delivered_at: datetime | None
    created_at: datetime
    updated_at: datetime
```

**MessageStatus**: `queued` | `sending` | `sent` | `delivered` | `failed`

### 6.4 Domain Entity: MessageTemplate

```python
@dataclass
class MessageTemplate:
    id: str                      # ULID
    name: str                    # unique slug, e.g. "email-verification"
    channel: Channel
    subject_template: str | None # Jinja2, email only
    body_template: str           # Jinja2
    variables: list[str]         # required variable names
    is_active: bool
    created_at: datetime
    updated_at: datetime
```

**Built-in templates** (must exist at bootstrap):

| Template ID | Channel | Trigger |
|-------------|---------|---------|
| `email-verification` | email | `identity.user.registered` |
| `password-reset` | email | `identity.auth.password_reset_requested` |
| `subscription-notification` | email | `projects.subscription.created` |
| `subscription-approved` | email | `projects.subscription.approved` |
| `subscription-rejected` | email | `projects.subscription.rejected` |
| `welcome` | email | `identity.user.activated` |

### 6.5 API Endpoints

#### Messages — `/api/v1/messages`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/` | Service | Send a message directly |
| GET | `/{id}` | admin | Get message status |
| GET | `/` | admin | List messages (paginated) |
| POST | `/{id}/retry` | admin | Retry failed message |

**POST /messages** (S2S)
```json
// Request
{
  "channel": "email",
  "recipient": "user@example.com",
  "template_id": "email-verification",
  "variables": { "verification_url": "https://cbba.cloud.org.bo/verify?token=..." }
}
// Response 201
{ "data": { "id": "01JXXX", "status": "queued" } }
```

#### Templates — `/api/v1/templates`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | admin | List templates |
| GET | `/{id}` | admin | Get template |
| POST | `/` | admin | Create template |
| PUT | `/{id}` | admin | Update template |
| DELETE | `/{id}` | admin | Delete template |

### 6.6 Events Consumed

| Event | Action |
|-------|--------|
| `identity.user.registered` | Send `email-verification` email |
| `identity.auth.password_reset_requested` | Send `password-reset` email |
| `identity.auth.verification_requested` | Resend `email-verification` email |
| `identity.user.activated` | Send `welcome` email |
| `projects.subscription.created` | Send `subscription-notification` to project admins |
| `projects.subscription.approved` | Send `subscription-approved` to subscriber |
| `projects.subscription.rejected` | Send `subscription-rejected` to subscriber |

### 6.7 Events Published

| Event | Trigger |
|-------|---------|
| `omnichannel.message.queued` | Message created |
| `omnichannel.message.sent` | Channel API accepted message |
| `omnichannel.message.delivered` | Delivery confirmed (webhook) |
| `omnichannel.message.failed` | All retries exhausted |

---

## 7. Mass Messaging

**Repo**: `ugsys-mass-messaging`
**Base URL**: `https://api.cbba.cloud.org.bo/campaigns`
**DynamoDB tables**: `ugsys-campaigns-{env}`, `ugsys-audiences-{env}`, `ugsys-campaign-analytics-{env}`
**Phase**: 4 — pending implementation

### 7.1 Domain Entity: Campaign

```python
@dataclass
class Campaign:
    id: str                          # ULID
    name: str                        # 1–200 chars
    description: str | None
    channel: Channel                 # email | sms | whatsapp | slack | telegram
    template_id: str                 # reference to omnichannel template
    audience_id: str                 # reference to Audience
    status: CampaignStatus
    scheduled_at: datetime | None    # None = send immediately
    started_at: datetime | None
    completed_at: datetime | None
    created_by: str                  # user_id
    stats: CampaignStats
    created_at: datetime
    updated_at: datetime

@dataclass
class CampaignStats:
    total_recipients: int = 0
    sent: int = 0
    delivered: int = 0
    failed: int = 0
    open_rate: float = 0.0           # email only
```

**CampaignStatus**: `draft` | `scheduled` | `executing` | `completed` | `failed` | `cancelled`

### 7.2 Domain Entity: Audience

```python
@dataclass
class Audience:
    id: str                          # ULID
    name: str
    description: str | None
    filter_criteria: dict            # e.g. { "roles": ["member"], "project_id": "01JXXX" }
    recipient_count: int             # cached count
    created_by: str
    created_at: datetime
    updated_at: datetime
```

### 7.3 API Endpoints

#### Campaigns — `/api/v1/campaigns`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | admin | List campaigns |
| GET | `/{id}` | admin | Get campaign |
| POST | `/` | admin | Create campaign |
| PUT | `/{id}` | admin | Update campaign |
| DELETE | `/{id}` | admin | Delete draft campaign |
| POST | `/{id}/schedule` | admin | Schedule campaign |
| POST | `/{id}/execute` | admin | Execute immediately |
| POST | `/{id}/cancel` | admin | Cancel scheduled/executing |
| GET | `/{id}/analytics` | admin | Get campaign analytics |

#### Audiences — `/api/v1/audiences`

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/` | admin | List audiences |
| GET | `/{id}` | admin | Get audience |
| POST | `/` | admin | Create audience |
| PUT | `/{id}` | admin | Update audience |
| DELETE | `/{id}` | admin | Delete audience |
| POST | `/{id}/preview` | admin | Preview recipient list |

### 7.4 Events Published

| Event | Trigger |
|-------|---------|
| `campaigns.campaign.created` | POST /campaigns |
| `campaigns.campaign.scheduled` | POST /campaigns/{id}/schedule |
| `campaigns.campaign.executing` | Execution starts |
| `campaigns.campaign.completed` | All messages dispatched |
| `campaigns.campaign.failed` | Execution error |
| `campaigns.campaign.cancelled` | POST /campaigns/{id}/cancel |

---

## 8. EventBridge Event Catalog

**Bus**: `ugsys-platform-bus` (custom EventBridge bus — actual CDK name from `EventBusStack`)
**Source prefix**: `ugsys.<service>`

All events follow this envelope:

```json
{
  "source": "ugsys.identity-manager",
  "detail-type": "identity.user.registered",
  "detail": {
    "event_id": "<ULID>",
    "event_version": "1.0",
    "timestamp": "2026-02-24T10:00:00Z",
    "correlation_id": "<UUID>",
    "payload": { ... }
  }
}
```

### 8.1 Identity Manager Events

| Detail-type | Source | Payload |
|-------------|--------|---------|
| `identity.user.registered` | `ugsys.identity-manager` | `{ user_id, email, full_name, verification_token, expires_at }` |
| `identity.user.activated` | `ugsys.identity-manager` | `{ user_id, email }` |
| `identity.user.deactivated` | `ugsys.identity-manager` | `{ user_id, email }` |
| `identity.user.role_changed` | `ugsys.identity-manager` | `{ user_id, email, role, action }` — action: `assigned` or `removed` |
| `identity.auth.login_success` | `ugsys.identity-manager` | `{ user_id, ip_address }` |
| `identity.auth.login_failed` | `ugsys.identity-manager` | `{ email, ip_address, attempt_count }` |
| `identity.auth.account_locked` | `ugsys.identity-manager` | `{ user_id, email, locked_until }` |
| `identity.auth.password_reset_requested` | `ugsys.identity-manager` | `{ user_id, email, reset_token, expires_at }` |
| `identity.auth.password_changed` | `ugsys.identity-manager` | `{ user_id, email }` |
| `identity.auth.verification_requested` | `ugsys.identity-manager` | `{ user_id, email, verification_token, expires_at }` |

### 8.2 Projects Registry Events

| Detail-type | Source | Payload |
|-------------|--------|---------|
| `projects.project.created` | `ugsys.projects-registry` | `{ project_id, name, status, created_by }` |
| `projects.project.updated` | `ugsys.projects-registry` | `{ project_id, changed_fields }` |
| `projects.project.published` | `ugsys.projects-registry` | `{ project_id, name }` |
| `projects.project.deleted` | `ugsys.projects-registry` | `{ project_id }` |
| `projects.subscription.created` | `ugsys.projects-registry` | `{ subscription_id, person_id, project_id, status, project_name, notification_emails }` |
| `projects.subscription.approved` | `ugsys.projects-registry` | `{ subscription_id, person_id, project_id, person_email, project_name }` |
| `projects.subscription.rejected` | `ugsys.projects-registry` | `{ subscription_id, person_id, project_id, person_email, project_name }` |
| `projects.subscription.cancelled` | `ugsys.projects-registry` | `{ subscription_id, person_id, project_id }` |

### 8.3 Omnichannel Events

| Detail-type | Source | Payload |
|-------------|--------|---------|
| `omnichannel.message.queued` | `ugsys.omnichannel` | `{ message_id, channel, recipient, template_id }` |
| `omnichannel.message.sent` | `ugsys.omnichannel` | `{ message_id, channel, sent_at }` |
| `omnichannel.message.delivered` | `ugsys.omnichannel` | `{ message_id, channel, delivered_at }` |
| `omnichannel.message.failed` | `ugsys.omnichannel` | `{ message_id, channel, error_message, retry_count }` |

### 8.4 Mass Messaging Events

| Detail-type | Source | Payload |
|-------------|--------|---------|
| `campaigns.campaign.created` | `ugsys.mass-messaging` | `{ campaign_id, name, channel }` |
| `campaigns.campaign.scheduled` | `ugsys.mass-messaging` | `{ campaign_id, scheduled_at }` |
| `campaigns.campaign.executing` | `ugsys.mass-messaging` | `{ campaign_id, total_recipients }` |
| `campaigns.campaign.completed` | `ugsys.mass-messaging` | `{ campaign_id, stats }` |
| `campaigns.campaign.failed` | `ugsys.mass-messaging` | `{ campaign_id, error_message }` |
| `campaigns.campaign.cancelled` | `ugsys.mass-messaging` | `{ campaign_id }` |

### 8.5 EventBridge Rules (consumers)

| Rule | Source pattern | Target |
|------|---------------|--------|
| `user-profile-on-register` | `identity.user.registered` | user-profile Lambda |
| `user-profile-on-deactivate` | `identity.user.deactivated` | user-profile Lambda |
| `user-profile-on-password-changed` | `identity.auth.password_changed` | user-profile Lambda |
| `omnichannel-on-register` | `identity.user.registered` | omnichannel Lambda |
| `omnichannel-on-password-reset` | `identity.auth.password_reset_requested` | omnichannel Lambda |
| `omnichannel-on-verification` | `identity.auth.verification_requested` | omnichannel Lambda |
| `omnichannel-on-activated` | `identity.user.activated` | omnichannel Lambda |
| `omnichannel-on-subscription` | `projects.subscription.created` | omnichannel Lambda |
| `omnichannel-on-sub-approved` | `projects.subscription.approved` | omnichannel Lambda |
| `omnichannel-on-sub-rejected` | `projects.subscription.rejected` | omnichannel Lambda |
| `projects-on-user-deactivated` | `identity.user.deactivated` | projects-registry Lambda |

---

## 9. Cross-Cutting Concerns

### 9.1 Logging (structlog, all services)

Every service configures structlog in `src/infrastructure/logging.py`:

```python
def configure_logging(service_name: str, log_level: str = "INFO") -> None:
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
    )
```

Called at top of `src/main.py` before anything else: `configure_logging(settings.service_name)`.

**Log format** (CloudWatch-ready JSON):
```json
{ "event": "register_user.completed", "timestamp": "2026-02-24T10:00:00Z", "level": "info", "service": "ugsys-identity-manager", "correlation_id": "...", "user_id": "550e..." }
```

**NEVER log**: `password`, `token`, `access_token`, `refresh_token`, `api_key`, `secret`, `hashed_password`

### 9.2 Security Headers (all services)

`SecurityHeadersMiddleware` must set all of the following. Note `X-XSS-Protection` is intentionally disabled — CSP is the correct modern defense.

```python
# Core headers — every response
response.headers["X-Content-Type-Options"] = "nosniff"
response.headers["X-Frame-Options"] = "DENY"
response.headers["X-XSS-Protection"] = "0"          # Disabled — CSP supersedes this
response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload"
response.headers["Permissions-Policy"] = (
    "accelerometer=(), camera=(), geolocation=(), microphone=(), payment=(), usb=()"
)
response.headers["Cross-Origin-Opener-Policy"] = "same-origin"
response.headers["Cross-Origin-Resource-Policy"] = "same-origin"

# CSP — strict for API endpoints, permissive only for /docs
if request.url.path.startswith("/docs") or request.url.path in {"/redoc", "/openapi.json"}:
    response.headers["Content-Security-Policy"] = (
        "default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; "
        "style-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; frame-ancestors 'none'"
    )
else:
    response.headers["Content-Security-Policy"] = (
        "default-src 'none'; frame-ancestors 'none'; base-uri 'none'; form-action 'none'"
    )

# Prevent caching of API responses
if request.url.path.startswith("/api/"):
    response.headers["Cache-Control"] = "no-store, no-cache, must-revalidate"
    response.headers["Pragma"] = "no-cache"
    response.headers["Expires"] = "0"

# Remove Server header — prevents technology fingerprinting
if "server" in response.headers:
    del response.headers["server"]
```
```

### 9.3 Input Validation (all services)

- All inputs validated via Pydantic v2 before reaching application layer
- `full_name` and all free-text fields: `html.escape(v.strip())`
- Request body size limit: 1 MB (middleware)
- Never trust client-provided user IDs — always use `sub` from JWT

### 9.4 IDOR Prevention Pattern

```python
# Every resource fetch must check ownership
async def get_resource(resource_id: UUID, requester_id: str, is_admin: bool) -> Resource:
    resource = await self._repo.find_by_id(resource_id)
    if resource is None:
        raise NotFoundError(
            message=f"Resource {resource_id} not found",
            user_message="Resource not found",
        )
    if not is_admin and str(resource.owner_id) != requester_id:
        raise AuthorizationError(
            message=f"User {requester_id} attempted access to resource owned by {resource.owner_id}",
            user_message="Access denied",
        )
    return resource
```

### 9.5 Config Pattern (all services)

```python
# src/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    service_name: str = "ugsys-<service>"
    environment: str = "dev"
    aws_region: str = "us-east-1"
    dynamodb_table_prefix: str = "ugsys"
    event_bus_name: str = "ugsys-platform-bus"  # actual CDK name
    log_level: str = "INFO"
    jwt_public_key: str          # RS256 public key PEM
    rate_limit_per_minute: int = 60

    class Config:
        env_file = ".env"
        case_sensitive = False

settings = Settings()
```

### 9.6 Application Factory + Lifespan Pattern (all services)

`configure_logging()` is called at **module level** — before any other import side-effects.
`create_app()` is the single composition root — all wiring happens inside it.

```python
# src/main.py
import structlog
from contextlib import asynccontextmanager
from collections.abc import AsyncGenerator
from fastapi import FastAPI

from src.config import settings
from src.infrastructure.logging import configure_logging
from src.domain.exceptions import DomainError
from src.presentation.middleware.correlation_id import CorrelationIdMiddleware
from src.presentation.middleware.security_headers import SecurityHeadersMiddleware
from src.presentation.middleware.rate_limiting import RateLimitMiddleware
from src.presentation.middleware.exception_handler import (
    domain_exception_handler, unhandled_exception_handler,
)
from src.presentation.api.v1 import health  # + domain routers

# ① configure logging FIRST — before any other code runs
configure_logging(settings.service_name, settings.log_level)
logger = structlog.get_logger()


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    logger.info("startup.begin", service=settings.service_name, version=settings.version)
    # wire infrastructure dependencies here
    yield
    logger.info("shutdown.complete", service=settings.service_name)


def create_app() -> FastAPI:
    """Single composition root — all wiring in one place."""
    app = FastAPI(
        title=settings.service_name,
        version=settings.version,
        docs_url="/docs" if settings.environment != "prod" else None,
        lifespan=lifespan,
    )
    # Middleware — last added = first executed
    app.add_middleware(CorrelationIdMiddleware)
    app.add_middleware(SecurityHeadersMiddleware)
    app.add_middleware(RateLimitMiddleware, requests_per_minute=settings.rate_limit_per_minute)
    # Exception handlers registered before routers
    app.add_exception_handler(DomainError, domain_exception_handler)
    app.add_exception_handler(Exception, unhandled_exception_handler)
    # Domain routers
    app.include_router(health.router)
    # app.include_router(auth.router, prefix="/api/v1")
    return app


app = create_app()
```

### 9.7 CI/CD Gates (all services)

#### `ci.yml` — every push to `feature/**`, `fix/**` and every PR to `main`

| Job | Command / Tool | Blocks merge |
|-----|---------------|-------------|
| `lint` | `uv run ruff check src/ tests/ && uv run ruff format --check src/ tests/` | Yes |
| `typecheck` | `uv run mypy src/ --strict` | Yes |
| `test` | `uv run pytest tests/unit/ -v --cov=src --cov-fail-under=80` | Yes |
| `sast` | `uv run bandit -r src/ -c pyproject.toml -ll` | Yes |
| `sast-semgrep` | Semgrep action — `p/python`, `p/security-audit`, `p/owasp-top-ten`, `p/secrets` | Yes |
| `dependency-scan` | `uv run safety check` | Advisory only |
| `sbom` | CycloneDX → Trivy scan (CRITICAL/HIGH, ignore-unfixed) | Yes |
| `secret-scan` | Gitleaks (`fetch-depth: 0`, every push) | Yes |
| `arch-guard` | grep domain/application layer imports | Yes |
| `notify-failure` | Slack `C0AE6QV0URH` via bot token | — |

**Why both Bandit and Semgrep**: Bandit is fast and Python-specific; Semgrep catches complex patterns (SQL injection, XSS, SSRF, unsafe deserialization) that Bandit misses. No single scanner catches everything.

**SBOM job** — generates a CycloneDX bill of materials then scans it with Trivy:
```yaml
- name: Generate SBOM
  run: |
    uv export --no-hashes --no-dev --frozen > requirements-frozen.txt
    uv tool run cyclonedx-py requirements requirements-frozen.txt -o sbom.json --of json

- name: Scan SBOM with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: sbom
    scan-ref: sbom.json
    exit-code: '1'
    severity: CRITICAL,HIGH
    ignore-unfixed: true
```
Upload `sbom.json` as artifact with 90-day retention.

#### `codeql.yml` — PR to `main` + weekly schedule (Monday 06:00 UTC)

```yaml
- uses: github/codeql-action/init@v3
  with:
    languages: python
    queries: +security-extended,security-and-quality
- uses: github/codeql-action/analyze@v3
```

Results upload to GitHub Security tab (SARIF). Does not block merge — findings reviewed manually.

#### `security-scan.yml` (DAST) — triggered post-deploy to `main`, also `workflow_dispatch`

Runs against the live deployed environment URL after `deploy.yml` completes.

| Scanner | What it tests | Blocks |
|---------|--------------|--------|
| OWASP ZAP baseline | Headers, XSS, SQLi, CORS, CSRF, cookie flags, error disclosure | Yes (FAIL rules) |
| Nuclei | CVEs, misconfigurations, default logins, exposures | Yes (critical findings) |

ZAP rules: FAIL on XSS, SQLi, CORS misconfiguration, missing HSTS, cookie without SameSite, PII disclosure. WARN on X-Frame-Options. IGNORE on CSRF double-submit cookie (JS-readable by design). Slack notification on any finding.

#### Git hooks — install once per clone via `just install-hooks`

```
scripts/
├── hooks/
│   ├── pre-commit   # blocks commits to main; ruff lint + format
│   └── pre-push     # blocks push to main; mypy + unit tests
└── install-hooks.sh # cp hooks → .git/hooks/ + chmod +x
```

**`pre-commit`** (fast — runs on every commit):
```bash
# Block direct commits to main
if [[ "$BRANCH" == "main" ]]; then exit 1; fi
# Must pass before commit is recorded
uv tool run ruff check src/ tests/
uv tool run ruff format --check src/ tests/
```

**`pre-push`** (slower — runs before `git push`):
```bash
# Block direct push to main
if [[ "$remote_ref" == "refs/heads/main" ]]; then exit 1; fi
# Catch regressions before they hit CI
uv run mypy src/ --strict
uv run pytest tests/unit/ -q
```

Install:
```bash
just install-hooks   # or: bash scripts/install-hooks.sh
```

Never bypass with `--no-verify`. If a hook fails, fix the code.

### 9.8 Ruff + Bandit Config (pyproject.toml, all services)

```toml
[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "S", "ANN", "RUF"]
ignore = ["S101"]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101", "S105", "S106", "ANN"]

[tool.bandit]
exclude_dirs = ["tests", ".venv"]
skips = ["B101"]
```

### 9.9 Shared Libs Usage

| Need | Library | Import |
|------|---------|--------|
| JWT validation | `ugsys-auth-client` | `from ugsys_auth_client import verify_token` |
| Structured logging | `ugsys-logging-lib` | `from ugsys_logging import configure_logging` |
| EventBridge publish | `ugsys-event-lib` | `from ugsys_event_lib import EventPublisher` |
| Test fixtures | `ugsys-testing-lib` | `from ugsys_testing import user_factory` |

Add to `pyproject.toml`:
```toml
[tool.uv.sources]
ugsys-auth-client = { git = "https://github.com/awscbba/ugsys-shared-libs", tag = "auth-client/v0.1.0", subdirectory = "auth-client" }
```

### 9.10 Paginated Response Shape (Registry — used by admin endpoints)

Registry admin endpoints that return lists use `PaginatedResponse` from `pagination.py`:

```json
{
  "items": [ { ...PersonResponse... } ],
  "pagination": {
    "currentPage": 1,
    "pageSize": 10,
    "totalItems": 42,
    "totalPages": 5,
    "hasNextPage": true,
    "hasPreviousPage": false,
    "startIndex": 1,
    "endIndex": 10
  }
}
```

Query parameters for paginated endpoints: `page` (1-based), `pageSize` (1–100), `sortBy`, `sortDirection` (`asc`|`desc`), `search`, `isAdmin`, `isActive`, `emailVerified`.

> **ugsys standard**: The ugsys envelope wraps this in `{ "data": { "items": [...], "pagination": {...} }, "meta": { "request_id": "..." } }`. The `pagination` object shape above is canonical for all paginated responses.

### 9.11 Domain Exception Hierarchy (all services)

Every service defines `src/domain/exceptions.py`. Never raise raw `Exception`, `ValueError`, or `HTTPException` from application or domain layers.

```python
# src/domain/exceptions.py
from dataclasses import dataclass, field
from typing import Any

@dataclass
class DomainError(Exception):
    message: str                              # internal — logs only, NEVER sent to client
    user_message: str = "An error occurred"  # safe — returned in API response
    error_code: str = "INTERNAL_ERROR"       # machine-readable
    additional_data: dict[str, Any] = field(default_factory=dict)

    def __str__(self) -> str:
        return self.message

class ValidationError(DomainError):
    error_code: str = "VALIDATION_ERROR"     # HTTP 422

class NotFoundError(DomainError):
    error_code: str = "NOT_FOUND"            # HTTP 404

class ConflictError(DomainError):
    error_code: str = "CONFLICT"             # HTTP 409

class AuthenticationError(DomainError):
    error_code: str = "AUTHENTICATION_FAILED"  # HTTP 401

class AuthorizationError(DomainError):
    error_code: str = "FORBIDDEN"            # HTTP 403

class AccountLockedError(DomainError):
    error_code: str = "ACCOUNT_LOCKED"       # HTTP 423

class RepositoryError(DomainError):
    error_code: str = "REPOSITORY_ERROR"     # HTTP 500 — never expose DB details

class ExternalServiceError(DomainError):
    error_code: str = "EXTERNAL_SERVICE_ERROR"  # HTTP 502
```

**Usage in application layer**:
```python
# ✅ message = internal detail (logs), user_message = safe client string
raise ConflictError(
    message=f"Email {cmd.email} already exists in DynamoDB users table",
    user_message="This email address is already in use",
    error_code="EMAIL_ALREADY_EXISTS",
)
```

**Rule**: `message` and `user_message` are NEVER the same string. `message` may contain IDs, DB details, stack context. `user_message` never does.

### 9.12 Error Response Envelope (all services)

All error responses use this shape (produced by `domain_exception_handler`):

```json
{
  "error": "EMAIL_ALREADY_EXISTS",
  "message": "This email address is already in use",
  "data": {}
}
```

The exception handler lives in `src/presentation/middleware/exception_handler.py`:

```python
STATUS_MAP = {
    ValidationError: 422, NotFoundError: 404, ConflictError: 409,
    AuthenticationError: 401, AuthorizationError: 403, AccountLockedError: 423,
}

async def domain_exception_handler(request: Request, exc: DomainError) -> JSONResponse:
    status = STATUS_MAP.get(type(exc), 500)
    logger.error("domain_error", error_code=exc.error_code,
                 message=exc.message, status=status, path=request.url.path)
    return JSONResponse(status_code=status, content={
        "error": exc.error_code,
        "message": exc.user_message,   # safe — never exc.message
        "data": exc.additional_data,
    })

async def unhandled_exception_handler(request: Request, exc: Exception) -> JSONResponse:
    logger.error("unhandled_exception", error=str(exc), path=request.url.path, exc_info=True)
    return JSONResponse(status_code=500, content={
        "error": "INTERNAL_ERROR", "message": "An unexpected error occurred",
    })
```

### 9.13 JWT Hardening (all services)

JWT validation is delegated to `ugsys-auth-client`. These are the invariants it must enforce — and what to verify when implementing or upgrading the lib.

**Algorithm restriction — checked BEFORE signature verification:**
```python
unverified_header = jwt.get_unverified_header(token)
alg = unverified_header.get("alg")
if alg not in ["RS256"]:          # Only RS256 — reject HS256 and "none"
    raise JWTError(f"Algorithm {alg} not allowed")
```
Checking the algorithm first prevents algorithm confusion attacks where an attacker signs a token with HS256 using the server's public key as the HMAC secret.

**Strict decode options:**
```python
claims = jwt.decode(
    token,
    public_key,
    algorithms=["RS256"],
    audience=settings.cognito_client_id,   # aud must match our client ID
    issuer=settings.cognito_issuer,        # iss must match our User Pool URL
    options={
        "verify_signature": True,
        "verify_exp": True,
        "verify_iat": True,
        "verify_aud": True,   # Never skip audience validation
        "require_exp": True,
        "require_iat": True,
    },
)
```

**Required claims — validated after decode:**
```python
REQUIRED_CLAIMS = ["sub", "exp", "iat", "iss"]
missing = [c for c in REQUIRED_CLAIMS if c not in claims]
if missing:
    raise JWTError(f"Missing required claims: {missing}")
```

**JWKS cache with TTL + forced refresh on key rotation:**
```python
async def _get_jwks(self, force_refresh: bool = False) -> dict:
    now = time.time()
    if not force_refresh and self._cache and (now - self._cache_time) < self._ttl:
        return self._cache
    # fetch from Cognito JWKS endpoint...
    # if key not found in cache → force_refresh=True once (handles key rotation)
```

**Never expose JWT errors to callers** — always return generic 401:
```python
except JWTError:
    raise AuthenticationError(
        message=f"JWT validation failed: {e}",   # internal — logs only
        user_message="Invalid or expired token",  # safe — returned to client
    )
```

### 9.14 CSRF Protection (browser-facing services)

Services that issue `httpOnly` cookies to browser clients (admin panel BFF, any service with a browser login flow) MUST implement CSRF protection using the **Double Submit Cookie** pattern with HMAC-signed tokens.

**How it works:**
1. Server generates a signed CSRF token and sets it in a `SameSite=Strict` cookie (NOT `httpOnly` — JS must read it)
2. Client reads the cookie and includes the same value in `X-CSRF-Token` header on every state-changing request
3. Server validates cookie token matches header token AND verifies the HMAC signature and TTL

**Protected methods**: `POST`, `PUT`, `PATCH`, `DELETE`
**Safe methods**: `GET`, `HEAD`, `OPTIONS` — no CSRF check needed

**Token structure**: `{random_hex}.{timestamp}.{hmac_signature[:16]}`

```python
# Validation — both checks must pass
cookie_token = request.cookies.get("csrf_token")
header_token = request.headers.get("X-CSRF-Token")

if not cookie_token or not header_token:
    return JSONResponse(status_code=403, content={"detail": "CSRF token missing"})

# Constant-time comparison prevents timing attacks
if not hmac.compare_digest(cookie_token, header_token):
    return JSONResponse(status_code=403, content={"detail": "CSRF token mismatch"})

# Verify signature and TTL (1 hour)
if not _validate_token_signature(cookie_token, secret_key):
    return JSONResponse(status_code=403, content={"detail": "CSRF token invalid"})
```

**Cookie settings for CSRF token:**
```python
response.set_cookie(
    key="csrf_token",
    value=token,
    max_age=3600,
    httponly=False,    # Must be readable by JavaScript
    secure=True,       # HTTPS only
    samesite="strict", # Prevent cross-origin requests
    path="/",
)
```

**Note**: Pure API services (no browser cookie auth) do not need CSRF middleware. The `ugsys-auth-client` Bearer token flow is not vulnerable to CSRF.

### 9.15 Cookie Security (browser-facing services)

When issuing authentication cookies (admin panel BFF, any browser login flow):

```python
# Access token — httpOnly prevents XSS token theft
response.set_cookie(
    key="access_token",
    value=access_token,
    max_age=3600,          # 1 hour
    httponly=True,         # Not accessible via JavaScript
    secure=True,           # HTTPS only in production
    samesite="lax",        # CSRF protection + allows navigation
    path="/",
)

# Refresh token — same flags, longer TTL
response.set_cookie(
    key="refresh_token",
    value=refresh_token,
    max_age=30 * 24 * 3600,  # 30 days
    httponly=True,
    secure=True,
    samesite="lax",
    path="/",
)
```

**Logout — always delete cookies explicitly:**
```python
response.delete_cookie(key="access_token", path="/", httponly=True, samesite="lax")
response.delete_cookie(key="refresh_token", path="/", httponly=True, samesite="lax")
```

**Rule**: Never store tokens in `localStorage` or `sessionStorage` — they are accessible to any JavaScript on the page (XSS attack surface).

### 9.16 CORS Policy (all services)

Every service must configure CORS explicitly. Never use wildcard `*` with credentials.

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,   # explicit list — never ["*"]
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID", "X-CSRF-Token"],
)
```

**`settings.allowed_origins`** must be an explicit list per environment:
```python
# dev
allowed_origins: list[str] = ["http://localhost:3000", "http://localhost:4321"]
# prod
allowed_origins: list[str] = ["https://admin.cbba.cloud.org.bo"]
```

**Rules:**
- Reject arbitrary origins — never reflect `Origin` header back
- Reject `null` origin — used by sandboxed iframes and local files
- No `Access-Control-Allow-Origin: *` when `Access-Control-Allow-Credentials: true`

### 9.17 Security Monitoring (all services)

Every service's CloudWatch log group must have these metric filters and alarms. Define them in the CDK stack for the service.

**Metric filters** (on the service log group):

| Filter name | Pattern | Metric |
|-------------|---------|--------|
| `ErrorCount` | `{ $.level = "error" }` | `ServiceErrors` |
| `CriticalCount` | `{ $.level = "critical" }` | `ServiceCriticals` |
| `SlowOperations` | `{ $.duration_ms > 1000 }` | `SlowOps` |
| `AuthFailures` | `{ $.event = "auth.*failed" }` | `AuthFailures` |
| `AccessDenied` | `{ $.error_code = "FORBIDDEN" }` | `AccessDenied` |

**Alarms** (SNS → Slack `C0AE6QV0URH`):

| Alarm | Threshold | Period |
|-------|-----------|--------|
| Error rate | ≥ 10 errors | 5 min |
| Critical errors | ≥ 1 critical | 1 min |
| Slow operations (p95) | > 1000 ms | 5 min |
| Auth failures (brute force) | ≥ 100 failures | 1 hour |
| Access denied spikes | ≥ 30 denials | 5 min |

**Security metrics to track** (CloudWatch dashboard per service):

| Metric | Target | Alert |
|--------|--------|-------|
| Critical vulnerabilities (Dependabot) | 0 | Yes |
| Failed auth attempts / hour | < 100 | Yes |
| Access denied / 5 min | < 30 | Yes |
| Error rate / 5 min | < 10 | Yes |
| API latency p95 | < 1000 ms | Yes |

All log groups must have KMS encryption (`CKV_AWS_158`) — enforced by Checkov in CI.

### 9.18 Repository Pattern (all services)

Every `ugsys-*` service MUST implement the repository pattern. This is the authoritative contract — see `.kiro/steering/repository-pattern.md` for full implementation guide with code examples.

#### Port definition (domain layer)

All outbound port interfaces live in `src/domain/repositories/` as ABCs. One file per aggregate root.

```python
# src/domain/repositories/project_repository.py
from abc import ABC, abstractmethod
from src.domain.entities.project import Project

class ProjectRepository(ABC):
    @abstractmethod
    async def save(self, project: Project) -> Project: ...
    @abstractmethod
    async def find_by_id(self, project_id: str) -> Project | None: ...
    @abstractmethod
    async def update(self, project: Project) -> Project: ...
    @abstractmethod
    async def delete(self, project_id: str) -> None: ...
    @abstractmethod
    async def list_paginated(self, page: int, page_size: int, status_filter: str | None, category_filter: str | None) -> tuple[list[Project], int]: ...
    @abstractmethod
    async def list_public(self, limit: int) -> list[Project]: ...
```

Non-persistence ports (`EventPublisher`, `IdentityClient`, etc.) also live in `src/domain/repositories/`.

#### Concrete implementation (infrastructure layer)

| Port ABC | Concrete class | Location |
|----------|---------------|----------|
| `XxxRepository` | `DynamoDBXxxRepository` | `src/infrastructure/persistence/` |
| `EventPublisher` | `EventBridgePublisher` | `src/infrastructure/messaging/` |
| `IdentityClient` | `IdentityManagerClient` | `src/infrastructure/adapters/` |

#### Mandatory implementation rules

1. Every `boto3`/`aioboto3` call MUST be wrapped in `try/except ClientError`
2. `_raise_repository_error(operation, e)` logs full `ClientError` internally, raises `RepositoryError` with safe `user_message` — never exposes DynamoDB error details to callers
3. `_to_item(entity) -> dict` and `_from_item(item) -> entity` are private serialization methods on every DynamoDB repository
4. `_from_item` MUST use `.get()` with safe defaults for all optional fields (backward compatibility)
5. `_to_item` MUST omit optional fields when `None` — never write `{"NULL": True}`
6. `ConditionalCheckFailedException` on `save` → `RepositoryError`; on `update` → `NotFoundError`
7. Repositories wired ONLY in `src/main.py` `create_app()` — no global singletons

#### Testing rules

- Unit tests: `AsyncMock(spec=XxxRepository)` — NEVER mock `boto3` directly
- Integration tests: `moto` `mock_aws` — NEVER call real AWS
- Integration tests MUST cover: round-trip serialization, backward-compatible deserialization, `ClientError` → `RepositoryError` wrapping

### 9.19 Outbox Pattern (reliable event delivery)

Solves the dual-write problem: when a service must persist state AND publish an event atomically. Write the event to an outbox table in the SAME DynamoDB transaction as the business write. A separate delivery process reads the outbox and publishes to EventBridge.

See `.kiro/steering/enterprise-patterns.md` Section 10 for full implementation guide with code examples.

#### When to use

| Scenario | Approach |
|----------|----------|
| Event loss is acceptable (analytics, notifications) | Log-and-continue |
| Event loss causes data inconsistency | Outbox |
| Event triggers financial or compliance actions | Outbox |
| Event is consumed by another service to create/update its own state | Outbox |

#### Outbox table schema

```
Table: ugsys-outbox-{service}-{env}
PK: OUTBOX#{ulid}
SK: EVENT
GSI: StatusIndex (PK=status, SK=created_at)

Required attributes: id, aggregate_type, aggregate_id, event_type, payload, created_at, published_at, retry_count, status
```

#### Mandatory rules

1. Outbox write MUST be in the same `TransactWriteItems` as the business write
2. Delivery process runs on EventBridge Scheduler (1-minute interval)
3. Events with `retry_count >= 5` → status `failed` + CloudWatch alarm
4. Published events older than 7 days → cleanup (DynamoDB TTL)
5. Consumers MUST be idempotent — outbox may re-deliver

### 9.20 Unit of Work (transactional consistency)

Groups multiple repository operations into a single atomic DynamoDB `TransactWriteItems` (up to 100 operations).

See `.kiro/steering/enterprise-patterns.md` Section 11 for full implementation guide with code examples.

#### When to use

- Approving a subscription AND incrementing participant count
- Creating a form submission AND updating submission count
- Any multi-aggregate write that must be all-or-nothing

#### Port definition

```python
# src/domain/repositories/unit_of_work.py
class UnitOfWork(ABC):
    @abstractmethod
    async def execute(self, operations: list[TransactionalOperation]) -> None: ...
```

#### Mandatory rules

1. `UnitOfWork` ABC lives in `src/domain/repositories/`
2. `DynamoDBUnitOfWork` lives in `src/infrastructure/persistence/`
3. Wired in `main.py` alongside repositories — same DynamoDB client
4. Never mix transactional and non-transactional writes for the same aggregate in the same use case
5. DynamoDB transaction limit: 100 operations — redesign aggregates if exceeded
6. Combines naturally with Outbox Pattern: outbox write is another operation in the transaction

### 9.21 Circuit Breaker (external service resilience)

Wraps calls to external services and fast-fails after repeated failures. Prevents cascade failures and gives downstream services time to recover.

See `.kiro/steering/enterprise-patterns.md` Section 12 for full implementation guide with code examples.

#### State machine

```
CLOSED ──[N failures]──→ OPEN ──[cooldown]──→ HALF_OPEN ──[success]──→ CLOSED
                                                         ──[failure]──→ OPEN
```

#### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `failure_threshold` | 5 | Consecutive failures before opening |
| `cooldown_seconds` | 30 | Time in OPEN before probing |
| `half_open_max_calls` | 1 | Probe calls in HALF_OPEN |

#### Mandatory rules

1. One circuit breaker instance per external service — wired in `main.py`
2. In-memory implementation (Lambda cold starts reset — acceptable for serverless)
3. Log every state transition with structlog
4. Never use for DynamoDB calls — those use repository error wrapping
5. Services that MUST use circuit breaker: `IdentityManagerClient`, `EventBridgePublisher` (when not using outbox)
6. When circuit is OPEN, raise `ExternalServiceError` with safe `user_message`

### 9.22 Specification / Query Object (composable filters)

Encapsulates query criteria as first-class objects. Prevents `list_paginated()` from growing unbounded parameters.

See `.kiro/steering/enterprise-patterns.md` Section 13 for full implementation guide with code examples.

#### Structure

```python
# src/application/queries/project_queries.py
@dataclass(frozen=True)
class ProjectListQuery:
    page: int = 1
    page_size: int = 20
    status: str | None = None
    category: str | None = None
    owner_id: str | None = None
    sort_by: str = "created_at"
    sort_order: str = "desc"
```

#### Mandatory rules

1. One query object per aggregate list operation — `ProjectListQuery`, `SubscriptionListQuery`, etc.
2. Query objects are frozen dataclasses — immutable after creation
3. Query objects live in `src/application/queries/`
4. Repository port accepts the query object — infrastructure translates to DynamoDB operations
5. Adding a new filter = add field to query object + update `_build_filter_expression()` — no signature changes elsewhere
6. Admin and public endpoints can share the same query object with different defaults

---

## 10. Implementation Gap Tracker

This section tracks every known gap between the spec and current implementation. Update this as work is completed.

### 10.1 Identity Manager Gaps

| Gap | Priority | Phase |
|-----|----------|-------|
| Account lockout: hard enforcement after 5 failures (currently graceful fallback only) | P0 | 1 |
| Login does not enforce lockout (no 423 response) | P0 | 1 |
| Login does not set `last_login_at` | P0 | 1 |
| Login does not enforce `require_password_change` (no 403 with flag) | P0 | 1 |
| Login does not block `pending_verification` users | P0 | 1 |
| Token blacklist for logout (jti → DynamoDB TTL) — logout is client-side only | P0 | 1 |
| `POST /verify-email` endpoint missing | P0 | 1 |
| `POST /resend-verification` endpoint missing | P0 | 1 |
| Email verification token not generated on register | P0 | 1 |
| `identity.user.registered` event not published on register | P0 | 1 |
| JWT algorithm is HS256 in auth-client v0.1.0 — must migrate to RS256 | P0 | 1 |
| `POST /api/v1/auth/service-token` endpoint missing (needed by ServiceAuthClient S2S) | P0 | 1 |
| `event_bus_name` env var must be `ugsys-platform-bus` not `ugsys-event-bus` | P0 | 1 |
| `require_password_change` field missing from login response | P1 | 1 |
| `identity.auth.login_success/failed` events not published | P1 | 1 |
| `identity.auth.account_locked` event not published | P1 | 1 |
| `identity.auth.password_changed` event not published | P1 | 1 |
| `POST /users/{id}/suspend` endpoint missing | P1 | 1 |
| `POST /users/{id}/activate` endpoint missing | P1 | 1 |
| `POST /users/{id}/require-password-change` endpoint missing | P1 | 1 |
| Password strength validation missing (only min_length=8 checked) | P1 | 1 |
| `POST /api/v1/auth/validate-token` endpoint missing (needed by auth-client remote validation) | P1 | 1 |
| `AuditEventType` enum — all 22 event types with dotted string values (see Section 13.6) | P1 | 1 |
| Response envelope not standardized (returns raw dict, not `{ data, meta }`) | P2 | 1 |
| Pagination missing on `GET /users` | P2 | 1 |
| Granular permissions in JWT claims | P3 | 2 |
| Password history (last 5) | P3 | 2 |

### 10.2 User Profile Service Gaps

| Gap | Priority | Phase |
|-----|----------|-------|
| EventBridge consumer for `identity.user.registered` missing — profiles not auto-created | P0 | 1 |
| EventBridge consumer for `identity.user.deactivated` missing | P0 | 1 |
| EventBridge consumer for `identity.auth.password_changed` missing | P0 | 1 |
| `notification_preferences` field missing from UserProfile entity | P1 | 1 |
| `language` field missing from UserProfile entity | P1 | 1 |
| `timezone` field missing from UserProfile entity | P1 | 1 |
| `avatar_url` field missing from UserProfile entity | P1 | 1 |
| `bio` field missing from UserProfile entity | P2 | 1 |
| `display_name` field missing from UserProfile entity | P2 | 1 |
| `POST /profiles/{id}/avatar` endpoint missing (S3 upload) | P1 | 1 |
| `DELETE /profiles/{id}/avatar` endpoint missing | P2 | 1 |
| `GET /profiles` (admin list) endpoint missing | P2 | 1 |
| `DELETE /profiles/{id}` (admin delete) endpoint missing | P2 | 1 |
| Response envelope not standardized | P2 | 1 |
| `profile.updated` event not published on PATCH | P2 | 1 |

### 10.3 Projects Registry Gaps

| Gap | Priority | Phase |
|-----|----------|-------|
| Entire service not yet scaffolded | P0 | 2 |
| All endpoints from Registry must be ported with hexagonal architecture | P0 | 2 |
| DynamoDB tables must be created (projects, subscriptions, form-submissions) | P0 | 2 |
| Subscription approval workflow (pending → active/rejected) | P0 | 2 |
| `POST /api/v1/public/check-email` — check if email exists | P1 | 2 |
| `POST /api/v1/public/register` — no-auth account creation | P1 | 2 |
| `POST /api/v1/public/subscribe` — no-auth register + subscribe | P1 | 2 |
| `POST /api/v1/admin/users/bulk-action` — activate/deactivate/delete | P1 | 2 |
| `GET /api/v1/admin/dashboard` — basic dashboard data | P1 | 2 |
| EventBridge consumers for `identity.user.deactivated` | P1 | 2 |
| All events must be published via `ugsys-event-lib` | P1 | 2 |
| Image upload via presigned S3 URLs | P1 | 2 |
| Form schema validation on submission (`text`, `poll_single`, `poll_multiple`) | P1 | 2 |
| Admin notification email target must be configurable (not hardcoded) | P1 | 2 |
| Cascade delete: subscriptions deleted when person deleted | P1 | 2 |
| Cascade status update: subscriptions updated when project completed/cancelled | P1 | 2 |
| `GET /api/v1/admin/dashboard/enhanced` — enhanced dashboard | P2 | 2 |
| `GET /api/v1/admin/analytics` — full analytics breakdown | P2 | 2 |
| `GET /api/v1/admin/users/paginated` — pagination + filtering | P2 | 2 |

### 10.4 Omnichannel Service Gaps

| Gap | Priority | Phase |
|-----|----------|-------|
| Entire service not yet scaffolded | P0 | 3 |
| SES email adapter | P0 | 3 |
| Jinja2 template rendering | P0 | 3 |
| EventBridge consumers for all identity + projects events | P0 | 3 |
| Supported channels: email, sms, whatsapp, facebook, instagram, linkedin (NOT slack/telegram) | P0 | 3 |
| WhatsApp adapter: Meta Graph API v18.0, requires `WHATSAPP_ACCESS_TOKEN` + `WHATSAPP_PHONE_NUMBER_ID` | P0 | 3 |
| SMS adapter: AWS SNS (not Twilio) | P0 | 3 |
| Email templates must be in Spanish (verified from Registry email_service.py) | P0 | 3 |
| Message processing uses EventBridge (not Kinesis as in devsecops-poc) | P0 | 3 |
| Facebook/Instagram adapters: Meta Graph API (same credentials as WhatsApp) | P1 | 3 |
| Admin notification email must be configurable via env var | P1 | 3 |
| Retry logic (max 3 retries with exponential backoff) | P1 | 3 |
| LinkedIn adapter: LinkedIn API (separate credentials) | P2 | 3 |
| Delivery webhook handlers | P2 | 3 |
| CertificationSubmission feature: decision needed on service placement | P2 | 3 |

### 10.5 Mass Messaging Gaps

| Gap | Priority | Phase |
|-----|----------|-------|
| Entire service not yet scaffolded | P0 | 4 |
| Campaign CRUD | P0 | 4 |
| Audience builder with filter criteria | P0 | 4 |
| Campaign execution engine (fan-out to omnichannel) | P0 | 4 |
| Analytics tracking | P1 | 4 |
| Scheduled campaign execution (EventBridge Scheduler) | P1 | 4 |

---

## 11. Phase Roadmap

### Phase 1 — Identity Manager + User Profile (current)

**Goal**: Complete auth + profile foundation before any other service.

**Identity Manager tasks** (in order):
1. Add missing fields to `User` entity: `failed_login_attempts`, `account_locked_until`, `last_login_at`, `last_password_change`, `require_password_change`, `email_verification_token`, `email_verified_at`
2. Update DynamoDB adapter to persist new fields
3. Implement account lockout logic in `AuthService.authenticate()`
4. Implement `require_password_change` enforcement in login
5. Implement `pending_verification` block in login
6. Add `POST /logout` with token blacklist
7. Add `POST /verify-email` endpoint + service logic
8. Add `POST /resend-verification` endpoint + service logic
9. Add `POST /change-password` endpoint + service logic
10. Add password strength validator (uppercase, lowercase, digit, special char)
11. Publish all missing EventBridge events via `ugsys-event-lib`
12. Add `POST /users/{id}/suspend`, `/activate`, `/require-password-change`
13. Standardize response envelopes
14. Add pagination to `GET /users`

**User Profile tasks** (in order):
1. Add missing fields to `UserProfile` entity: `notification_preferences`, `language`, `timezone`, `avatar_url`, `bio`, `display_name`
2. Update DynamoDB adapter
3. Implement EventBridge consumer for `identity.user.registered` → auto-create profile
4. Implement EventBridge consumer for `identity.user.deactivated`
5. Implement EventBridge consumer for `identity.auth.password_changed`
6. Add `POST /profiles/{id}/avatar` with S3 upload
7. Add `GET /profiles` (admin list) + `DELETE /profiles/{id}`
8. Publish `profile.updated` event on PATCH
9. Standardize response envelopes

### Phase 2 — Projects Registry

**Goal**: Extract Registry monolith into `ugsys-projects-registry` with hexagonal architecture.

1. Scaffold service with canonical structure
2. Port all domain entities (Project, Subscription, FormSchema, FormSubmission, ProjectImage)
3. Implement DynamoDB adapters for all 3 tables
4. Port all endpoints from Registry `/v2/projects`, `/v2/subscriptions`, `/v2/form-submissions`
5. Implement subscription approval workflow
6. Implement EventBridge consumers + publishers
7. Implement S3 presigned URL for image upload
8. Migrate data from Registry PostgreSQL → DynamoDB

### Phase 3 — Omnichannel Service

**Goal**: Replace Registry's `email_service.py` with a proper multi-channel service.

1. Scaffold service
2. Implement SES email adapter
3. Implement Jinja2 template engine
4. Bootstrap built-in templates
5. Implement EventBridge consumers for all events
6. Add retry logic
7. Add SNS SMS adapter
8. Add WhatsApp, Slack, Telegram adapters

### Phase 4 — Mass Messaging + Admin Panel

**Goal**: Campaign orchestration and unified admin UI.

1. Scaffold mass-messaging service
2. Implement campaign CRUD + audience builder
3. Implement execution engine (fan-out via omnichannel)
4. Implement EventBridge Scheduler for scheduled campaigns
5. Implement analytics tracking
6. Scaffold admin panel (Astro + React)
7. Integrate all services into admin panel

---

*Last updated: 2026-02-24 — derived from Registry registry-api, ugsys-identity-manager, ugsys-user-profile-service source code audit.*


---

## 12. Verified Infrastructure & Shared Libs Reference

> This section was added after a full code audit of all source repos in session 2 (2026-02-24).
> All values here are verified from actual source code — not inferred.

### 12.1 Actual AWS Resource Names (from CDK stacks)

#### EventBridge

| Resource | Actual name |
|----------|-------------|
| Custom bus | `ugsys-platform-bus` |
| Bus archive | `ugsys-platform-bus-archive` (30-day retention) |
| Bus log group | `/ugsys/platform/event-bus` |
| Bus CFn export (name) | `UgsysPlatformBusName` |
| Bus CFn export (ARN) | `UgsysPlatformBusArn` |

> **Correction**: The bus is named `ugsys-platform-bus`, NOT `ugsys-event-bus` as previously assumed.
> All services must use `EVENT_BUS_NAME=ugsys-platform-bus` in their environment config.

#### KMS

| Resource | Actual name |
|----------|-------------|
| Key alias | `alias/ugsys-platform` |
| Key description | "Shared encryption key for all ugsys platform services" |
| Key rotation | Enabled |
| Removal policy | RETAIN (never deleted) |
| CFn export (ARN) | `UgsysPlatformKeyArn` |
| CFn export (ID) | `UgsysPlatformKeyId` |

The KMS key policy explicitly grants `logs.<region>.amazonaws.com` the following actions:
`kms:Encrypt`, `kms:Decrypt`, `kms:ReEncrypt*`, `kms:GenerateDataKey*`, `kms:DescribeKey`

#### Identity Manager Stack

| Resource | Actual name pattern |
|----------|---------------------|
| DynamoDB table | `ugsys-identity-manager-users-{env}` |
| Table PK | `pk` (String) |
| Table SK | `sk` (String) |
| GSI | `email-index` (PK = `email`) |
| Lambda function | `ugsys-identity-manager-{env}` |
| Lambda handler | `handler.handler` |
| Lambda runtime | Python 3.13 |
| Lambda memory | 512 MB |
| Lambda timeout | 30s |
| Lambda tracing | X-Ray ACTIVE |
| API Gateway | `ugsys-identity-manager-{env}` (HTTP API) |
| Log group | `/aws/lambda/ugsys-identity-manager-{env}` |
| IAM role | `ugsys-identity-manager-lambda-{env}` |
| CFn export (API URL) | `UgsysIdentityManagerApiUrl-{env}` |
| CFn export (table) | `UgsysIdentityManagerUsersTable-{env}` |

Lambda environment variables:
```
APP_ENV={env}
AWS_ACCOUNT_ID={account}
DYNAMODB_TABLE_NAME=ugsys-identity-manager-users-{env}
EVENT_BUS_NAME=ugsys-platform-bus
LOG_LEVEL=INFO
```

#### User Profile Service Stack

| Resource | Actual name pattern |
|----------|---------------------|
| DynamoDB table | `ugsys-user-profiles-{env}` |
| Table PK | `pk` (String) |
| Table SK | `sk` (String) |
| GSI | `email-index` (PK = `email`) |
| Lambda function | `ugsys-user-profile-service-{env}` |
| Lambda handler | `handler.handler` |
| Lambda runtime | Python 3.13 |
| Lambda memory | 512 MB |
| Lambda timeout | 30s |
| Lambda tracing | X-Ray ACTIVE |
| API Gateway | `ugsys-user-profile-service-{env}` (HTTP API) |
| Log group | `/aws/lambda/ugsys-user-profile-service-{env}` |
| IAM role | `ugsys-user-profile-service-lambda-{env}` |
| CFn export (API URL) | `UgsysUserProfileServiceApiUrl-{env}` |
| CFn export (table) | `UgsysUserProfilesTable-{env}` |

Lambda environment variables:
```
APP_ENV={env}
DYNAMODB_TABLE_PREFIX=ugsys
ENVIRONMENT={env}
EVENT_BUS_NAME=ugsys-platform-bus
LOG_LEVEL=INFO
```

#### CORS Policy (all API Gateways)

```
allow_origins: ["https://cbba.cloud.org.bo"]
allow_methods: ANY
allow_headers: ["Content-Type", "Authorization", "X-Request-ID"]
max_age: 1 day
```

#### GitHub OIDC Deploy Roles

All 8 repos have a scoped IAM role. Role name pattern: `ugsys-github-deploy-{repo}`.

| Repo | Role name |
|------|-----------|
| `ugsys-identity-manager` | `ugsys-github-deploy-ugsys-identity-manager` |
| `ugsys-user-profile-service` | `ugsys-github-deploy-ugsys-user-profile-service` |
| `ugsys-admin-panel` | `ugsys-github-deploy-ugsys-admin-panel` |
| `ugsys-projects-registry` | `ugsys-github-deploy-ugsys-projects-registry` |
| `ugsys-mass-messaging` | `ugsys-github-deploy-ugsys-mass-messaging` |
| `ugsys-omnichannel-service` | `ugsys-github-deploy-ugsys-omnichannel-service` |
| `ugsys-platform-infrastructure` | `ugsys-github-deploy-ugsys-platform-infrastructure` |
| `ugsys-shared-libs` | `ugsys-github-deploy-ugsys-shared-libs` |

OIDC trust conditions:
```
sub: repo:awscbba/{repo}:ref:refs/heads/main
     repo:awscbba/{repo}:environment:prod
aud: sts.amazonaws.com
```

---

### 12.2 Shared Libs API Surface (v0.1.0)

#### `ugsys-auth-client`

**`TokenPayload`** (Pydantic model — decoded JWT):
```python
class TokenPayload(BaseModel):
    sub: str          # user_id
    email: str
    roles: list[str]  # e.g. ["admin", "member"]
    is_admin: bool    # convenience flag
    type: str         # "access" | "refresh" | "service"
```

**`ServiceCredentials`** (Pydantic model):
```python
class ServiceCredentials(BaseModel):
    client_id: str
    client_secret: str
    identity_url: str
```

**`TokenValidator`**:
```python
# Local validation (fast path — uses shared secret)
validator = TokenValidator(jwt_secret=settings.jwt_secret, jwt_algorithm="HS256")
payload: TokenPayload | None = validator.validate(token)

# Remote validation fallback (calls /api/v1/auth/validate-token)
validator = TokenValidator(identity_url="https://api.cbba.cloud.org.bo/identity")
payload = await validator.validate_remote(token)
```

> **Note**: Current implementation uses HS256. The security standard requires RS256.
> This is a known gap — see Section 12.4.

**`make_auth_dependency(validator)`** — FastAPI dependency factory:
```python
validator = TokenValidator(jwt_secret=settings.jwt_secret)
get_current_user = make_auth_dependency(validator)

@router.get("/me")
async def me(user: Annotated[TokenPayload, Depends(get_current_user)]):
    ...
```

**`AuthMiddleware`** — ASGI middleware (attaches payload to `request.state.user`, does NOT block):
```python
app.add_middleware(AuthMiddleware, validator=validator)
```

**`ServiceAuthClient`** — S2S token acquisition with caching:
```python
client = ServiceAuthClient(ServiceCredentials(
    client_id="projects-registry",
    client_secret=settings.service_secret,
    identity_url="https://api.cbba.cloud.org.bo/identity",
))
headers = await client.get_headers()  # {"Authorization": "Bearer <token>"}
```
Token is cached until 60 seconds before expiry. Calls `POST /api/v1/auth/service-token`.

#### `ugsys-event-lib`

**`EventMetadata`** (Pydantic model):
```python
class EventMetadata(BaseModel):
    event_id: str          # UUID4, auto-generated
    event_version: str     # "1.0"
    source_service: str    # e.g. "identity-manager"
    correlation_id: str    # from X-Request-ID header
    timestamp: str         # ISO 8601 UTC, auto-generated
```

**`UgsysEvent`** (Pydantic model — the standard envelope):
```python
class UgsysEvent(BaseModel):
    detail_type: str   # e.g. "identity.user.created"
    source: str        # e.g. "ugsys.identity-manager"
    metadata: EventMetadata
    payload: dict[str, Any]

# Serialize to boto3 put_events format:
entry = event.to_eventbridge_entry(event_bus_name="ugsys-platform-bus")
```

**`EventPublisher`**:
```python
publisher = EventPublisher(
    event_bus_name="ugsys-platform-bus",
    source_service="identity-manager",
    region="us-east-1",
)
publisher.publish(event)           # single event
publisher.publish_batch(events)    # up to 10 per batch (EventBridge limit)
```

Both methods return `bool` (True = all succeeded). Batching is automatic — batches of 10.

**Correct event source naming** (verified from `UgsysEvent` usage):
```
source = "ugsys.{service-name}"
# Examples:
"ugsys.identity-manager"
"ugsys.projects-registry"
"ugsys.omnichannel"
"ugsys.mass-messaging"
"ugsys.user-profile"
```

---

### 12.3 Registry Monolith — Verified Business Rules

These rules are extracted from actual service code and must be preserved when porting to `ugsys-projects-registry`.

#### People / User Management

**Account deletion rules** (`PeopleService.delete_person`):
- Cannot delete a person with `active` or `pending` subscriptions → 422 with `BUSINESS_RULE_VIOLATION`
- On successful deletion: all orphaned subscriptions are also deleted (cascade)
- Requesting user ID is logged for audit trail

**Bulk action** (`AdminService.execute_bulk_action`):
- Valid actions: `activate`, `deactivate`, `delete`
- Returns per-user success/failure results (see Section 13.4 for full shape)

**Account status operations** (verified methods):
- `activate_person(person_id)` — sets account active
- `deactivate_person(person_id)` — sets account inactive
- `unlock_account(person_id)` — calls `activate_person` internally
- `update_admin_status(person_id, is_admin)` — legacy admin flag

**Pagination** (`list_people_paginated`):
- Parameters: `page`, `page_size`, `sort_by`, `sort_direction`, `search`, `filters`
- Returns `(list[PersonResponse], total_count)` tuple

#### RBAC (Registry model — to be superseded by identity-manager)

**Roles** (from `RoleType` enum in `rbac_service.py`):
- `GUEST` — unauthenticated / no roles found
- `USER` — default for authenticated users
- `ADMIN` — can manage users, projects, subscriptions
- `SUPER_ADMIN` — all permissions including system admin
- `AUDITOR` — read-only + audit log access

**Permission matrix** (from `AuthorizationService.ROLE_PERMISSIONS`):

| Permission | USER | ADMIN | SUPER_ADMIN | AUDITOR |
|------------|------|-------|-------------|---------|
| `user:read` | ✅ | ✅ | ✅ | ✅ |
| `user:create` | ❌ | ✅ | ✅ | ❌ |
| `user:update` | ❌ | ✅ | ✅ | ❌ |
| `user:delete` | ❌ | ❌ | ✅ | ❌ |
| `user:admin` | ❌ | ✅ | ✅ | ❌ |
| `project:read` | ✅ | ✅ | ✅ | ✅ |
| `project:create` | ❌ | ✅ | ✅ | ❌ |
| `project:update` | ❌ | ✅ | ✅ | ❌ |
| `project:delete` | ❌ | ❌ | ✅ | ❌ |
| `project:admin` | ❌ | ✅ | ✅ | ❌ |
| `subscription:read` | ✅ | ✅ | ✅ | ✅ |
| `subscription:create` | ✅ | ✅ | ✅ | ❌ |
| `subscription:update` | ✅ | ✅ | ✅ | ❌ |
| `subscription:delete` | ❌ | ✅ | ✅ | ❌ |
| `subscription:admin` | ❌ | ✅ | ✅ | ❌ |
| `system:admin` | ❌ | ❌ | ✅ | ❌ |
| `system:audit` | ❌ | ❌ | ✅ | ✅ |

**Account lockout** (from `AuthorizationService`):
- 5 failed login attempts within 1 hour → account locked for 30 minutes
- `record_failed_login(user_id)` — call on every failed login
- `clear_failed_attempts(user_id)` — call on successful login
- `is_account_locked(user_id)` — check before processing login

> **Note**: This lockout logic is in-memory in Registry. In `ugsys-identity-manager` it must be persisted to DynamoDB (`failed_login_attempts` + `account_locked_until` fields on the User entity).

#### Projects Business Rules

**Project creation** (`ProjectsService.create_project`):
- `max_participants` must be > 0
- `end_date` must be >= `start_date` (date comparison, strips time component)

**Project status cascade** (`_update_subscription_statuses`):
- When project status → `completed` or `cancelled`: all `active` subscriptions are updated to match

**Dynamic forms** (`validate_form_schema`):
- Field IDs must be unique within the form
- Validation raises `ValueError` on duplicate IDs

#### Email Templates (Registry `EmailService` — source for Omnichannel)

The following email types are implemented in Registry and must be ported to `ugsys-omnichannel-service` as Jinja2 templates:

| Template name | Trigger | Subject (Spanish) |
|---------------|---------|-------------------|
| `password-reset` | Forgot password | "Restablecimiento de Contraseña - AWS User Group Cochabamba" |
| `subscription-welcome` | Subscription approved/active | "¡Bienvenido al proyecto {project_name}!" |
| `subscription-pending` | New subscription (pending approval) | "Suscripción Pendiente - {project_name}" |
| `subscription-notification` | Admin notification of new subscription | "Nueva Suscripción Pendiente - {project_name}" |
| `subscription-status-update` | Generic status change | "Actualización de Suscripción - {project_name}" |

**Email sender**: `"AWS User Group Cochabamba <{from_email}>"` (from `SES_FROM_EMAIL` env var)
**Admin notification target**: `admin@cbba.cloud.org.bo` (hardcoded in Registry — must be made configurable)
**Frontend URL**: from `FRONTEND_URL` env var (used in email CTAs)

**Password reset link format**: `{frontend_url}/reset-password?token={reset_token}`
**Token expiry**: 1 hour

#### Public Endpoints (no auth required)

Registry exposes two public endpoints under `/v2/public/`:

**`POST /v2/public/register`** — create account without project:
```json
// Request
{
  "email": "user@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "password": "SecurePass123!",
  "phone": "+591 70000000",
  "dateOfBirth": "1990-01-15",
  "address": { "street": "", "city": "", "state": "", "postalCode": "", "country": "" }
}
// Response 201
{ "personId": "...", "email": "...", "firstName": "...", "lastName": "...", "emailSent": true, "message": "Account created successfully! You can now login with your credentials." }
// Errors: 400 if email already exists
```

**`POST /v2/public/subscribe`** — register + subscribe in one call:
```json
// Request
{
  "email": "user@example.com",
  "projectId": "01JXXX",
  "firstName": "John",
  "lastName": "Doe",
  "phone": "+591 70000000",
  "dateOfBirth": "1990-01-15",
  "address": { ... }
}
// Response 201
{ "subscription": { ... }, "personCreated": true, "emailSent": true, "message": "Successfully subscribed to the project! Check your email for next steps." }
// Business rules:
//   - If person exists: check for duplicate subscription (409 if exists), create PENDING subscription
//   - If person does not exist: create person with temp password, create PENDING subscription
//   - ALL public subscriptions start as PENDING (admin approval required)
//   - Sends pending-approval email to subscriber
//   - Sends admin notification email to admin@cbba.cloud.org.bo
//   - Race condition handled: retry on duplicate email error
```

> **Migration note**: These public endpoints must be ported to `ugsys-projects-registry` as
> `POST /api/v1/public/register` and `POST /api/v1/public/subscribe`.
> The person creation part should call `ugsys-identity-manager` via S2S token.

#### Image Upload (Registry `S3ImageService`)

**`POST /v2/images/upload-url`** — presigned S3 upload URL:
```json
// Request
{ "filename": "workshop.jpg", "content_type": "image/jpeg", "file_size": 2048000 }
// Response 200
{ "upload_url": "https://s3.amazonaws.com/...", "image_id": "<UUID4>", "cloud_front_url": "https://cdn.cbba.cloud.org.bo/..." }
// Constraints: max 10MB, JPEG/PNG/GIF/WebP only
// CloudFront URL format: https://cdn.cbba.cloud.org.bo/{filename}
```

---

### 12.4 Registry Config — Actual Table Names

From `Registry/registry-api/src/core/config.py`:

| Config key | Env var | Default |
|------------|---------|---------|
| `people_table` | `PEOPLE_TABLE_V2_NAME` | `PeopleTableV2` |
| `projects_table` | `PROJECTS_TABLE_V2_NAME` | `ProjectsTableV2` |
| `subscriptions_table` | `SUBSCRIPTIONS_TABLE_V2_NAME` | `SubscriptionsTableV2` |
| `from_email` | `SES_FROM_EMAIL` | `noreply@example.com` |
| `frontend_url` | `FRONTEND_URL` | `http://localhost:3000` |
| `jwt_algorithm` | — | `HS256` (hardcoded) |
| `access_token_expire_hours` | — | `48` |
| `refresh_token_expire_days` | — | `30` |

> **Security gap**: Registry uses HS256. All ugsys services must use RS256 per security standards.

---

### 12.5 Omnichannel — Verified Channel Adapters (from devsecops-poc)

The devsecops-poc worker implements 6 channel adapters. These are the source of truth for the omnichannel service implementation.

#### Supported Channels (`ChannelType` enum)

```python
class ChannelType(str, Enum):
    WHATSAPP = "whatsapp"
    FACEBOOK = "facebook"
    INSTAGRAM = "instagram"
    LINKEDIN = "linkedin"
    EMAIL = "email"
    SMS = "sms"
```

> **Note**: Slack and Telegram are NOT in the devsecops-poc `ChannelType`. They were in the original spec as future channels. The 6 verified channels are: email, sms, whatsapp, facebook, instagram, linkedin.

#### Email Adapter (`EmailGateway`)

- Provider: AWS SES via `aiobotocore`
- `recipient_id`: email address
- Subject: hardcoded `"New Message"` in poc — omnichannel must use template subject
- Supports optional `media_url` (embedded as `<img>` in HTML body)
- Returns `DeliveryResult(success=bool, external_id=ses_message_id)`

#### WhatsApp Adapter (`WhatsAppGateway`)

- Provider: Meta Graph API v18.0
- Base URL: `https://graph.facebook.com/v18.0`
- Auth: Bearer token (`WHATSAPP_ACCESS_TOKEN`)
- Config: `phone_number_id` (from `WHATSAPP_PHONE_NUMBER_ID`)
- Text message payload: `{ "messaging_product": "whatsapp", "to": recipient, "type": "text", "text": { "body": content } }`
- Media message payload: `{ "type": "image", "image": { "link": media_url, "caption": content } }`
- Returns `DeliveryResult(success=bool, external_id=whatsapp_message_id)`

#### Other Adapters (files exist, same pattern)

| Channel | File | Provider |
|---------|------|----------|
| Facebook | `channels/facebook.py` | Meta Graph API |
| Instagram | `channels/instagram.py` | Meta Graph API |
| LinkedIn | `channels/linkedin.py` | LinkedIn API |
| SMS | `channels/sms.py` | AWS SNS |

#### Message Processing Flow (devsecops-poc worker)

```
Kinesis stream → consumer.py → MessageProcessor.process_scheduled_message()
  → MessageRepository.get_by_id()
  → MessageRepository.update_status("processing")
  → MessageDeliveryService.deliver(content, channels, media_url)
    → ChannelGateway.send() per channel
  → MessageRepository.mark_channel_delivered/failed()
  → MessageRepository.update_status("delivered" | "partial" | "failed")
```

> **Architecture note**: devsecops-poc uses Kinesis for message queuing. The ugsys omnichannel service will use EventBridge instead (per platform decision). The processing logic is the same.

#### Certification Submission System (devsecops-poc — separate feature)

This is a distinct feature in devsecops-poc for announcing AWS certifications on social media. It is NOT part of the core omnichannel messaging flow but may be incorporated into the omnichannel service as a specialized use case.

**`CertificationTypeEnum`** — 12 AWS certification types:
`cloud-practitioner`, `solutions-architect-associate`, `solutions-architect-professional`,
`developer-associate`, `sysops-administrator-associate`, `devops-engineer-professional`,
`database-specialty`, `security-specialty`, `machine-learning-specialty`,
`data-analytics-specialty`, `advanced-networking-specialty`, `sap-specialty`

**`CertificationLevel`**: `foundational` | `associate` | `professional` | `specialty`

**`CertificationSubmission`** entity:
```python
@dataclass
class CertificationSubmission:
    id: UUID
    member_name: str
    certification_type: CertificationTypeEnum
    certification_date: datetime
    channels: list[ChannelType]   # which social channels to post to
    user_id: str
    status: SubmissionStatus      # scheduled | processing | delivered | partially_delivered | failed
    photo_url: str | None
    linkedin_url: str | None
    personal_message: str | None
    deliveries: list[CertificationDelivery]
```

**Auto-generated post content** (`generate_post_content()`):
```
🎉 Congratulations to {member_name}! 🎉

{member_name} has just earned the AWS {cert_name} certification!

"{personal_message}"

Welcome to the club of AWS certified professionals! 🚀

#AWSCertified #{hashtag} #CloudCommunity #AWSCommunity
```

> **Decision needed**: Should `CertificationSubmission` be a first-class entity in `ugsys-omnichannel-service`, or a separate `ugsys-certifications-service`? Currently unresolved — add to Gap Tracker.

---

### 12.6 Registry App Startup — Middleware Order

From `Registry/registry-api/src/app.py` (`create_app()`):

Middleware is added in this order (FastAPI processes in reverse — last added = first executed):
1. `CORSMiddleware` — added first, executed last (outermost)
2. `SecurityHeadersMiddleware`
3. `RateLimitingMiddleware` (100 req/min)
4. `EnterpriseMiddleware`
5. `InputValidationMiddleware`
6. `AuthorizationMiddleware`
7. `AuthenticationMiddleware` — added last, executed first (innermost)

> **Note**: Registry uses `allow_origins=["*"]` (all origins). ugsys services must restrict to `["https://cbba.cloud.org.bo"]` per the CDK CORS config.

Routers registered:
- `/v2/people` — `people_router`
- `/v2/projects` — `projects_router`
- `/v2/subscriptions` — `subscriptions_router`
- `/auth` — `auth_router`
- `/v2/admin` — `admin_router`
- `/v2/public` — `public_router`
- `/v2/form-submissions` — `form_submissions_router`
- `/v2/images` — `image_upload_router`

---

### 12.7 Scaffold Status of Pending Services

| Service | Repo status | Has src/ | Has CDK stack |
|---------|-------------|----------|---------------|
| `ugsys-projects-registry` | Empty (LICENSE only) | ❌ | ❌ |
| `ugsys-omnichannel-service` | Has README only | ❌ | ❌ |
| `ugsys-mass-messaging` | Empty (LICENSE only) | ❌ | ❌ |

All three services need full scaffold before Phase 2/3/4 work begins.

---

### 12.8 Gap Tracker — Audit Additions

> All gaps identified in this session have been merged into Section 10 (10.1, 10.2, 10.3, 10.4). This subsection is retained for traceability only.

---

*Section 12 added: 2026-02-24 — verified from full source code audit of Registry services, devsecops-poc worker/channels, ugsys-platform-infrastructure CDK stacks, and ugsys-shared-libs v0.1.0.*


---

## Section 13 — Registry Deep Audit: Admin API, Audit Events, Auth System & Subscription Workflow

> **Reference Appendix** — This section is a reference appendix containing the raw audit findings from the Registry source code. All canonical information (gaps, entity shapes, endpoint contracts, business rules) has been integrated into Sections 2–12 above. Use this section for traceability and historical context only.

*Added: 2026-02-24 — verified from exhaustive source read: `admin_router.py`, `admin_service.py`, `audit_logger.py`, `form_submission_service.py`, `auth_router.py`, `AUTHENTICATION_SYSTEM.md`, `V2_API_ENDPOINTS.md`, `ENHANCED_SUBSCRIPTION_WORKFLOW.md`*

---

### 13.1 Registry Actual Production Coordinates

| Item | Value |
|------|-------|
| Base URL | `https://2t9blvt2c1.execute-api.us-east-1.amazonaws.com/prod` |
| API version prefix | `/v2/` |
| Auth algorithm | HS256 (Registry) — **ugsys MUST use RS256** |
| Access token TTL | 3600s (1h) |
| Refresh token TTL | 604800s (7d) |

> **Envelope divergence**: Registry v2 uses `{ "success": bool, "version": "v2", "timestamp": "...", "data": [...], "count": N, "metadata": { "total_count": N } }`. The ugsys standard envelope is `{ "data": {...}, "meta": { "page": N, "total": N } }`. Projects-registry (Phase 2) MUST adopt the ugsys envelope — do NOT copy the Registry envelope.

---

### 13.2 Registry Auth Endpoints (source: `AUTHENTICATION_SYSTEM.md`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/auth/login` | None | Returns `access_token`, `refresh_token`, `expires_in`, `user`, `require_password_change` |
| `GET` | `/auth/me` | Bearer | Returns current user object |
| `POST` | `/auth/logout` | Bearer | Client-side token removal, returns timestamp |

**Login response shape** (verified):
```json
{
  "access_token": "...",
  "refresh_token": "...",
  "token_type": "bearer",
  "expires_in": 3600,
  "user": { "id": "...", "email": "...", "firstName": "...", "lastName": "..." },
  "require_password_change": false
}
```

> **ugsys identity-manager mapping**: `/api/v1/auth/login` → same shape, add `require_password_change` field. `/api/v1/auth/me` → same. `/api/v1/auth/logout` → same.

---

### 13.3 Complete Admin API Endpoint Table (source: `admin_router.py`)

All endpoints under prefix `/v2/admin/`. All require `require_admin` dependency (JWT + `isAdmin: true`) except `GET /debug-stats` (no auth — debug only).

#### Dashboard & Analytics

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v2/admin/dashboard` | Basic dashboard: `totalUsers`, `activeUsers`, `totalProjects`, `activeProjects`, `totalSubscriptions`, `activeSubscriptions`, `lastUpdated` |
| `GET` | `/v2/admin/dashboard/enhanced` | Enhanced: adds `adminUsers`, `recentSignups`, `projectStats`, `systemHealth` |
| `GET` | `/v2/admin/analytics` | Full analytics: `users{}`, `projects{}`, `subscriptions{}`, `generatedAt` |
| `GET` | `/v2/admin/stats` | Comprehensive: dashboard + performance stats + `system.version` |
| `GET` | `/v2/admin/debug-stats` | **No auth** — debug only, returns dashboard data |

#### User Management

| Method | Path | Query Params | Description |
|--------|------|-------------|-------------|
| `GET` | `/v2/admin/users` | `search`, `limit` | List all users with roles attached |
| `GET` | `/v2/admin/users/paginated` | `page`, `pageSize`, `sortBy`, `sortDirection`, `search`, `isAdmin`, `isActive`, `emailVerified` | Paginated user list |
| `GET` | `/v2/admin/users/{user_id}` | — | Get single user |
| `POST` | `/v2/admin/users` | — | Create user (`PersonCreate` body) |
| `PUT` | `/v2/admin/users/{user_id}` | — | Update user (`PersonUpdate` body) |
| `DELETE` | `/v2/admin/users/{user_id}` | — | Delete user (blocked if active subscriptions exist) |
| `POST` | `/v2/admin/users/bulk-action` | — | Bulk action (see 13.4) |

#### People & Subscriptions (Admin Aliases)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v2/admin/people` | List all people (alias for users) |
| `PUT` | `/v2/admin/people/{person_id}` | Edit person (`PersonUpdate` body) |
| `GET` | `/v2/admin/subscriptions` | List all subscriptions |

#### Performance & Health

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v2/admin/performance/health` | Health status: `status`, `overallScore`, `timestamp` |
| `GET` | `/v2/admin/performance/stats` | Perf stats: `total_requests`, `average_response_time_ms`, `uptime_seconds` |
| `GET` | `/v2/admin/performance/cache/stats` | Cache: `hitRate`, `missRate`, `totalRequests`, `cacheSize`, `evictions` |
| `GET` | `/v2/admin/performance/dashboard` | Performance dashboard data |
| `GET` | `/v2/admin/performance/analytics` | Performance analytics |
| `GET` | `/v2/admin/performance/slowest-endpoints` | Slowest endpoints list |

#### Database & Debug

| Method | Path | Query Params | Description |
|--------|------|-------------|-------------|
| `GET` | `/v2/admin/database/performance/metrics` | — | DB metrics: `queryTime`, `connections`, `throughput` |
| `GET` | `/v2/admin/database/performance/optimization-history` | `range` (default `24h`) | Optimization history |
| `GET` | `/v2/admin/debug/session` | — | Current session info (debug) |
| `GET` | `/v2/admin/test` | — | Admin system smoke test |

> **ugsys projects-registry decision**: The performance/cache/database endpoints are Registry-specific operational tooling. Phase 2 does NOT need to replicate them. The ugsys admin surface should expose: dashboard, analytics, user CRUD, bulk-action, subscriptions list. Performance monitoring belongs in CloudWatch/observability stack.

---

### 13.4 Bulk User Action Contract (source: `admin_service.py`)

**Endpoint**: `POST /v2/admin/users/bulk-action`

**Request body**:
```json
{
  "action": "activate | deactivate | delete",
  "userIds": ["uuid1", "uuid2", "..."]
}
```

**Response**:
```json
{
  "action": "activate",
  "totalUsers": 3,
  "successCount": 2,
  "failureCount": 1,
  "results": [
    { "userId": "uuid1", "status": "success" },
    { "userId": "uuid2", "status": "success" },
    { "userId": "uuid3", "status": "failed", "error": "..." }
  ]
}
```

**Valid actions**: `activate`, `deactivate`, `delete`
**Validation**: Both `action` and `userIds` are required — returns 400 if either missing.

> **ugsys projects-registry**: Implement `POST /api/v1/admin/users/bulk-action` with identical contract. Map to `activate_person` / `deactivate_person` / `delete` on DynamoDB adapter.

**Basic dashboard** (`get_dashboard_data`):
```json
{
  "totalUsers": 42,
  "activeUsers": 38,
  "totalProjects": 5,
  "activeProjects": 3,
  "totalSubscriptions": 120,
  "activeSubscriptions": 95,
  "lastUpdated": "2025-08-01T13:00:00"
}
```

**Enhanced dashboard** (`get_enhanced_dashboard_data`) — adds:
```json
{
  "adminUsers": 2,
  "recentSignups": 7,
  "projectStats": {
    "<project_id>": { "name": "...", "subscriptions": 15, "status": "active" }
  },
  "systemHealth": { "status": "healthy", "uptime": "99.9%", "lastCheck": "..." }
}
```

**Analytics** (`get_analytics_data`):
```json
{
  "users": { "totalUsers": N, "activeUsers": N, "adminUsers": N, "inactiveUsers": N },
  "projects": { "totalProjects": N, "activeProjects": N, "projectsByStatus": { "active": N, "completed": N } },
  "subscriptions": { "totalSubscriptions": N, "activeSubscriptions": N, "subscriptionsByProject": { "<id>": N } },
  "generatedAt": "..."
}
```

---

### 13.6 AuditEventType Enum — Full Canonical List (source: `audit_logger.py`)

This is the authoritative event taxonomy. The ugsys identity-manager audit logging implementation MUST cover at minimum the Auth and Security categories.

| Category | Enum Name | String Value |
|----------|-----------|-------------|
| Auth | `LOGIN_SUCCESS` | `auth.login.success` |
| Auth | `LOGIN_FAILED` | `auth.login.failed` |
| Auth | `LOGOUT` | `auth.logout` |
| Auth | `PASSWORD_CHANGED` | `auth.password.changed` |
| Auth | `PASSWORD_RESET_REQUESTED` | `auth.password.reset.requested` |
| Auth | `PASSWORD_RESET_COMPLETED` | `auth.password.reset.completed` |
| Auth | `ACCOUNT_LOCKED` | `auth.account.locked` |
| Data | `DATA_READ` | `data.read` |
| Data | `DATA_CREATE` | `data.create` |
| Data | `DATA_UPDATE` | `data.update` |
| Data | `DATA_DELETE` | `data.delete` |
| Admin | `ADMIN_ACTION` | `admin.action` |
| Admin | `BULK_OPERATION` | `admin.bulk.operation` |
| Admin | `PERMISSION_GRANTED` | `admin.permission.granted` |
| Admin | `PERMISSION_REVOKED` | `admin.permission.revoked` |
| Security | `UNAUTHORIZED_ACCESS` | `security.unauthorized.access` |
| Security | `SUSPICIOUS_ACTIVITY` | `security.suspicious.activity` |
| Security | `RATE_LIMIT_EXCEEDED` | `security.rate.limit.exceeded` |
| Security | `INPUT_VALIDATION_FAILED` | `security.input.validation.failed` |
| System | `SYSTEM_ERROR` | `system.error` |
| System | `SYSTEM_STARTUP` | `system.startup` |
| System | `SYSTEM_SHUTDOWN` | `system.shutdown` |

**Audit record shape** (verified from `AuditLogger.log_event`):
```json
{
  "timestamp": "2025-08-01T13:00:00+00:00",
  "event_type": "auth.login.success",
  "user_id": "...",
  "resource_type": "User",
  "resource_id": "...",
  "action": "login",
  "result": "success | error | security_event",
  "ip_address": "...",
  "user_agent": "...",
  "details": {}
}
```

> **ugsys identity-manager**: Implement `AuditEventType` enum with these exact string values. Use structlog (not `logging.getLogger`) — emit as structured JSON. Store in DynamoDB `AuditLogsTable` (GSI on `user_id`, TTL 90 days).

---

### 13.7 Subscription Workflow — Decision Tree (source: `ENHANCED_SUBSCRIPTION_WORKFLOW.md`)

Public subscription flow (no auth required at entry point):

```
POST /v2/people/check-email  { "email": "..." }
         │
         ├─ exists: false ──► POST /v2/public/subscribe  → account created + subscription pending
         │
         └─ exists: true
                  │
                  └─► POST /v2/subscriptions/check  { "email": "...", "projectId": "..." }
                               │
                               ├─ subscribed: false ──► Show "please login to subscribe"
                               │
                               └─ subscribed: true  ──► Show "already subscribed — login to view status"
```

**All new subscriptions start as `pending`** — require admin approval before activation.

#### Check Email Endpoint

```
POST /v2/people/check-email
Body: { "email": "user@example.com" }
Response: { "exists": true, "version": "v2" }
```

#### Check Subscription Status Endpoint

```
POST /v2/subscriptions/check
Body: { "personId": "550e...", "projectId": "01JXXX" }
Response: { "exists": true }
```

> **Note**: The workflow uses `POST /v2/people/check-email` first to resolve email → personId, then passes personId here. The endpoint does NOT accept email directly.

> **ugsys projects-registry gap**: Add `POST /api/v1/public/check-email` and `POST /api/v1/subscriptions/check` (body: `personId` + `projectId`) to the Phase 2 endpoint list.

---

### 13.8 Form Submission Validation Rules (source: `form_submission_service.py`)

`FormSubmissionService.validate_submission_against_schema` enforces:

| Rule | Detail |
|------|--------|
| Required fields | If `field.required == true` and `field_id` not in `submission.responses` → `ValueError` |
| `poll_single` | `response_value` must be in `field.options` (string match) |
| `poll_multiple` | `response_value` must be a `list`; each item must be in `field.options` |
| Other field types | No additional validation beyond required check |

**ProjectSubmission model fields**: `id`, `projectId`, `personId`, `responses: dict[str, Any]`, `createdAt`, `updatedAt`

> **ugsys projects-registry**: Implement identical validation in `application/services/FormSubmissionService`. Field types to support at minimum: `text`, `poll_single`, `poll_multiple`.

---

### 13.9 Registry Auth Security Notes (for ugsys migration)

From `AUTHENTICATION_SYSTEM.md` — items that differ from ugsys standards and require explicit decisions:

| Item | Registry (current) | ugsys Standard | Action |
|------|--------------------|----------------|--------|
| JWT algorithm | HS256 | RS256 | Phase 1: identity-manager issues RS256 tokens; Registry keeps HS256 until migrated |
| Password hashing | bcrypt 12 rounds | bcrypt (via `ugsys-auth-client`) | Consistent — no change needed |
| Token storage | `localStorage` (frontend) | HttpOnly cookie or `localStorage` | Decision needed in Phase 4 (admin panel) |
| `require_password_change` flag | Present in login response | ✅ Added to `POST /api/v1/auth/login` response shape | No action needed |
| `lastLoginAt` field | Updated on login | ✅ Added to `User` entity in identity-manager | No action needed |
| Account lockout | Graceful fallback (not enforced) | Must enforce (security standard) | Phase 1: implement hard lockout after N failed attempts |

---

### 13.10 Gap Tracker Updates

> All gaps from this section have been merged into Section 10 (10.1 and 10.3). This subsection is retained for traceability only.

---

*Section 13 added: 2026-02-24 — verified from exhaustive source read of Registry admin_router.py, admin_service.py, audit_logger.py, form_submission_service.py, AUTHENTICATION_SYSTEM.md, V2_API_ENDPOINTS.md, ENHANCED_SUBSCRIPTION_WORKFLOW.md.*

---

## Section 14 — Admin Panel Plugin Architecture

*Added: 2026-02-24 — see ADR-0005 for full decision rationale.*

---

### 14.1 Overview

`ugsys-admin-panel` is a unified control plane for all platform services. It is a single React + Vite SPA organized as a monorepo with Feature-Sliced Design (FSD) internal boundaries. Each service's admin UI is a *slice* inside the monorepo — not a separately deployed remote module.

Services register themselves at startup (push model). The panel fetches the service registry from identity-manager and renders the appropriate nav and config screens dynamically. No hardcoded service knowledge in the panel code.

```
┌──────────────────────────────────────────────────────────┐
│                   ugsys-admin-panel                      │
│   React + Vite SPA  (FSD monorepo)                       │
│                                                          │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────┐  │
│  │  identity  │ │  projects  │ │ omnichannel│ │  ... │  │
│  │  /features │ │  /features │ │  /features │ │slice │  │
│  └────────────┘ └────────────┘ └────────────┘ └──────┘  │
│         ↑ FSD slices, same bundle, clean domain seams    │
└──────────────────────────────────────────────────────────┘
         ↑ service list fetched from identity-manager at runtime
         ↑ each service registers itself at startup (push)

All API calls → admin panel BFF proxy → target service
JWT lives in HttpOnly cookie — never in JS memory
```

> **Why not micro-frontends**: MFEs are worth the complexity when the bottleneck is organizational scale (many independent teams). We have one team. The costs — remote entry waterfall, version skew matrix, build complexity, and XSS risk from passing JWT to dynamically loaded remote code — exceed the benefit. FSD boundaries inside a monorepo deliver the same domain isolation. See ADR-0005 for full rationale.

---

### 14.2 Service Registration Contract

Every `ugsys-*` service MUST call both endpoints at startup (FastAPI lifespan):

#### 14.2.1 Register with identity-manager (schema + roles)

```
POST /api/v1/services/register
Host: identity-manager
Authorization: Bearer <service S2S token>
```

Request body:
```json
{
  "service_id": "ugsys-projects-registry",
  "display_name": "Projects Registry",
  "version": "1.2.0",
  "nav_icon": "folder",
  "health_url": "https://api.cbba.cloud.org.bo/projects/api/v1/health",
  "roles": [
    { "name": "projects:admin", "description": "Full projects management" },
    { "name": "projects:viewer", "description": "Read-only access to projects" }
  ],
  "config_schema": {
    "type": "object",
    "properties": {
      "max_subscriptions_per_project": {
        "type": "integer",
        "default": 100,
        "description": "Maximum subscriptions allowed per project"
      },
      "admin_notification_email": {
        "type": "string",
        "format": "email",
        "description": "Email address for admin notifications"
      },
      "subscription_approval_required": {
        "type": "boolean",
        "default": false,
        "description": "Require manual approval for subscriptions"
      }
    }
  }
}
```

Response `200`:
```json
{ "registered": true, "service_id": "ugsys-projects-registry" }
```

> Identity-manager stores the schema, roles, and service metadata. The admin panel fetches the full service registry from identity-manager to build its nav and config screens.

#### 14.2.2 Fetch operator config at startup

After registering, each service fetches its operator-set config values and applies them over env var defaults:

```
GET /api/v1/services/{service_id}/config
Host: identity-manager
Authorization: Bearer <service S2S token>
```

Response `200`:
```json
{
  "service_id": "ugsys-projects-registry",
  "config": {
    "admin_notification_email": "admin@cbba.cloud.org.bo",
    "subscription_approval_required": true
  }
}
```

> If no operator config has been set yet, returns `{ "config": {} }` — service uses env var defaults.

---

### 14.3 Identity-Manager Service Registry Endpoints (new — Phase 4)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/v1/services/register` | S2S token | Register service schema, roles, metadata |
| `GET` | `/api/v1/services` | `super_admin` JWT | List all registered services with schemas |
| `GET` | `/api/v1/services/{service_id}` | `super_admin` JWT | Get service schema + roles + metadata |
| `PUT` | `/api/v1/services/{service_id}/config` | `super_admin` JWT | Set operator config values for a service |
| `GET` | `/api/v1/services/{service_id}/config` | S2S token or `{service}:admin` JWT | Get current operator config values |
| `DELETE` | `/api/v1/services/{service_id}` | `super_admin` JWT | Deregister a service (admin only) |

> `PUT /config` stores operator-set values in DynamoDB. Services fetch their own config at startup via `GET /config` using their S2S token, overriding env var defaults with operator-set values.

---

### 14.4 Frontend Architecture — React SPA with FSD

**Stack**: React 18 + Vite + TypeScript + Feature-Sliced Design

The admin panel is a single deployable SPA. Each service's admin UI is a FSD *feature slice* inside the monorepo. Slices share a design system, router, and auth context — no network hops between them.

```
ugsys-admin-panel/src/
├── app/                        # App shell, router, providers
├── pages/                      # Route-level pages
│   ├── identity/               # Identity manager admin pages
│   ├── projects/               # Projects registry admin pages
│   ├── omnichannel/            # Omnichannel admin pages
│   ├── mass-messaging/         # Mass messaging admin pages
│   └── platform/               # Platform-wide config, service registry
├── features/                   # Cross-cutting features (auth, nav, notifications)
├── entities/                   # Shared domain models (User, Service, Config)
├── shared/
│   ├── ui/                     # Design system components
│   ├── api/                    # BFF proxy client (typed fetch wrappers)
│   └── lib/                    # Utilities, hooks
```

**FSD public API rule**: Each slice exports only from its `index.ts`. No deep imports across slices. This is the contract that makes future MFE extraction safe if the team scales.

**Adding a new service**: Add a `pages/{service}/` slice and a `features/{service}-nav/` entry. No changes to the shell or router config — the nav is built dynamically from the identity-manager service registry at runtime.

---

### 14.5 Auth — HttpOnly Cookie + BFF Proxy

**JWT is never accessible to JavaScript.** This is a hard security requirement.

#### Login flow

```
1. Admin submits credentials to admin panel BFF
   POST /bff/auth/login  { email, password }

2. BFF calls identity-manager
   POST /api/v1/auth/login

3. Identity-manager returns RS256 JWT

4. BFF sets HttpOnly, Secure, SameSite=Strict cookie
   Set-Cookie: session=<JWT>; HttpOnly; Secure; SameSite=Strict; Path=/

5. BFF returns { user, roles } to the SPA (no token in response body)

6. SPA stores only non-sensitive user metadata in React context
```

#### API call flow (all service admin calls)

```
SPA → POST /bff/proxy/projects/admin/users
      Cookie: session=<JWT>  (browser sends automatically)

BFF:
  1. Reads HttpOnly cookie
  2. Validates JWT signature (RS256, ugsys-auth-client)
  3. Checks caller has required role for this proxy route
  4. Forwards request to target service:
     POST https://api.cbba.cloud.org.bo/projects/api/v1/admin/users
     Authorization: Bearer <JWT>
     X-Request-ID: <correlation-id>

Target service:
  1. Validates JWT via ugsys-auth-client
  2. Checks required role from JWT claims
  3. Returns response

BFF → SPA: forwards response body + status
```

#### CSRF protection

All state-mutating BFF endpoints require a CSRF token:
- On login, BFF sets a second cookie: `csrf_token=<random>; SameSite=Strict; Secure` (NOT HttpOnly — JS must read it)
- SPA reads `csrf_token` cookie and sends it as `X-CSRF-Token` header on every POST/PUT/DELETE
- BFF validates that `X-CSRF-Token` header matches the `csrf_token` cookie value

> `SameSite=Strict` on the session cookie already prevents most CSRF. The explicit CSRF token is defense-in-depth.

#### BFF proxy route table

| BFF path prefix | Proxied to |
|-----------------|-----------|
| `/bff/proxy/identity/` | `ugsys-identity-manager` admin endpoints |
| `/bff/proxy/projects/` | `ugsys-projects-registry` admin endpoints |
| `/bff/proxy/omnichannel/` | `ugsys-omnichannel-service` admin endpoints |
| `/bff/proxy/messaging/` | `ugsys-mass-messaging` admin endpoints |
| `/bff/proxy/profiles/` | `ugsys-user-profile-service` admin endpoints |

---

### 14.6 RBAC for Admin Panel

All service-specific roles are defined by each service at registration time and stored in identity-manager. The admin panel manages role assignments through identity-manager's existing RBAC endpoints.

Platform-level roles:

| Role | Access |
|------|--------|
| `super_admin` | Full access to all services, platform config, service registry |
| `{service}:admin` | Full access to that service's admin slice only |
| `{service}:viewer` | Read-only access to that service's admin slice |

> `super_admin` is the only role that can manage other admins and update service configs. It is assigned only through identity-manager directly — never through the admin panel itself (prevents privilege escalation loop).

The BFF enforces role checks before proxying. A `projects:admin` user cannot reach `/bff/proxy/omnichannel/` — the BFF rejects the request before it ever hits the target service.

---

### 14.7 Service Startup Sequence (with registration)

```python
# src/main.py — lifespan pattern (every ugsys-* service)

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    logger.info("startup.begin", service=settings.service_name, version=settings.version)

    # 1. Register with identity-manager (schema + roles + metadata)
    #    Non-fatal: log warning and continue if identity-manager is unreachable
    try:
        await identity_manager_client.register_service(
            service_id=settings.service_id,
            display_name=settings.display_name,
            version=settings.version,
            nav_icon=settings.nav_icon,
            health_url=f"{settings.public_base_url}/api/v1/health",
            config_schema=SERVICE_CONFIG_SCHEMA,
            roles=SERVICE_ROLES,
        )
        logger.info("service.registered", service=settings.service_id)
    except Exception as e:
        logger.warning("service.registration_failed", service=settings.service_id, error=str(e))

    # 2. Fetch operator config from identity-manager (overrides env defaults)
    #    Non-fatal: if unreachable, service runs with env var defaults
    try:
        config = await identity_manager_client.get_service_config(settings.service_id)
        settings.apply_remote_config(config)
        logger.info("service.config_loaded", service=settings.service_id)
    except Exception as e:
        logger.warning("service.config_load_failed", service=settings.service_id, error=str(e))

    logger.info("startup.complete", service=settings.service_name)
    yield
    logger.info("shutdown.complete", service=settings.service_name)
```

> Both calls are non-fatal. A service that cannot reach identity-manager at startup still serves its primary API with env var defaults. This prevents a cascading failure where identity-manager downtime takes down all services.

---

### 14.8 Admin Panel BFF — Architecture

The BFF (Backend for Frontend) is a lightweight FastAPI Lambda that:
- Handles login/logout and manages the HttpOnly session cookie
- Proxies all admin API calls to target services (adding `Authorization` header)
- Enforces role-based access per proxy route before forwarding
- Adds correlation IDs to all proxied requests
- Logs all admin actions (audit trail)

```
ugsys-admin-panel/
├── bff/                        # FastAPI BFF Lambda
│   ├── src/
│   │   ├── presentation/api/v1/
│   │   │   ├── auth.py         # /bff/auth/login, /bff/auth/logout, /bff/auth/me
│   │   │   └── proxy.py        # /bff/proxy/{service}/{path:path}
│   │   ├── application/
│   │   │   └── services/
│   │   │       ├── auth_service.py     # cookie management, CSRF
│   │   │       └── proxy_service.py    # JWT validation, role check, forward
│   │   ├── infrastructure/
│   │   │   └── adapters/
│   │   │       └── identity_manager_client.py
│   │   └── config.py
│   └── main.py
└── frontend/                   # React SPA (Vite)
    └── src/  (FSD structure above)
```

---

### 14.9 Gap Tracker — Admin Panel (Phase 4)

| Gap | Priority | Phase |
|-----|----------|-------|
| Scaffold `ugsys-admin-panel` repo (BFF + React SPA) | P0 | 4 |
| BFF: `POST /bff/auth/login` — sets HttpOnly cookie | P0 | 4 |
| BFF: `POST /bff/auth/logout` — clears cookie | P0 | 4 |
| BFF: `GET /bff/proxy/{service}/{path}` — proxy with JWT forwarding | P0 | 4 |
| BFF: role-based proxy route guard | P0 | 4 |
| BFF: CSRF token cookie + header validation | P0 | 4 |
| `POST /api/v1/services/register` in identity-manager | P0 | 4 |
| `GET/PUT /api/v1/services/{id}/config` in identity-manager | P0 | 4 |
| Service registry DynamoDB table in identity-manager | P0 | 4 |
| React SPA: FSD structure + nav built from service registry | P0 | 4 |
| React SPA: identity-manager admin slice (users, RBAC) | P0 | 4 |
| React SPA: projects-registry admin slice | P1 | 4 |
| React SPA: omnichannel admin slice | P1 | 4 |
| React SPA: mass-messaging admin slice | P1 | 4 |
| React SPA: platform config slice (service registry viewer) | P1 | 4 |
| `super_admin` bootstrap — first admin creation flow | P0 | 4 |
| BFF: audit log for all admin actions (structlog → CloudWatch) | P1 | 4 |
| `settings.apply_remote_config()` in every service | P1 | 4 |
| Contract tests: BFF proxy routes vs service admin endpoints | P1 | 4 |

---

*Section 14 added: 2026-02-24 — see ADR-0005 for full decision rationale.*
