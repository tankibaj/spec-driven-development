# WP-007-FE: Admin Auth UI — Frontend Work Package

**Feature:** 001-guest-checkout
**Target workspace:** `workspaces/admin-app`
**Status:** awaiting_review
**Generated:** 2026-04-11
**Depends on:** None (Wave 5 — can start with MSW mocks; switch to real order-service auth endpoints after WP-006-BE deploys)

---

## Objective

Implement the admin authentication UI in admin-app: login form (email + password), MFA verification step (6-digit TOTP code), error handling for invalid credentials, and auth guards that redirect unauthenticated users. This WP also scaffolds the admin-app workspace (package.json, Vite, React Router, TanStack Query, MSW, shell layout with sidebar nav and header).

This WP is self-contained. An implementer should be able to complete it reading only this file.

---

## Tech Stack

| Concern | Tool |
|---|---|
| Language | TypeScript (strict mode) |
| UI framework | React 18+ |
| Build tool | Vite |
| Routing | React Router v6+ |
| Data fetching | TanStack Query (React Query) v5+ |
| Auth state | Zustand |
| Testing | Vitest + React Testing Library |
| API mocking | MSW (Mock Service Worker) v2+ |
| Linting/Formatting | Biome |

---

## Acceptance Criteria (verbatim from FS-001)

### AC-021 — Admin login

Merchant admin users (`merchant_owner`, `merchant_admin`, `merchant_viewer`, `merchant_support`) sign in with email and password. For `merchant_viewer` and `merchant_support`, the returned JWT grants full read access. For `merchant_owner` and `merchant_admin`, the JWT is marked as requiring MFA completion.

**Testable:** `POST /auth/login` *(to be created)* with valid email and password for a merchant role returns `200` with a JWT. For `merchant_viewer`/`merchant_support`, the JWT allows access to `GET /orders`. For `merchant_owner`/`merchant_admin`, the JWT is flagged as pre-MFA.

---

### AC-022 — MFA verification for privileged roles

`merchant_owner` and `merchant_admin` users must complete TOTP verification after password authentication to gain full access.

**Testable:** `POST /auth/mfa/verify` *(to be created)* with a valid TOTP code and a pre-MFA JWT returns `200` with a fully authenticated JWT that grants access to admin endpoints.

---

### AC-023 — Invalid credentials rejected

Login with an incorrect email or password is rejected with a generic error message (no information leakage about which field is wrong).

**Testable:** `POST /auth/login` with an incorrect password returns `401` with an error message. UI displays "Invalid email or password" without indicating which field was incorrect.

---

### AC-024 — MFA enforcement

Users with `merchant_owner` or `merchant_admin` roles cannot access admin-protected endpoints without completing MFA.

**Testable:** `GET /orders` with a pre-MFA JWT for a `merchant_owner` or `merchant_admin` user returns `403`. UI redirects to the MFA verification step.

---

## Test Scenarios (from TS-001)

### TS-001-039 — Viewer/support login returns fully authenticated JWT
- **Preconditions:** MSW mock: `POST /auth/login` for `viewer@example.com` / `ValidPass123!` returns `200` with JWT, `mfa_required: false`.
- **Action:** User enters email and password on login page, clicks "Sign in".
- **Expected:** UI navigates to `/orders` (dashboard). Auth store contains `accessToken` and `user.role = "merchant_viewer"`. No MFA redirect.

### TS-001-040 — Owner/admin login returns pre-MFA JWT
- **Preconditions:** MSW mock: `POST /auth/login` for `owner@example.com` / `ValidPass123!` returns `200` with JWT, `mfa_required: true`.
- **Action:** User enters email and password on login page, clicks "Sign in".
- **Expected:** UI navigates to `/mfa` (MFA page). Auth store contains `accessToken` with `mfaVerified: false`. User is NOT redirected to `/orders`.

### TS-001-041 — Valid TOTP code upgrades JWT to fully authenticated
- **Preconditions:** User on MFA page with pre-MFA token in auth store. MSW mock: `POST /auth/mfa/verify` with `code: "123456"` returns `200` with new fully authenticated JWT.
- **Action:** User enters 6-digit TOTP code, clicks "Verify".
- **Expected:** UI navigates to `/orders`. Auth store updated with new `accessToken`, `mfaVerified: true`.

### TS-001-042 — Invalid TOTP code is rejected
- **Preconditions:** User on MFA page with pre-MFA token. MSW mock: `POST /auth/mfa/verify` with `code: "000000"` returns `401`.
- **Action:** User enters invalid 6-digit TOTP code, clicks "Verify".
- **Expected:** MFA page displays error "Invalid verification code". User remains on `/mfa`. Auth store unchanged (`mfaVerified: false`).

