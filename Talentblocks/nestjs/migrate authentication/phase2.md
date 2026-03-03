# PRD: Phase 2 — Magic Link + OAuth Migration

  

> Builds on: `docs/prd/authentication/setup-nestjs-authentication-phase1.md`

> Last updated: 2026-02-24

  

---

  

## 1. Background & Problem

  

Phase 1 established the NestJS authentication skeleton: JWT access/refresh tokens, session cookies, user provisioning, and `/auth/me`. The frontend still relies entirely on Supabase Auth for the two actual login paths:

  

- **Magic Link**: `supabase.auth.signInWithOtp(...)` + `/verify` + `supabase.auth.verifyOtp(...)`

- **OAuth popup**: `supabase.auth.signInWithOAuth({ skipBrowserRedirect: true })` + `onAuthStateChange` to collect the session

  

This phase migrates both login paths into NestJS, eliminating the Supabase Auth dependency for user-facing authentication. The Supabase Postgres database is retained.

  

---

  

## 2. Goals

  

Phase 2 must deliver two independently testable sub-milestones:

  

### 2A — Magic Link

  

1. NestJS issues a one-time login link via email (no Supabase Auth OTP).

2. NestJS verifies the link, provisions the user, and issues session cookies.

3. Frontend can complete a full login with `POST /auth/magic-link` → email click → `GET /auth/me`.

  

### 2B — OAuth (Google / Azure / GitHub / LinkedIn)

  

1. Frontend opens a popup to `GET /auth/oauth/:provider/start` (NestJS-driven OAuth, no Supabase).

2. NestJS handles the callback, provisions the user, issues session cookies.

3. Popup signals the main window via `postMessage`; main window calls `GET /auth/me` to initialise state.

4. Popup UX is preserved — no full-page redirect visible to the user.

  

---

  

## 3. Non-Goals

  

- ❌ No removal of the existing `POST /auth/login` endpoint (provider access-token flow stays for now).

- ❌ No business API migration (Phase 4).

- ❌ No Supabase RLS changes (policy review deferred).

- ❌ No multi-device session management UI.

- ❌ No account-merge strategy beyond existing `findOrCreateUserByIdentity` logic.

  

---

  

## 4. Stakeholders

  

- Engineering: full-stack lead, backend reviewer

- Product: login UX must be preserved (popup OAuth, magic link email)

- Security: cookie strategy, token expiry, free-email domain block rules

  

---

  

## 5. User Stories

  

### Magic Link

  

1. As a user, I enter my email on the login page and receive a one-time login link that signs me in automatically when clicked.

2. As a user, if my email is from a blocked free-provider domain (e.g. gmail.com for a `client` user type), I see a clear error before any email is sent.

3. As the system, once the magic-link token is used or expired, any replay attempt must be rejected.

  

### OAuth

  

4. As a user, I click "Sign in with Google" and a popup opens; after I authenticate I am logged in to the main tab without a full page reload.

5. As the system, after the OAuth callback I provision (or link) the user identity and issue session cookies before the popup closes.

6. As a developer, I can test any supported provider (Google, Azure, GitHub, LinkedIn) end-to-end in a local environment.

  

---

  

## 6. Requirements & Constraints

  

### 6.1 Technology Constraints

  

- NestJS with Passport (`passport-magic-link` or custom strategy for magic link; `passport-google-oauth20`, `passport-azure-ad`, `passport-github2`, `passport-linkedin-oauth2` for OAuth).

- Database: Supabase Postgres via existing TypeORM connection.

- Email: use a transactional mail provider (e.g. Resend, SendGrid, Nodemailer + SMTP). Provider is configurable via env; template management is out of scope.

- OAuth credentials (client ID / secret) per provider stored in env, never in code.

  

### 6.2 Security Constraints

  

- **Magic Link token**:

- High-entropy random value (≥ 32 bytes, URL-safe base64).

