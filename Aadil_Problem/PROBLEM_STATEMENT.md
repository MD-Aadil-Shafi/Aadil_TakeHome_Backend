# Problem Statement — FastAPI Users: Extended User Management Backend

---

## Part 1 — Issue Description

The provided `fastapi-users` library delivers a solid authentication foundation (JWT login/logout, registration, password reset, email verification) built on SQLAlchemy and Beanie. However, it lacks several capabilities required by production-grade user management systems:

1. **No role-based access control.** Every authenticated user has identical API access. There is no way to restrict endpoints to admins, moderators, or specific user segments without hand-rolling middleware each time.

2. **No session activity tracking.** The system records no login or logout timestamps, making audit trails, "last seen" displays, and compliance reporting impossible.

3. **No app-usage metrics.** Time spent in the application is not calculated or stored, blocking engagement analytics and billing-by-usage scenarios.

4. **No rate limiting.** Authentication endpoints are unprotected against brute-force and credential-stuffing attacks. A bad actor can attempt unlimited logins without consequence.

5. **No account lockout.** Three consecutive failed login attempts carry no penalty, leaving the system vulnerable to targeted credential attacks.

6. **No AI-generated user profile.** Users must write their own "About" bio from scratch. This creates friction at onboarding and results in sparse, inconsistent profiles that reduce platform quality.

The goal of this take-home is to extend the existing `fastapi-users` SQLAlchemy example to address all six gaps above, using Google Gemini as the AI layer for bio generation.

---

## Part 2 — Requirements

### 2.1 Role-Based Access Control (RBAC)

- **R-1.1** The `User` database model must gain a `role` field of type `enum` with at minimum three values: `viewer`, `moderator`, `admin`. Default value is `viewer`.
- **R-1.2** Three FastAPI dependency callables must be implemented: `require_viewer`, `require_moderator`, `require_admin`. Each must raise `HTTP 403 Forbidden` when the authenticated user's role is insufficient.
- **R-1.3** Role hierarchy must be enforced: `admin > moderator > viewer`. A user with `admin` role must be able to access any route gated at `moderator` or `viewer` level.
- **R-1.4** Only a user with `admin` role may call `PATCH /users/{id}` to change another user's `role`. A user may never elevate their own role.
- **R-1.5** At least one example-protected route per role level must be present in the app (e.g., `GET /admin/stats`, `GET /mod/reports`, `GET /me/dashboard`).

### 2.2 Session Timestamp Tracking

- **R-2.1** The `User` model must gain two nullable `DateTime` (UTC) columns: `last_login_at` and `last_logout_at`.
- **R-2.2** `last_login_at` must be written on every successful `/auth/jwt/login` response, before the JWT is issued.
- **R-2.3** `last_logout_at` must be written on every successful `/auth/jwt/logout` call.
- **R-2.4** Both timestamps must be returned in the `GET /users/me` and `GET /users/{id}` responses (ISO 8601, UTC, nullable).
- **R-2.5** Timestamps must persist across server restarts (i.e., stored in the database, not in-memory).

### 2.3 App-Usage Time Tracking

- **R-3.1** The `User` model must gain a `total_time_spent_seconds` integer column (default `0`, non-negative).
- **R-3.2** On each successful logout, the elapsed seconds since `last_login_at` must be computed and added to `total_time_spent_seconds`. If `last_login_at` is `None` (no matching login recorded), the delta is skipped.
- **R-3.3** `GET /users/me` must expose `total_time_spent_seconds` and a derived read-only field `total_time_spent_human` formatted as `Xh Ym Zs`.
- **R-3.4** Admins must be able to retrieve usage stats for any user via `GET /admin/users/{id}/stats`.

### 2.4 Rate Limiting