### TS-001-043 — Wrong password returns generic 401
- **Preconditions:** MSW mock: `POST /auth/login` for `viewer@example.com` / `WrongPassword!` returns `401` with generic error.
- **Action:** User enters email and wrong password on login page, clicks "Sign in".
- **Expected:** Login page displays "Invalid email or password". No JWT stored. User remains on `/login`.

### TS-001-044 — Non-existent email returns same generic 401
- **Preconditions:** MSW mock: `POST /auth/login` for `nobody@example.com` / `AnyPassword!` returns `401` with generic error.
- **Action:** User enters non-existent email and password on login page, clicks "Sign in".
- **Expected:** Login page displays "Invalid email or password" (same message as TS-001-043). No JWT stored. User remains on `/login`.

### TS-001-045 — Pre-MFA JWT cannot access admin endpoints (FE redirect)
- **Preconditions:** Auth store contains pre-MFA token (`mfaVerified: false`). User attempts to navigate to `/orders`.
- **Action:** User enters `/orders` in browser URL bar (or clicks a link).
- **Expected:** Auth guard detects pre-MFA state, redirects to `/mfa`. Order list page is NOT rendered.

---

## UI Flow

```
[Login Page /login]
    ↓ POST /auth/login
    ├── 200, mfa_required=false → [Order List /orders]
    ├── 200, mfa_required=true  → [MFA Page /mfa]
    └── 401                     → "Invalid email or password" (stay on /login)

[MFA Page /mfa]
    ↓ POST /auth/mfa/verify
    ├── 200 → [Order List /orders]
    └── 401 → "Invalid verification code" (stay on /mfa)

[Auth Guard]
    ├── No token          → redirect to /login
    ├── Token, !mfaVerified, mfa_required → redirect to /mfa
    └── Fully authenticated → allow access
```

### Step 1 — Login Page (`/login`)

The login page displays a centered form with the application title/logo, email field, password field, and a "Sign in" button.

| Field | Label | Required | Validation |
|---|---|---|---|
| `email` | Email address | Yes | Valid email format |
| `password` | Password | Yes | Minimum 8 characters |

**CTA:** "Sign in"

- On success with `mfa_required === false`: store tokens, navigate to `/orders`.
- On success with `mfa_required === true`: store pre-MFA tokens, navigate to `/mfa`.
- On `401`: display "Invalid email or password" below the form. Re-enable the submit button.
- If user is already fully authenticated: redirect to `/orders` (skip login).

### Step 2 — MFA Page (`/mfa`)

The MFA page displays a centered form with instructions ("Enter the 6-digit code from your authenticator app"), a single code input field, and a "Verify" button.

| Field | Label | Required | Validation |
|---|---|---|---|
| `code` | Verification code | Yes | Exactly 6 digits |

**CTA:** "Verify"

- On success: store new tokens, set `mfaVerified: true`, navigate to `/orders`.
- On `401`: display "Invalid verification code" below the form. Re-enable the submit button. Clear the code input.
- If user has no pre-MFA token: redirect to `/login`.
- If user is already fully authenticated: redirect to `/orders`.

### Step 3 — Shell Layout (post-authentication)

After successful authentication, all protected pages render inside a shell layout:

- **Sidebar:** Navigation links — "Orders" (links to `/orders`).
- **Header:** Displays tenant name (from JWT or hardcoded for MVP) and a "Logout" button.
- **Main area:** `<Outlet />` for child routes.

"Logout" clears the auth store and `sessionStorage`, then redirects to `/login`.

---

## Error Handling

### Invalid login credentials

**When:** `POST /auth/login` returns `401`.
**Display:** "Invalid email or password"
**Behavior:** Form stays on the login page. Submit button re-enabled. Password field cleared.

### Invalid TOTP code

**When:** `POST /auth/mfa/verify` returns `401`.
**Display:** "Invalid verification code"
**Behavior:** Form stays on the MFA page. Submit button re-enabled. Code input cleared.

### Network error (login or MFA)

**When:** Fetch to `/auth/login` or `/auth/mfa/verify` fails (network error, timeout).
**Display:** "Unable to connect. Please try again."
**Behavior:** Form stays on current page. Submit button re-enabled.

### Unauthenticated access attempt

**When:** User navigates to a protected route without any token.
**Display:** No error message — silent redirect.
**Behavior:** Redirect to `/login`. After login, redirect back to the originally requested route (store `returnTo` in the auth store or URL param).

### Pre-MFA access attempt