- Only the SHA-256 hash is stored in DB; plaintext sent in email only.

- Single-use: marked `used_at` on first successful verify; any subsequent use → 400 `MAGIC_LINK_ALREADY_USED`.

- Expiry: 15 minutes from issuance.

- Rate-limited: maximum 3 magic-link requests per email per 10-minute window.

- **OAuth state parameter**:

- Cryptographically random per-request `state` value, stored server-side (short-lived, e.g. Redis or DB with 10-minute TTL).

- Callback must validate `state` before processing the code exchange.

- **Free-email domain block** (Magic Link):

- Mirrors the existing frontend Redux rules.

- Rule list is server-side configuration (env or file); not hardcoded in service logic.

- Applied in `MagicLinkService` before token generation or email send.

- If `userType` is `client`, known free-email domains (gmail.com, yahoo.com, hotmail.com, outlook.com, etc.) are rejected with `MAGIC_LINK_FREE_EMAIL_BLOCKED`.

- **OAuth popup postMessage origin check**:

- Popup page must only `postMessage` to a whitelisted origin (`CORS_ORIGIN` / `FRONTEND_URL` env).

- **PKCE** (optional, recommended for public clients):

- If any provider supports PKCE, use it for the authorization code exchange.

  

---

  

## 7. High-Level Solution

  

### 7A — Magic Link

  

```

Frontend NestJS DB Email

│ │ │ │

│─ POST /auth/magic-link ──────►│ │ │

│ { email, userType } │── domain-block check │ │

│ │── generate token (random) │ │

│ │── hash token ──────────────►│ INSERT │

│ │ │ magic_link_ │

│ │ │ tokens │

│ │── send email ──────────────────────────────►│

│◄─ 200 { ok: true } ──────────│ │ │

│ │ │ │

│ (user clicks link in email) │ │ │

│─ GET /auth/verify?token=... ─►│ │ │

│ │── hash token ──────────────►│ SELECT + lock │

│ │── validate (expiry, used) │ │

│ │── mark used_at ────────────►│ UPDATE │

│ │── findOrCreateUserByIdentity►│ │

│ │── issue access + refresh ───│ │

│ │── set cookies │ │

│◄─ 302 → /[type]/profile ─────│ │ │

```

  

### 7B — OAuth (popup flow)

  

```

Main Window Popup NestJS Provider

│ │ │ │

│─ window.open ───►│ │ │

│ /auth/oauth/ │ │ │

│ :provider/start │ │ │

│ │─ GET /auth/oauth/───────►│ │

│ │ :provider/start │── generate state ────────│

│ │ │── store state (TTL 10m) │

│ │◄─ 302 to provider ───────│ │

│ │── auth flow ────────────────────────────────────► │

│ │◄── redirect with code+state ────────────────────── │

│ │─ GET /auth/oauth/ ───────►│ │

│ │ :provider/callback │── validate state │

│ │ │── exchange code ────────►│

│ │ │◄── id_token + profile ───│

│ │ │── findOrCreateUser │

│ │ │── issue cookies │

│ │◄─ HTML page (postMessage)─│ │

│◄─ message { ok } │ │ │

│─ GET /auth/me ──────────────────────────────►│ │

│◄─ { user, ... } ────────────────────────────│ │

│── Redux init / redirect │ │

```

  

---

  

## 8. New Data Model

  

### 8.1 magic_link_tokens

  

```sql

CREATE TABLE magic_link_tokens (

id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

token_hash TEXT NOT NULL UNIQUE, -- SHA-256 of plaintext token

email TEXT NOT NULL,

user_type TEXT, -- 'freelancer' | 'client' | NULL

expires_at TIMESTAMPTZ NOT NULL,

used_at TIMESTAMPTZ,

created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

);

  

CREATE INDEX idx_magic_link_tokens_email_created

ON magic_link_tokens (email, created_at); -- for rate-limit queries

```

  

### 8.2 oauth_states (short-lived)

  