- **R-4.1** All authentication endpoints (`/auth/jwt/login`, `/auth/register`, `/auth/forgot-password`) must be rate-limited globally using an in-memory or Redis-backed store.
- **R-4.2** The default limit must be **10 requests per minute per IP address** on login, and **5 requests per minute per IP** on registration and forgot-password.
- **R-4.3** When the limit is exceeded, the response must be `HTTP 429 Too Many Requests` with a `Retry-After` header indicating seconds until the window resets.
- **R-4.4** Rate-limit configuration (window size, max calls) must be injectable via environment variables (`RATE_LIMIT_LOGIN_RPM`, `RATE_LIMIT_REGISTER_RPM`) so tests can override them.

### 2.5 Account Lockout After Consecutive Failed Logins

- **R-5.1** The `User` model must gain two columns: `failed_login_attempts` (integer, default `0`) and `locked_until` (nullable `DateTime` UTC).
- **R-5.2** On each failed login attempt for a known email, `failed_login_attempts` must be incremented atomically.
- **R-5.3** When `failed_login_attempts` reaches **3**, `locked_until` must be set to `now() + 10 minutes` and `failed_login_attempts` reset to `0`.
- **R-5.4** While `locked_until > now()`, any login attempt for that account must return `HTTP 403 Forbidden` with a JSON body: `{"detail": "Account locked. Try again after <ISO timestamp>."}`.
- **R-5.5** A successful login must reset `failed_login_attempts` to `0` and `locked_until` to `None`.
- **R-5.6** Admins must be able to manually unlock an account via `POST /admin/users/{id}/unlock`.

### 2.6 AI-Generated "About" Section (Google Gemini)

- **R-6.1** The `User` model must gain three new columns: `profession` (nullable `String`), `years_of_experience` (nullable `Integer`), and `about` (nullable `Text`).
- **R-6.2** A new endpoint `POST /users/me/generate-about` must accept a JSON body `{"profession": "...", "years_of_experience": N}`, call the Google Gemini API to generate a professional bio, and persist both the inputs and the generated text to the `User` row.
- **R-6.3** The Gemini prompt must incorporate `profession`, `years_of_experience`, and the user's `email` domain to produce a contextual, first-person bio of 2–4 sentences.
- **R-6.4** The generated `about` text must be directly editable by the user via `PATCH /users/me` (same as any other user-writable field), without re-triggering Gemini.
- **R-6.5** The Gemini API key must be read from the environment variable `GEMINI_API_KEY` and never hard-coded.
- **R-6.6** If the Gemini call fails (network error, quota exceeded, invalid key), the endpoint must return `HTTP 503 Service Unavailable` with `{"detail": "AI service unavailable. Please try again later."}` and must not corrupt existing `about` data.

---

## Part 3 — Component Contract and Visual Specification

### 3.1 Updated User Schema

```
User (DB model)
├── id                      UUID, PK
├── email                   String, unique, indexed
├── hashed_password         String
├── is_active               Boolean, default=True
├── is_superuser            Boolean, default=False   [kept for library compat]
├── is_verified             Boolean, default=False
│
├── role                    Enum("viewer","moderator","admin"), default="viewer"   [NEW]
│
├── last_login_at           DateTime (UTC), nullable                               [NEW]
├── last_logout_at          DateTime (UTC), nullable                               [NEW]
├── total_time_spent_seconds Integer, default=0                                    [NEW]
│
├── failed_login_attempts   Integer, default=0                                     [NEW]
├── locked_until            DateTime (UTC), nullable                               [NEW]
│
├── profession              String, nullable                                       [NEW]
├── years_of_experience     Integer, nullable                                      [NEW]
└── about                   Text, nullable                                         [NEW]
```

### 3.2 API Endpoint Contract

