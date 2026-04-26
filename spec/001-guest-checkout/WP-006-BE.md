# WP-006-BE: Admin Authentication — Backend Work Package

**Feature:** 001-guest-checkout
**Target workspace:** `workspaces/order-service`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Implement admin authentication in order-service: login with email/password, MFA (TOTP) for privileged roles, JWT issuance with access and refresh tokens, and auth guards on admin endpoints. Merchant admin users authenticate separately from storefront guests. This WP delivers the `admin_users` table, password hashing (bcrypt), TOTP verification (PyOTP), JWT encode/decode (PyJWT), the three auth API endpoints (`POST /auth/login`, `POST /auth/mfa/verify`, `POST /auth/token/refresh`), and a reusable FastAPI auth guard dependency that protects all admin endpoints.

All external service calls (none for this WP — auth is self-contained in order-service) are internal. The auth guard dependency created here is consumed by WP-007-BE and WP-008-BE.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-021 — Admin login

Merchant admin users (`merchant_owner`, `merchant_admin`, `merchant_viewer`, `merchant_support`) sign in with email and password. For `merchant_viewer` and `merchant_support`, the returned JWT grants full read access. For `merchant_owner` and `merchant_admin`, the JWT is marked as requiring MFA completion.

**Testable:** `POST /auth/login` *(to be created)* with valid email and password for a merchant role returns `200` with a JWT. For `merchant_viewer`/`merchant_support`, the JWT allows access to `GET /orders`. For `merchant_owner`/`merchant_admin`, the JWT is flagged as pre-MFA.

### AC-022 — MFA verification for privileged roles

`merchant_owner` and `merchant_admin` users must complete TOTP verification after password authentication to gain full access.

**Testable:** `POST /auth/mfa/verify` *(to be created)* with a valid TOTP code and a pre-MFA JWT returns `200` with a fully authenticated JWT that grants access to admin endpoints.

### AC-023 — Invalid credentials rejected

Login with an incorrect email or password is rejected with a generic error message (no information leakage about which field is wrong).

**Testable:** `POST /auth/login` with an incorrect password returns `401` with an error message. UI displays "Invalid email or password" without indicating which field was incorrect.

### AC-024 — MFA enforcement

Users with `merchant_owner` or `merchant_admin` roles cannot access admin-protected endpoints without completing MFA.

**Testable:** `GET /orders` with a pre-MFA JWT for a `merchant_owner` or `merchant_admin` user returns `403`. UI redirects to the MFA verification step.

---

## Test Scenarios (verbatim from TS-001)

### TS-001-039 — Viewer/support login returns fully authenticated JWT
- **Preconditions:** order-service running; user `viewer@example.com` exists with role `merchant_viewer`, password `ValidPass123!`
- **Action:** `POST /auth/login` with body `{ "email": "viewer@example.com", "password": "ValidPass123!" }`
- **Expected:** HTTP 200; response contains JWT with `role: "merchant_viewer"` and `tenant_id`; JWT is NOT flagged as pre-MFA; using this JWT on `GET /orders` returns 200

### TS-001-040 — Owner/admin login returns pre-MFA JWT
- **Preconditions:** order-service running; user `owner@example.com` exists with role `merchant_owner`, password `ValidPass123!`, TOTP secret configured
- **Action:** `POST /auth/login` with body `{ "email": "owner@example.com", "password": "ValidPass123!" }`
- **Expected:** HTTP 200; response contains JWT with `role: "merchant_owner"` flagged as pre-MFA (`mfa_verified: false` or `mfa_required: true`); using this JWT on `GET /orders` returns 403

### TS-001-041 — Valid TOTP code upgrades JWT to fully authenticated
- **Preconditions:** order-service running; `merchant_owner` user has pre-MFA JWT (from TS-001-040); TOTP code generated from user's configured secret
- **Action:** `POST /auth/mfa/verify` with body `{ "code": "{validTOTPCode}" }` and header `Authorization: Bearer {preMfaJwt}`
- **Expected:** HTTP 200; response contains new JWT that is fully authenticated (`mfa_verified: true`); using new JWT on `GET /orders` returns 200