```sql

CREATE TABLE oauth_states (

state TEXT PRIMARY KEY, -- random value

provider TEXT NOT NULL,

nonce TEXT, -- optional PKCE / nonce

expires_at TIMESTAMPTZ NOT NULL,

created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

);

```

  

> A scheduled cleanup job (extend the existing `refresh-token-cleanup.service.ts` pattern) should purge expired rows from both tables.

  

### 8.3 Existing table changes

  

No schema changes required to `users`, `auth_identities`, or `refresh_tokens`. The `auth_identities.provider` column already accepts `'email'` as a value; magic-link identities will use `provider = 'email'` with `provider_subject = SHA-256(email)` (consistent with Phase 1 email identity convention).

  

---

  

## 9. API Design

  

> All endpoints are under the `/auth` prefix.

  

### 9A — Magic Link Endpoints

  

#### POST /auth/magic-link

  

**Purpose**: Request a one-time login link.

  

- **Auth**: none (public)

- **Rate limit**: 3 requests / email / 10 min (ThrottlerGuard with custom key)

- **Request body**:

  

```json

{

"email": "alice@company.com",

"userType": "freelancer"

}

```

  

- **Behaviour**:

1. Validate email format (class-validator `@IsEmail()`).

2. If `userType === 'client'`, check email domain against the free-email block list → reject with `MAGIC_LINK_FREE_EMAIL_BLOCKED` if matched.

3. Generate 32-byte random token (URL-safe base64), compute SHA-256 hash.

4. INSERT into `magic_link_tokens` with 15-minute `expires_at`.

5. Send email with link: `${APP_BASE_URL}/auth/verify?token=<plaintext>`.

6. Always return 200 (no enumeration of whether the email exists).

  

- **Response 200**:

  

```json

{ "ok": true }

```

  

- **Response 400** (domain block):

  

```json

{ "code": "MAGIC_LINK_FREE_EMAIL_BLOCKED", "message": "Please use a work email address." }

```

  

- **Response 429**: rate limit exceeded.

  

---

  

#### GET /auth/verify

  

**Purpose**: Validate the magic-link token and start a session.

  

- **Auth**: none (public)

- **Query param**: `token` (plaintext)

- **Behaviour**:

1. Hash the `token` parameter (SHA-256).

2. SELECT from `magic_link_tokens` WHERE `token_hash = ?` (with row lock).

3. If not found → 400 `MAGIC_LINK_INVALID`.

4. If `used_at IS NOT NULL` → 400 `MAGIC_LINK_ALREADY_USED`.

5. If `expires_at < NOW()` → 400 `MAGIC_LINK_EXPIRED`.

6. UPDATE `used_at = NOW()`.

7. `userProvisioningService.findOrCreateUserByIdentity({ provider: 'email', providerSubject: sha256(email), email, emailVerified: true, userType })`.

8. Issue access token + refresh token (same as Phase 1 `signInWithEmail` path).

9. Set `tb_at` / `tb_rt` cookies.

10. Redirect 302 to `/${userType}/profile` (or a configurable `redirectUrl`).

  

- **Response 302**: redirect on success.

- **Response 400**: invalid / used / expired token.

  

---

  

### 9B — OAuth Endpoints

  

#### GET /auth/oauth/:provider/start

  

**Purpose**: Initiate the OAuth authorization code flow.

  

- **Auth**: none (public)

- **Path param**: `provider` ∈ `{ google, azure, github, linkedin }`

- **Behaviour**:

1. Validate `provider` against the allowed list → 400 `OAUTH_PROVIDER_UNSUPPORTED` if unknown.

2. Generate a random `state` value (32 bytes, URL-safe base64).

3. Store `state` in `oauth_states` with 10-minute TTL.

4. Build the provider authorization URL with `state`, `redirect_uri`, and required scopes.

5. Redirect 302 to the provider.

  

- **Response 302**: redirect to provider login page.

  

---

  