**When:** User with `mfaVerified === false` navigates to a protected route (AC-024 FE behavior).
**Display:** No error message — silent redirect.
**Behavior:** Redirect to `/mfa`. After MFA verification, redirect to `/orders`.

---

## State Management

### Auth Store (Zustand)

```typescript
interface AuthState {
  accessToken: string | null;
  refreshToken: string | null;
  user: {
    role: string;      // e.g. "merchant_owner", "merchant_viewer"
    tenantId: string;
    mfaVerified: boolean;
  } | null;

  // Actions
  login: (email: string, password: string) => Promise<LoginResult>;
  verifyMfa: (code: string) => Promise<MfaResult>;
  logout: () => void;
}
```

**Persistence:** Tokens persisted to `sessionStorage` (not `localStorage` — admin sessions should not survive browser close). On app load, rehydrate from `sessionStorage`. If no token found, user starts unauthenticated.

**JWT decoding:** After receiving `access_token` from login or MFA verify, decode the JWT payload (without verification — verification is BE responsibility) to extract `role`, `tenant_id`, and `mfa_verified` (or equivalent claims). Store these in the `user` object.

**Login flow:**
1. `login(email, password)` → `POST /auth/login` with `{ email, password }`.
2. On `200`: store `access_token` and `refresh_token` in state + `sessionStorage`. Decode JWT to populate `user`. Return `{ success: true, mfaRequired: response.mfa_required }`.
3. On `401`: return `{ success: false, error: "Invalid email or password" }`.

**MFA flow:**
1. `verifyMfa(code)` → `POST /auth/mfa/verify` with `{ code }` and `Authorization: Bearer {preMfaToken}`.
2. On `200`: replace `access_token` and `refresh_token`. Re-decode JWT. Set `user.mfaVerified = true`. Return `{ success: true }`.
3. On `401`: return `{ success: false, error: "Invalid verification code" }`.

**Logout:**
1. Clear all auth state (`accessToken`, `refreshToken`, `user` → null).
2. Clear `sessionStorage`.
3. Redirect to `/login`.

---

## API Client

Create an API client module (e.g., `src/lib/api-client.ts`) wrapping `fetch`:

- **Base URL:** configurable via `VITE_API_BASE_URL` env variable (default: `http://localhost:3001/v1` for local dev).
- **Default headers:** `Content-Type: application/json`.
- **Auth header:** If `accessToken` exists in the auth store, include `Authorization: Bearer {accessToken}`.
- **Tenant header:** Include `X-Tenant-ID: {user.tenantId}` from the auth store on all authenticated requests.
- **Error handling:** Parse JSON error responses. On `401` from non-auth endpoints, call `logout()` (session expired).

---

## Routing

| Route | Component | Guard |
|---|---|---|
| `/login` | `LoginPage` | Public — redirect to `/orders` if already fully authenticated |
| `/mfa` | `MfaPage` | Requires pre-MFA token (has token but `mfaVerified === false`) — redirect to `/login` if no token, redirect to `/orders` if fully authenticated |
| `/orders` | `OrderListPage` | Requires full auth (token + `mfaVerified === true` or non-MFA role) |
| `/orders/:orderId` | `OrderDetailPage` | Requires full auth |

**Auth guard implementation:** Use a React Router wrapper component (e.g., `<RequireAuth />`) or layout route that checks the auth store:
1. No token → redirect to `/login`.
2. Token present, `mfaRequired && !mfaVerified` → redirect to `/mfa`.
3. Fully authenticated → render `<Outlet />`.

**Public guard for `/login`:** If user is fully authenticated, redirect to `/orders`. This prevents authenticated users from seeing the login page.

---

## Workspace Scaffolding

This WP includes scaffolding the `admin-app` workspace from scratch. Create the following project structure:

```
workspaces/admin-app/
├── package.json
├── tsconfig.json
├── tsconfig.node.json
├── vite.config.ts
├── biome.json
├── index.html
├── public/
├── src/
│   ├── main.tsx                 # App entry point
│   ├── App.tsx                  # Router setup
│   ├── lib/
│   │   └── api-client.ts       # Fetch wrapper with auth/tenant headers
│   ├── stores/
│   │   └── auth-store.ts       # Zustand auth store
│   ├── components/
│   │   ├── shell/
│   │   │   ├── ShellLayout.tsx  # Sidebar + header + outlet
│   │   │   ├── Sidebar.tsx
│   │   │   └── Header.tsx
│   │   └── auth/
│   │       └── RequireAuth.tsx  # Auth guard wrapper
│   ├── pages/
│   │   ├── LoginPage.tsx
│   │   ├── MfaPage.tsx
│   │   ├── OrderListPage.tsx    # Placeholder — implemented in WP-008-FE
│   │   └── OrderDetailPage.tsx  # Placeholder — implemented in WP-009-FE
│   └── test/
│       ├── setup.ts             # Vitest setup (RTL, MSW server)
│       ├── mocks/
│       │   ├── handlers.ts      # MSW request handlers
│       │   └── server.ts        # MSW setupServer
│       └── pages/
│           ├── LoginPage.test.tsx
│           └── MfaPage.test.tsx
```