### TS-001-042 — Invalid TOTP code is rejected
- **Preconditions:** order-service running; `merchant_owner` user has pre-MFA JWT
- **Action:** `POST /auth/mfa/verify` with body `{ "code": "000000" }` and header `Authorization: Bearer {preMfaJwt}`
- **Expected:** HTTP 401 or 403; response indicates invalid verification code; pre-MFA JWT is NOT upgraded — `GET /orders` with same JWT still returns 403

### TS-001-043 — Wrong password returns generic 401
- **Preconditions:** order-service running; user `viewer@example.com` exists with known password
- **Action:** `POST /auth/login` with body `{ "email": "viewer@example.com", "password": "WrongPassword!" }`
- **Expected:** HTTP 401; response message is generic ("Invalid email or password") — does not indicate whether email or password was incorrect; no JWT issued

### TS-001-044 — Non-existent email returns same generic 401
- **Preconditions:** order-service running; no user exists with email `nobody@example.com`
- **Action:** `POST /auth/login` with body `{ "email": "nobody@example.com", "password": "AnyPassword!" }`
- **Expected:** HTTP 401; same generic message as TS-001-043; response time comparable to real auth check (no timing-based user enumeration)

### TS-001-045 — Pre-MFA JWT cannot access admin endpoints
- **Preconditions:** order-service running; `merchant_owner` user logged in with pre-MFA JWT (not yet TOTP-verified)
- **Action:** `GET /orders` with headers `Authorization: Bearer {preMfaJwt}` and `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 403; response indicates MFA verification is required; no order data returned

---

## Relevant Contracts

### order-service.openapi.yaml (excerpts)

#### Security Schemes

```yaml
securitySchemes:
  bearerAuth:
    type: http
    scheme: bearer
    bearerFormat: JWT
```

#### POST /auth/login

```yaml
/auth/login:
  post:
    operationId: adminLogin
    summary: Authenticate an admin user
    tags: [Auth]
    description: >
      Authenticates a merchant admin user with email and password.
      Returns a JWT. For `merchant_viewer` and `merchant_support` roles,
      the JWT grants full read access. For `merchant_owner` and
      `merchant_admin` roles, the JWT is flagged as pre-MFA and requires
      TOTP verification before accessing admin endpoints.
    parameters:
      - name: X-Tenant-ID
        in: header
        required: true
        schema:
          type: string
          format: uuid
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/LoginRequest"
    responses:
      "200":
        description: Authentication successful.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/LoginResponse"
      "401":
        description: Invalid email or password.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ErrorResponse"
```

#### POST /auth/mfa/verify

```yaml
/auth/mfa/verify:
  post:
    operationId: verifyMfa
    summary: Verify TOTP code for MFA
    tags: [Auth]
    security:
      - bearerAuth: []
    parameters:
      - name: X-Tenant-ID
        in: header
        required: true
        schema:
          type: string
          format: uuid
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/MfaVerifyRequest"
    responses:
      "200":
        description: MFA verification successful.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/MfaVerifyResponse"
      "401":
        description: Invalid or expired TOTP code.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ErrorResponse"
      "403":
        description: JWT is not a pre-MFA token or MFA not required for this role.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ErrorResponse"
```

#### POST /auth/token/refresh

```yaml
/auth/token/refresh:
  post:
    operationId: refreshToken
    summary: Refresh an expired JWT
    tags: [Auth]
    security:
      - bearerAuth: []
    parameters:
      - name: X-Tenant-ID
        in: header
        required: true
        schema:
          type: string
          format: uuid
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/TokenRefreshRequest"
    responses:
      "200":
        description: Token refreshed.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/TokenRefreshResponse"
      "401":
        description: Refresh token is invalid or expired.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ErrorResponse"
```

#### Auth Schemas

```yaml
LoginRequest:
  type: object
  required: [email, password]
  properties:
    email:
      type: string
      format: email
    password:
      type: string
      minLength: 8