| Method | Path | Auth Required | Min Role | Description |
|--------|------|--------------|----------|-------------|
| `POST` | `/auth/jwt/login` | No | — | Login; writes `last_login_at`; enforces lockout + rate limit |
| `POST` | `/auth/jwt/logout` | Yes | viewer | Logout; writes `last_logout_at`; accumulates `total_time_spent_seconds` |
| `POST` | `/auth/register` | No | — | Register; rate limited |
| `GET` | `/users/me` | Yes | viewer | Returns full user schema including timestamps and usage |
| `PATCH` | `/users/me` | Yes | viewer | Update own editable fields (including `about`, `profession`, `years_of_experience`) |
| `POST` | `/users/me/generate-about` | Yes | viewer | Calls Gemini and persists generated bio |
| `GET` | `/users/{id}` | Yes | moderator | Retrieve any user's profile |
| `PATCH` | `/users/{id}` | Yes | admin | Update any user (including `role` change) |
| `GET` | `/admin/stats` | Yes | admin | Platform-wide summary (total users, total time, active sessions) |
| `GET` | `/admin/users/{id}/stats` | Yes | admin | Per-user usage stats |
| `POST` | `/admin/users/{id}/unlock` | Yes | admin | Clear `locked_until` and reset `failed_login_attempts` |
| `GET` | `/mod/reports` | Yes | moderator | Placeholder moderation report endpoint |
| `GET` | `/me/dashboard` | Yes | viewer | Viewer-level dashboard (own data only) |

### 3.3 Login Flow (State Diagram)

```
POST /auth/jwt/login
        │
        ▼
  Is locked_until > now()?
  ┌─────┴──────┐
 YES           NO
  │             │
  ▼             ▼
HTTP 403    Verify credentials
            ┌──────┴──────┐
          WRONG          CORRECT
            │              │
            ▼              ▼
   failed_attempts++   Reset failed_attempts=0
            │           locked_until=None
   attempts == 3?      Write last_login_at=now()
   ┌────────┴──────┐        │
  YES              NO       ▼
   │               │    Issue JWT → HTTP 200
   ▼               ▼
lock 10 min   HTTP 401 (wrong password)
HTTP 401
```

### 3.4 Logout Flow

```
POST /auth/jwt/logout
        │
        ▼
  Authenticated?  ──NO──▶  HTTP 401
        │
       YES
        │
        ▼
  delta = now() - last_login_at   (skip if last_login_at is None)
  total_time_spent_seconds += delta
  last_logout_at = now()
        │
        ▼
  Invalidate token → HTTP 200
```

### 3.5 Generate-About Flow

```
POST /users/me/generate-about
  Body: { "profession": "...", "years_of_experience": N }
        │
        ▼
  Validate input (profession required, years >= 0)
        │
        ▼
  Build Gemini prompt:
    "Write a professional first-person bio in 2–4 sentences for a
     {profession} with {N} years of experience. Keep it concise."
        │
        ▼
  Call Gemini API
  ┌─────┴──────┐
FAIL          SUCCESS
  │              │
  ▼              ▼
HTTP 503     Persist profession, years_of_experience, about
             (existing `about` unchanged on failure)
                 │
                 ▼
           Return updated User schema → HTTP 200
```

### 3.6 Rate Limit Response Contract

```
HTTP 429 Too Many Requests
Headers:
  Retry-After: <seconds>
  X-RateLimit-Limit: <max_requests>
  X-RateLimit-Remaining: 0
  X-RateLimit-Reset: <unix_timestamp>
Body:
{
  "detail": "Rate limit exceeded. Try again in <N> seconds."
}
```

### 3.7 Environment Variable Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | Yes | — | SQLAlchemy async DSN |
| `JWT_SECRET` | Yes | — | Secret for signing JWT tokens |
| `GEMINI_API_KEY` | Yes | — | Google Gemini API key |
| `RATE_LIMIT_LOGIN_RPM` | No | `10` | Max login attempts per minute per IP |
| `RATE_LIMIT_REGISTER_RPM` | No | `5` | Max register attempts per minute per IP |
| `LOCKOUT_DURATION_MINUTES` | No | `10` | Account lockout duration after 3 failed logins |
| `LOCKOUT_THRESHOLD` | No | `3` | Number of consecutive failures before lockout |