**Placeholder pages:** `OrderListPage` and `OrderDetailPage` should render minimal placeholder content (e.g., `<h1>Orders</h1>` and `<h1>Order Detail</h1>`). They will be fully implemented in WP-008-FE and WP-009-FE.

### package.json dependencies

**dependencies:** `react`, `react-dom`, `react-router-dom`, `@tanstack/react-query`, `zustand`

**devDependencies:** `typescript`, `vite`, `@vitejs/plugin-react`, `vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`, `jsdom`, `msw`, `@biomejs/biome`

---

## Related Contracts

- `workspaces/order-service/docs/api/openapi.json` — `POST /auth/login`, `POST /auth/mfa/verify`, `LoginRequest`, `LoginResponse`, `MfaVerifyRequest`, `MfaVerifyResponse` schemas, `bearerAuth` security scheme

### Key Schema Reference

**LoginRequest:**
```json
{ "email": "string (email)", "password": "string (min 8)" }
```

**LoginResponse:**
```json
{
  "access_token": "string (JWT)",
  "refresh_token": "string",
  "token_type": "Bearer",
  "expires_in": 3600,
  "mfa_required": true | false
}
```

**MfaVerifyRequest:**
```json
{ "code": "string (6 digits)" }
```

**MfaVerifyResponse:**
```json
{
  "access_token": "string (fully authenticated JWT)",
  "refresh_token": "string",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**ErrorResponse (401):**
```json
{ "code": "INVALID_CREDENTIALS", "message": "Invalid email or password" }
```

**Mock strategy:** Use MSW against the OpenAPI spec above during development. Switch to the real order-service URL once WP-006-BE is deployed.

---

## MSW Handlers

Implement the following MSW handlers in `src/test/mocks/handlers.ts`:

### POST /auth/login
- If `email === "viewer@example.com"` and `password === "ValidPass123!"`: return `200` with `mfa_required: false` and a mock JWT with `role: "merchant_viewer"`.
- If `email === "owner@example.com"` and `password === "ValidPass123!"`: return `200` with `mfa_required: true` and a mock pre-MFA JWT with `role: "merchant_owner"`.
- Otherwise: return `401` with `{ "code": "INVALID_CREDENTIALS", "message": "Invalid email or password" }`.

### POST /auth/mfa/verify
- If Authorization header present and `code === "123456"`: return `200` with a fully authenticated mock JWT.
- Otherwise: return `401` with `{ "code": "INVALID_MFA", "message": "Invalid verification code" }`.

Use mock JWTs that are base64-encoded JSON payloads (they don't need real signatures for FE tests — just decodable payloads with `role`, `tenant_id`, and `mfa_verified` claims).

---

## Implementation Order

1. Scaffold admin-app workspace (package.json, Vite config, TypeScript config, Biome config, index.html, `src/main.tsx`)
2. Auth Zustand store (`src/stores/auth-store.ts` — login, MFA verify, logout, token management, sessionStorage persistence)
3. API client module (`src/lib/api-client.ts` — fetch wrapper with Authorization + X-Tenant-ID headers)
4. Login page component (`src/pages/LoginPage.tsx`)
5. MFA page component (`src/pages/MfaPage.tsx`)
6. Auth guard route wrapper (`src/components/auth/RequireAuth.tsx`)
7. Shell layout (`src/components/shell/ShellLayout.tsx`, `Sidebar.tsx`, `Header.tsx`)
8. Router setup (`src/App.tsx` — wire routes, guards, shell layout)
9. Placeholder pages for OrderListPage and OrderDetailPage
10. MSW handlers for auth endpoints (`src/test/mocks/handlers.ts`, `server.ts`)
11. Vitest setup (`src/test/setup.ts`)
12. Tests for all TS scenarios (`LoginPage.test.tsx`, `MfaPage.test.tsx`, auth guard tests)
13. Run Definition of Done checklist

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-039 through TS-001-045)
- [ ] `biome check` passes with zero errors
- [ ] `tsc --noEmit` passes with zero errors
- [ ] No secrets, tokens, or credentials in code or test fixtures
- [ ] `status.yaml` `phase_4.WP-007-FE.status` set to `done`