LoginResponse:
  type: object
  required: [access_token, token_type, expires_in]
  properties:
    access_token:
      type: string
      description: JWT access token.
    refresh_token:
      type: string
      description: Refresh token for obtaining new access tokens.
    token_type:
      type: string
      example: "Bearer"
    expires_in:
      type: integer
      description: Token lifetime in seconds.
    mfa_required:
      type: boolean
      description: >
        True for `merchant_owner` and `merchant_admin` roles.
        When true, the access_token is a pre-MFA token that cannot
        access admin endpoints until MFA verification is completed.

MfaVerifyRequest:
  type: object
  required: [code]
  properties:
    code:
      type: string
      minLength: 6
      maxLength: 6
      description: 6-digit TOTP code from the user's authenticator app.

MfaVerifyResponse:
  type: object
  required: [access_token, token_type, expires_in]
  properties:
    access_token:
      type: string
      description: Fully authenticated JWT (MFA verified).
    refresh_token:
      type: string
    token_type:
      type: string
      example: "Bearer"
    expires_in:
      type: integer

TokenRefreshRequest:
  type: object
  required: [refresh_token]
  properties:
    refresh_token:
      type: string

TokenRefreshResponse:
  type: object
  required: [access_token, token_type, expires_in]
  properties:
    access_token:
      type: string
    refresh_token:
      type: string
    token_type:
      type: string
      example: "Bearer"
    expires_in:
      type: integer

ErrorResponse:
  type: object
  required: [code, message]
  properties:
    code:
      type: string
    message:
      type: string
    details:
      type: array
      items:
        type: object
        properties:
          field:
            type: string
          issue:
            type: string
```

### roles.md — Merchant Roles and Authentication (excerpts)

#### Merchant / Admin Roles

| Role | Description | Scope |
|---|---|---|
| `merchant_owner` | The primary account holder for a tenant. Full access to all merchant features. Can invite and manage other admin users. | Tenant |
| `merchant_admin` | An invited admin user. Full access to order management, inventory, and reporting. Cannot manage billing or other admin users. | Tenant |
| `merchant_viewer` | Read-only access to the admin portal. Can view orders and reports but cannot modify data. | Tenant |
| `merchant_support` | Access to order detail and refund/cancel actions only. Intended for customer support agents. Cannot access inventory or reporting. | Tenant |

#### Permission Matrix (relevant actions)

| Action | `merchant_owner` | `merchant_admin` | `merchant_viewer` | `merchant_support` |
|---|---|---|---|---|
| View all tenant orders | ✅ | ✅ | ✅ | ✅ |
| Cancel / refund order | ✅ | ✅ | ❌ | ✅ |
| Manage inventory | ✅ | ✅ | ✅ (read) | ❌ |
| Manage admin users | ✅ | ❌ | ❌ | ❌ |
| View reports | ✅ | ✅ | ✅ | ❌ |
| Configure tenant settings | ✅ | ❌ | ❌ | ❌ |

#### Authentication

- **Merchant Admins:** Separate admin auth. Email + password with MFA required for `merchant_owner` and `merchant_admin`.
- **Platform Roles:** Internal SSO only. Not exposed through any public-facing endpoint.

---

## Implementation Notes

### Project structure

```
workspaces/order-service/
  src/
    config.py                   # Settings: JWT_SECRET, JWT_ACCESS_TTL, JWT_REFRESH_TTL, BCRYPT_ROUNDS
    main.py                     # FastAPI app setup
    dependencies.py             # Dependency injection (auth guard lives here)
    models/
      admin_user.py             # AdminUser ORM model
    schemas/
      auth.py                   # LoginRequest, LoginResponse, MfaVerifyRequest, etc.
    repositories/
      admin_user_repo.py        # Admin user data access
    services/
      password_service.py       # bcrypt hash/verify
      jwt_service.py            # JWT encode/decode, access + refresh tokens
      totp_service.py           # PyOTP verify
      auth_service.py           # Login + MFA orchestration
    api/
      health.py                 # GET /health, /ready, /metrics
      v1/
        auth.py                 # POST /auth/login, /auth/mfa/verify, /auth/token/refresh
  tests/
    conftest.py                 # Shared fixtures (test DB, test admin users)
    integration/
      test_auth.py              # All TS-001-039 through TS-001-045 scenarios
  alembic/
    versions/
      xxx_create_admin_users.py # Migration for admin_users table
  Dockerfile
  docker-compose.yml
  pyproject.toml