#### GET /auth/oauth/:provider/callback

  

**Purpose**: Handle the authorization code callback from the provider.

  

- **Auth**: none (public — called by provider redirect)

- **Path param**: `provider`

- **Query params**: `code`, `state` (and optionally `error`)

- **Behaviour**:

1. If `error` query param is set → render the error popup page (see §9B.1).

2. Look up `state` in `oauth_states`; if not found or expired → 400 `OAUTH_STATE_INVALID`.

3. Delete the `oauth_states` row (one-time use).

4. Exchange `code` for tokens via the provider's token endpoint.

5. Fetch or decode the user profile (OIDC `id_token` or `/userinfo` endpoint, consistent with existing `ProviderVerificationService` claims extraction).

6. `userProvisioningService.findOrCreateUserByIdentity({ provider, providerSubject, email, emailVerified, userType: null })`.

7. Issue access token + refresh token; set `tb_at` / `tb_rt` cookies.

8. Render the "popup close" HTML page (see §9B.1).

  

- **Response 200**: inline HTML page that calls `postMessage` and closes the popup.

- **Response 400**: state invalid, code exchange failure.

  

##### 9B.1 — Popup Close Page (HTML response)

  

```html

<!DOCTYPE html>

<html>

<head><title>Signing in…</title></head>

<body>

<script>

if (window.opener) {

window.opener.postMessage({ ok: true }, '__FRONTEND_ORIGIN__');

}

window.close();

</script>

</body>

</html>

```

  

On error:

  

```html

<script>

if (window.opener) {

window.opener.postMessage({ ok: false, code: '__ERROR_CODE__' }, '__FRONTEND_ORIGIN__');

}

window.close();

</script>

```

  

`__FRONTEND_ORIGIN__` is injected server-side from the `FRONTEND_URL` / `CORS_ORIGIN` env variable. It must never be `*`.

  

---

  

## 10. New Services

  

### MagicLinkService

  

Responsibilities:

- `requestMagicLink(email, userType)` — domain block, token generation, DB insert, email send.

- `verifyMagicLink(plaintextToken)` — hash, DB lookup + lock, validate, mark used, return `{ email, userType }`.

- `generateToken()` — private: 32-byte crypto random, URL-safe base64.

- Free-email domain list loaded from config (env variable `FREE_EMAIL_DOMAINS` as comma-separated list, with a sensible default list).

  

### OAuthService

  

Responsibilities:

- `buildAuthorizationUrl(provider, state)` — returns the provider authorization URL.

- `exchangeCode(provider, code, redirectUri)` — code → tokens → profile.

- `extractClaims(provider, tokenResponse)` — reuse / extend `ProviderVerificationService` logic.

- `generateState()` + `storeState(state, provider)` + `validateAndConsumeState(state)`.

  

### EmailService

  

Responsibilities:

- `sendMagicLink(to, link)` — thin wrapper around the configured mail transport.

- Transport is injected via env (`EMAIL_TRANSPORT`: `smtp` | `resend` | `sendgrid`).

- Template: plain-text + HTML; kept minimal for Phase 2.

  

---

  

## 11. Module Changes

  

```

src/auth/

auth.module.ts ← add MagicLinkService, OAuthService, EmailService, OAuthController

auth.controller.ts ← add POST /magic-link, GET /verify

oauth.controller.ts ← new: GET /oauth/:provider/start, GET /oauth/:provider/callback

services/

magic-link.service.ts ← new

oauth.service.ts ← new

email.service.ts ← new

entities/

magic-link-token.entity.ts ← new

oauth-state.entity.ts ← new

dto/

magic-link.dto.ts ← new (RequestMagicLinkDto, VerifyMagicLinkDto)

templates/

magic-link.html ← email template

oauth-popup-success.html ← popup close page

oauth-popup-error.html ← popup close page (error)

```

  

---

  

## 12. Environment Variables (New)

  