```

### Tech stack

- **Language:** Python 3.12+
- **Framework:** FastAPI
- **ORM:** SQLAlchemy 2.x (async)
- **Migrations:** Alembic
- **Tests:** pytest + pytest-asyncio + httpx (AsyncClient)
- **Password hashing:** bcrypt (via `bcrypt` package)
- **TOTP:** PyOTP
- **JWT:** PyJWT

### admin_users table

```sql
CREATE TABLE admin_users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    email           VARCHAR(254) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(30) NOT NULL CHECK (role IN ('merchant_owner', 'merchant_admin', 'merchant_viewer', 'merchant_support')),
    totp_secret     VARCHAR(100),       -- NULL for viewer/support roles (MFA not required)
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

### Key patterns

#### Password service (bcrypt)

- Hash passwords with `bcrypt` using 12 salt rounds.
- Verify using `bcrypt.checkpw()`.
- **Anti-enumeration:** When a login request has a non-existent email, still run a dummy `bcrypt.checkpw()` against a pre-hashed value. This ensures constant response time regardless of whether the email exists (prevents timing-based user enumeration, per AC-023).

#### JWT structure

```json
{
  "sub": "<user_id (UUID)>",
  "tenant_id": "<tenant_id (UUID)>",
  "role": "merchant_owner",
  "mfa_verified": false,
  "exp": 1713200000,
  "iat": 1713198200,
  "type": "access"
}
```

- **Access token TTL:** 30 minutes (1800 seconds)
- **Refresh token TTL:** 7 days (604800 seconds)
- Refresh tokens use `"type": "refresh"` in the payload.
- Signing algorithm: `HS256` with a server-side secret (`JWT_SECRET` env var).

#### Login flow

1. Look up user by `email` + `tenant_id` in `admin_users` table.
2. Verify password with `bcrypt.checkpw()`. If user not found, run dummy hash comparison (constant time).
3. If invalid: return 401 with `{"code": "INVALID_CREDENTIALS", "message": "Invalid email or password"}`.
4. If role is `merchant_viewer` or `merchant_support`: issue JWT with `mfa_verified: true`.
5. If role is `merchant_owner` or `merchant_admin`: issue JWT with `mfa_verified: false`.
6. Return `LoginResponse` with `mfa_required: true` or `false` accordingly.
7. Always issue both `access_token` and `refresh_token`.

#### MFA verify flow

1. Extract pre-MFA JWT from `Authorization: Bearer {token}` header.
2. Decode and verify JWT. Confirm `mfa_verified` is `false` — if already `true`, return 403 (MFA not required).
3. Look up user by `sub` claim. Verify `totp_secret` is not null.
4. Validate TOTP code using `pyotp.TOTP(totp_secret).verify(code, valid_window=1)` — allows 1 step tolerance (30-second window before/after).
5. If invalid: return 401 with `{"code": "INVALID_MFA_CODE", "message": "Invalid verification code"}`.
6. Issue new JWT with `mfa_verified: true`. Issue new refresh token.
7. Return `MfaVerifyResponse`.

#### Token refresh flow

1. Accept `refresh_token` in request body.
2. Decode the refresh token. Verify `type` is `"refresh"` and token is not expired.
3. Look up user by `sub` claim. Verify user is still active.
4. Issue a new access token (preserving `mfa_verified` state from the refresh token) and a new refresh token.
5. Return `TokenRefreshResponse`.

#### Auth guard (FastAPI dependency)

```python
async def require_admin_auth(
    authorization: str = Header(...),
    x_tenant_id: str = Header(...),
) -> AdminContext:
    """
    Extracts and validates JWT from Authorization header.
    Returns AdminContext(user_id, tenant_id, role, mfa_verified).
    
    Raises:
    - 401 if token is missing, invalid, or expired
    - 403 if mfa_verified is false for merchant_owner/merchant_admin
    """
```

- Decode JWT from `Authorization: Bearer {token}`.
- Verify expiry and signature.
- If `role` is `merchant_owner` or `merchant_admin` and `mfa_verified` is `false`: return 403 `{"code": "MFA_REQUIRED", "message": "MFA verification is required"}`.
- If `tenant_id` from JWT does not match `X-Tenant-ID` header: return 403.
- Return `AdminContext` dataclass with `user_id`, `tenant_id`, `role`, `mfa_verified`.

#### Seed test users

Create seed data (via Alembic migration or test fixture) with one user per role:

| Email | Role | Password | TOTP Secret | Tenant ID |
|---|---|---|---|---|
| `owner@example.com` | `merchant_owner` | `ValidPass123!` | `JBSWY3DPEHPK3PXP` (base32) | `{test_tenant_id}` |
| `admin@example.com` | `merchant_admin` | `ValidPass123!` | `JBSWY3DPEHPK3PXP` | `{test_tenant_id}` |
| `viewer@example.com` | `merchant_viewer` | `ValidPass123!` | NULL | `{test_tenant_id}` |
| `support@example.com` | `merchant_support` | `ValidPass123!` | NULL | `{test_tenant_id}` |

- Passwords are stored as bcrypt hashes (not plaintext).
- TOTP secret `JBSWY3DPEHPK3PXP` is a well-known test secret — use `pyotp.TOTP("JBSWY3DPEHPK3PXP").now()` in tests to generate valid codes.

### Key constraints

- **Generic error messages:** AC-023 requires the same error message for wrong email and wrong password. Never disclose which field was incorrect.
- **Constant-time comparison:** Always hash-compare even for non-existent users to prevent timing side-channels.
- **No PII in logs:** Do not log email addresses, passwords, or TOTP codes. Log only user IDs and roles.
- **MFA enforcement is server-side:** The auth guard checks `mfa_verified` on every admin request. The FE cannot bypass this.

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios (TS-001-039 through TS-001-045) have passing automated tests
- [ ] `ruff check src/` passes with zero errors
- [ ] `mypy src/` passes with zero errors
- [ ] `/health`, `/ready`, `/metrics` endpoints return expected responses
- [ ] No secrets, tokens, or credentials in code or test fixtures (TOTP test secrets are acceptable in test fixtures only — not in production config)
- [ ] Contract validation passes (Schemathesis against `POST /auth/login`, `POST /auth/mfa/verify`, `POST /auth/token/refresh`)

---

## Implementation Order

Implement in this order:

1. **DB model + migration:** `admin_users` table with Alembic migration (see schema above)
2. **Seed test users:** One per role with known passwords (bcrypt-hashed) and TOTP secrets; use test fixture or seed migration
3. **Password service:** `password_service.py` — bcrypt hash/verify with 12 salt rounds, dummy hash for non-existent users
4. **JWT service:** `jwt_service.py` — encode/decode with PyJWT, access + refresh tokens, `HS256` signing
5. **TOTP service:** `totp_service.py` — PyOTP verify with `valid_window=1`
6. **Auth guard dependency:** `dependencies.py` — `require_admin_auth` FastAPI dependency extracting JWT, enforcing MFA
7. **API: POST /auth/login:** `api/v1/auth.py` — login endpoint with password verification, JWT issuance, MFA flag
8. **API: POST /auth/mfa/verify:** MFA verification endpoint with TOTP validation, JWT upgrade
9. **API: POST /auth/token/refresh:** Token refresh endpoint with refresh token validation, new token issuance
10. **Tests:** All TS scenarios (TS-001-039 through TS-001-045) as integration tests using httpx AsyncClient
11. **Run DoD checklist:** ruff, mypy, contract validation, security review

---

## Dependencies

- **Wave:** 2 (can start parallel with WP-002-BE)
- **Depends on:** Nothing — this WP is self-contained within order-service
- **Depended on by:** WP-007-BE and WP-008-BE (both consume the auth guard), WP-007-FE (calls auth endpoints)