```

# Magic Link

MAGIC_LINK_BASE_URL=https://api.example.com # base for /auth/verify link

MAGIC_LINK_TTL_MINUTES=15 # default 15

MAGIC_LINK_RATE_LIMIT_PER_10MIN=3 # default 3

FREE_EMAIL_DOMAINS=gmail.com,yahoo.com,hotmail.com,outlook.com,...

  

# Email transport

EMAIL_TRANSPORT=smtp # smtp | resend | sendgrid

EMAIL_FROM=noreply@talentblocks.io

SMTP_HOST=

SMTP_PORT=587

SMTP_USER=

SMTP_PASS=

RESEND_API_KEY=

SENDGRID_API_KEY=

  

# OAuth (per provider)

GOOGLE_CLIENT_ID=

GOOGLE_CLIENT_SECRET=

GOOGLE_CALLBACK_URL=https://api.example.com/auth/oauth/google/callback

  

AZURE_CLIENT_ID=

AZURE_CLIENT_SECRET=

AZURE_TENANT_ID=

AZURE_CALLBACK_URL=https://api.example.com/auth/oauth/azure/callback

  

GITHUB_CLIENT_ID=

GITHUB_CLIENT_SECRET=

GITHUB_CALLBACK_URL=https://api.example.com/auth/oauth/github/callback

  

LINKEDIN_CLIENT_ID=

LINKEDIN_CLIENT_SECRET=

LINKEDIN_CALLBACK_URL=https://api.example.com/auth/oauth/linkedin/callback

  

# Frontend origin for postMessage

FRONTEND_URL=https://app.example.com

```

  

---

  

## 13. Error Codes (New)

  

Extend the existing `AuthErrorCode` enum:

  

```typescript

// Magic Link

MAGIC_LINK_FREE_EMAIL_BLOCKED = 'MAGIC_LINK_FREE_EMAIL_BLOCKED',

MAGIC_LINK_INVALID = 'MAGIC_LINK_INVALID',

MAGIC_LINK_EXPIRED = 'MAGIC_LINK_EXPIRED',

MAGIC_LINK_ALREADY_USED = 'MAGIC_LINK_ALREADY_USED',

MAGIC_LINK_RATE_LIMITED = 'MAGIC_LINK_RATE_LIMITED',

  

// OAuth

OAUTH_PROVIDER_UNSUPPORTED = 'OAUTH_PROVIDER_UNSUPPORTED',

OAUTH_STATE_INVALID = 'OAUTH_STATE_INVALID',

OAUTH_CODE_EXCHANGE_FAILED = 'OAUTH_CODE_EXCHANGE_FAILED',

OAUTH_PROFILE_UNAVAILABLE = 'OAUTH_PROFILE_UNAVAILABLE',

```

  

---

  

## 14. Security Notes

  

| Concern | Mitigation |

|---------|-----------|

| Magic link replay | `used_at` set on first use; subsequent use → 400 |

| Magic link enumeration | Always return 200 on `POST /auth/magic-link` |

| OAuth CSRF | `state` param validated against server-side store |

| Open redirect | Redirect targets are hard-coded or validated against an allowlist |

| postMessage origin | `FRONTEND_URL` env var; never `*` |

| Free-email block bypass | Domain check is server-side, not client-side |

| Token storage | Magic link hash only in DB; OAuth `state` purged after use |

| Refresh token security | Unchanged from Phase 1 (rotation, family tracking, HttpOnly cookie) |

  

---

  

## 15. Test Plan

  

### Unit Tests

  

- `MagicLinkService`:

- Domain block fires for `client` + free-email domain.

- Domain block does not fire for `freelancer` + any domain.

- Token hash stored, plaintext never persisted.

- Expired token → `MAGIC_LINK_EXPIRED`.

- Used token → `MAGIC_LINK_ALREADY_USED`.

- Unknown token → `MAGIC_LINK_INVALID`.

  

- `OAuthService`:

- Unknown provider → `OAUTH_PROVIDER_UNSUPPORTED`.

- Invalid / missing state → `OAUTH_STATE_INVALID`.

- State is single-use (second validation fails).

- Claims extraction for each provider (Google, Azure, GitHub, LinkedIn).

  

- `EmailService`:

- `sendMagicLink` calls the transport with correct recipient and link.

  

### E2E Tests

  

#### Magic Link (6 cases)

  

| # | Scenario | Expected |

|---|----------|----------|

| 1 | Valid email + userType → verify → `/auth/me` | 200 with user |

| 2 | Replay same token | 400 `MAGIC_LINK_ALREADY_USED` |

| 3 | Expired token (mock TTL to 0) | 400 `MAGIC_LINK_EXPIRED` |

| 4 | Invalid token (garbage string) | 400 `MAGIC_LINK_INVALID` |

| 5 | `client` + free-email domain | 400 `MAGIC_LINK_FREE_EMAIL_BLOCKED` |

| 6 | Rate limit: 4th request within window | 429 |

  

#### OAuth (4 cases, per supported provider in CI)

  

| # | Scenario | Expected |

|---|----------|----------|

| 1 | Full flow (mock provider) → popup close page rendered | 200 HTML with `postMessage` |

| 2 | Missing / tampered `state` | 400 `OAUTH_STATE_INVALID` |

| 3 | Provider returns `error` in callback | Error popup page rendered |

| 4 | Returning user (same identity, second login) | 200, same `user.id` as first login |

  

---

  

## 16. Acceptance Criteria

  

### Phase 2A — Magic Link

  

- [ ] `POST /auth/magic-link` returns 200 and an email is sent (verified in test via mail transport mock / test inbox).

- [ ] Clicking the link (i.e., `GET /auth/verify?token=...`) sets `tb_at` / `tb_rt` cookies and redirects correctly.

- [ ] `/auth/me` returns the provisioned user after a magic-link login.

- [ ] Replayed or expired tokens are rejected with correct error codes.

- [ ] Free-email block for `client` users is enforced server-side.

- [ ] Magic link does not rely on Supabase Auth in any code path.

  

### Phase 2B — OAuth

  

- [ ] `GET /auth/oauth/:provider/start` redirects to the correct provider for all four supported providers.

- [ ] After provider callback, `tb_at` / `tb_rt` cookies are set and the popup-close page is rendered.

- [ ] Main window receives `{ ok: true }` via `postMessage`, calls `/auth/me`, and gets a valid user.

- [ ] Invalid `state` is rejected; provider errors surface gracefully via `postMessage { ok: false }`.

- [ ] OAuth flow does not rely on Supabase Auth in any code path.

- [ ] All four providers (Google, Azure, GitHub, LinkedIn) are tested end-to-end (staging environment).

  

---

  

## 17. Milestones & Deliverables

  

### Milestones

  

| ID | Milestone | Gate |

|----|-----------|------|

| M1 | `MagicLinkService` + DB migration + email mock | Unit tests pass |

| M2 | `POST /auth/magic-link` + `GET /auth/verify` wired and tested | E2E magic-link cases 1–6 pass |

| M3 | `OAuthService` + state table + provider passport strategies registered | Unit tests pass |

| M4 | `GET /auth/oauth/:provider/start` + `GET /auth/oauth/:provider/callback` wired | E2E OAuth cases 1–4 pass (at least Google in CI) |

| M5 | All four OAuth providers verified in staging | Staging sign-off |

  

### Deliverables

  

- NestJS `MagicLinkService`, `OAuthService`, `EmailService` + controller endpoints

- DB migrations for `magic_link_tokens` and `oauth_states`

- Email HTML template

- Popup HTML templates (success + error)

- Updated `AuthErrorCode` constants

- Unit tests (colocated `.spec.ts`)

- E2E test additions to the existing `test/` suite

- Updated Postman / Swagger collection