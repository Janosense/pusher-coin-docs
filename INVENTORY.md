# Pusher Coin — Inventory

Snapshot of the REST surface, persistence, and frontend stub state.
Updated by every phase that lands. Source files referenced by `path:line`.

History:
- Phase 0 — captured the original audit (5 endpoints, drift fixes, meta-key registry).
- Phase 1 — added auth hardening (logout, refresh, accept-terms, set-nickname, Apple stub), refresh-token store, audit log, terms gate, blocking nickname picker, inactivity timer, server-side logout.
- Phase 2 — wired the account page to real data, added email confirmation (link-based, 24h TTL), inline two-step password change, email-verified gate in `Permissions::require_play_ready`, faceless SVG avatar, Google 2FA static info panel.

Status legend (matching `ROADMAP.md`):

- `[done]` — implemented and exercised by the SPA.
- `[partial]` — scaffolding exists, behaviour incomplete.
- `[todo]` — not started.

---

## REST surface (`pc/v1`)

Endpoints are registered through `backend/wp-content/themes/pc/app/rest-api.php`
and live in four controllers. Public auth endpoints are gated by
`Rate_Limiter`; bearer-required endpoints use `Permissions::*` callbacks.

| Method | Path | Callback | Status |
| --- | --- | --- | --- |
| `POST` | `/pc/v1/user/sign-up/` | `UserController::create_user` | `[done]` (Phase 1: T&C gate, rate limit, audit) |
| `POST` | `/pc/v1/user/request-verification/` | `UserController::request_verification_code` | `[done]` (Phase 1: rate limit, audit) |
| `POST` | `/pc/v1/user/verify-code/` | `UserController::verify_code` | `[done]` (Phase 1: returns auth envelope) |
| `POST` | `/pc/v1/user/accept-terms` | `UserController::accept_terms` | `[done]` (Phase 1) |
| `POST` | `/pc/v1/user/set-nickname` | `UserController::set_nickname` | `[done]` (Phase 1) |
| `POST` | `/pc/v1/google-auth/authentication` | `GoogleAuthController::authenticate_with_google` | `[done]` (Phase 1: rate limit, audit, ordering bug fixed) |
| `POST` | `/pc/v1/google-auth/verify-code` | `GoogleAuthController::verify_google_code` | `[done]` (Phase 1: returns auth envelope, sets `nickname_required`) |
| `POST` | `/pc/v1/apple-auth/authentication` | `AppleAuthController::authenticate_with_apple` | `[partial]` — stub returning `apple_not_configured` until Apple Developer enrollment |
| `POST` | `/pc/v1/apple-auth/verify-code` | `AppleAuthController::verify_apple_code` | `[partial]` — same gate |
| `POST` | `/pc/v1/auth/logout` | `AuthController::logout` | `[done]` (Phase 1) |
| `POST` | `/pc/v1/auth/refresh` | `AuthController::refresh` | `[done]` (Phase 1; rotates refresh tokens, detects reuse) |
| `GET` / `PATCH` | `/pc/v1/user/me` | `UserController::get_me` / `patch_me` | `[done]` (Phase 2; canonical account shape) |
| `POST` | `/pc/v1/user/request-email-confirmation` | `UserController::request_email_confirmation` | `[done]` (Phase 2; rate-limited) |
| `POST` | `/pc/v1/user/confirm-email` | `UserController::confirm_email` | `[done]` (Phase 2; public, token-as-credential) |
| `POST` | `/pc/v1/user/request-password-change` | `UserController::request_password_change` | `[done]` (Phase 2; rate-limited) |
| `POST` | `/pc/v1/user/confirm-password-change` | `UserController::confirm_password_change` | `[done]` (Phase 2; revokes other refresh tokens, returns new pair) |

JWT issuance/validation delegates to the bundled
`jwt-authentication-for-wp-rest-api` plugin and the `Tmeister\Firebase\JWT`
library. Phase 1 introduces a 15-minute access TTL (controlled by the
`pc_access_token_ttl_seconds` option, read by both
`AuthController::issue_access_token` and the `jwt_auth_expire` filter).
Refresh tokens are stored hashed in `wp_pc_refresh_tokens`.

Account-page endpoints, rooms, wallet, transactions, support: still
deferred to later phases.

## Player role

Registered in `backend/wp-content/themes/pc/app/utils/role-player.php:10-21`.

- Slug: `player`.
- Caps: `{ read: false, play: true }`.
- The `play` capability is also added to `administrator` (line 12).
- Hooked on `init` (line 21).

## User meta keys

All meta-key string literals were consolidated into
`backend/wp-content/themes/pc/app/utils/user-meta-keys.php` during Phase 0.
The `User_Meta_Keys` class is the source of truth; controllers reference
constants only.

| Constant | Storage key | Owning controller | Purpose |
| --- | --- | --- | --- |
| `User_Meta_Keys::PHONE` | `phone` | `UserController` | Optional phone, captured at sign-up. |
| `User_Meta_Keys::VERIFICATION_CODE` | `verification_code` | `UserController` | 6-digit email-code 2FA. |
| `User_Meta_Keys::VERIFICATION_CODE_EXPIRY` | `verification_code_expiry` | `UserController` | Unix timestamp, 15-minute TTL. |
| `User_Meta_Keys::GOOGLE_ID` | `google_id` | `GoogleAuthController` | Google account `sub`. |
| `User_Meta_Keys::GOOGLE_VERIFICATION_CODE` | `google_verification_code` | `GoogleAuthController` | 6-digit email-code 2FA for Google flow. |
| `User_Meta_Keys::GOOGLE_VERIFICATION_CODE_EXPIRY` | `google_verification_code_expiry` | `GoogleAuthController` | Unix timestamp, 15-minute TTL. |
| `User_Meta_Keys::TERMS_ACCEPTED_AT` | `terms_accepted_at` | `UserController` (Phase 1) | Unix timestamp; required for play / top-up. |
| `User_Meta_Keys::TERMS_ACCEPTED_VERSION` | `terms_accepted_version` | `UserController` (Phase 1) | Compared against `pc_terms_current_version` option. |
| `User_Meta_Keys::NICKNAME_CHOSEN` | `nickname_chosen` | `UserController` (Phase 1) | `'1'` once user has picked a nickname. |
| `User_Meta_Keys::APPLE_ID` | `apple_id` | `AppleAuthController` (stub) | Apple `sub` claim. |
| `User_Meta_Keys::APPLE_VERIFICATION_CODE` | `apple_verification_code` | `AppleAuthController` (stub) | Mirrors Google. |
| `User_Meta_Keys::APPLE_VERIFICATION_CODE_EXPIRY` | `apple_verification_code_expiry` | `AppleAuthController` (stub) | |
| `User_Meta_Keys::EMAIL_VERIFIED_AT` | `email_verified_at` | `UserController` (Phase 2) | Unix timestamp; required by `Permissions::require_play_ready`. |
| `User_Meta_Keys::EMAIL_CONFIRMATION_TOKEN` | `email_confirmation_token` | `UserController` (Phase 2) | URL-safe base64; cleared on confirm/expiry. |
| `User_Meta_Keys::EMAIL_CONFIRMATION_EXPIRY` | `email_confirmation_expiry` | `UserController` (Phase 2) | 24-hour TTL. |
| `User_Meta_Keys::PASSWORD_CHANGE_CODE` | `password_change_code` | `UserController` (Phase 2) | 6-digit code; cleared on confirm/expiry. |
| `User_Meta_Keys::PASSWORD_CHANGE_CODE_EXPIRY` | `password_change_code_expiry` | `UserController` (Phase 2) | 15-minute TTL. |

New custom tables (Phase 1):

- `wp_pc_refresh_tokens` — active refresh tokens (hashed). Installed by `Install_Schema`.
- `wp_pc_auth_audit_log` — append-only auth event log. Installed by `Install_Schema`.

Future keys (email confirmation, etc.) must be added to `User_Meta_Keys`
and to `DATA-MODEL.md`.

## Frontend stub inventory

Pages and components in `frontend/src/` that render placeholder data and
are not yet wired to the API. Phase 4–7 owners will replace these with
real API calls.

- ~~`views/AccountView.vue` — hardcoded nickname/balance, forms have no
  submission handlers, no `/me`-style fetch.~~ Replaced in Phase 2 with a
  full data-driven page (avatar, editable nickname, email-verify flow,
  inline two-step password change, Google 2FA panel, gated balance buttons).
- `views/HistoryView.vue` — table renders 50 empty placeholder rows,
  no fetch.
- `views/RoomView.vue` — embedded YouTube iframe stand-in for the live
  broadcast; no room-data fetch.
- `views/SupportView.vue` — form has `action=""` and no handler.
- `components/Chat.vue` — six hardcoded messages; `sendMessage` mutates
  a local array.
- `components/Queue.vue` — 24 fake users, two with hardcoded balance.
- `components/ReplenishmentBalance.vue` — input is wired (IMask), the
  submit button has no handler.
- `components/PlaceBet.vue` — same as above.
- `components/Rooms.vue` — placeholder list (rooms data is hardcoded).

## Frontend ↔ backend contract drift (resolved in Phase 0)

Both fixes landed during Phase 0 — listed here for the audit trail.

| Field | Old (frontend) | Backend | Resolution |
| --- | --- | --- | --- |
| Path | `POST /google-auth/verify` (`services/authService.js`) | `POST /google-auth/verify-code` (`GoogleAuthController.php:37`) | SPA renamed to `verify-code`. |
| Body field | `{ id_token, code }` (`services/authService.js`) | `{ id_token, verification_code }` (`GoogleAuthController.php:178`) | SPA now sends `verification_code`. |
| Response flag | `requires_2fa \|\| requires_verification` (`services/googleAuthService.js`) | `requires_verification` only (`GoogleAuthController.php:163`) | SPA reads `requires_verification`; internal alias renamed to `requiresVerification`; emit/event renamed to `requires-verification`. |

## Phase 1 highlights

- **Auth envelope** — `verify-code`, `google-auth/verify-code`,
  `apple-auth/verify-code` (stub), and `auth/refresh` all return
  `{ access_token, access_token_expires_in, refresh_token,
  refresh_token_expires_in, user_*, terms_accepted, nickname_required }`.
  The legacy single-`token` field is gone.
- **Refresh-on-401** — `frontend/src/services/api.js` performs a single
  refresh attempt on every 401, retrying the failing request once.
  Concurrent 401s share the in-flight refresh.
- **Inactivity timer** — `services/sessionService.js` watches mouse,
  keyboard, click, focus, and visibility; 15-minute idle window calls
  `authStore.logout(true)`.
- **Server-side logout** — `authStore.logout` calls `/auth/logout` with
  the refresh token, then clears local state.
- **Router gates** — guard redirects authenticated users to
  `/choose-nickname` when `nickname_required`, then `/accept-terms` when
  `!terms_accepted`, before any other authenticated route is reachable.

## Phase 2 highlights

- **Account page wired to real data.** `AccountView.vue` calls
  `accountService.getMe()` on mount; nickname is click-to-edit; balances
  pull from the (placeholder) wallet fields; the email verify badge
  reflects `email_verified`.
- **Faceless avatar** — `FacelessAvatar.vue` renders a deterministic
  5×5 SVG mosaic seeded from `user.id`. No new dependency.
- **Email confirmation** — link-based, 24-hour TTL. Outbound link is
  built from `pc_spa_base_url` + `/confirm-email?token=…`. `ConfirmEmailView.vue`
  handles the redemption and offers a "Send a new link" button on
  failure.
- **Inline two-step password change** — Account page submits current +
  new password, mails a 6-digit code, then redeems on the same page.
  Successful change revokes all other refresh tokens; the SPA gets a new
  pair in the response and applies it via `authStore.applyEnvelope`.
- **Email-verified gate.** `Permissions::require_play_ready` now also
  enforces `email_verified_at > 0`. The router refuses
  `meta.requiresPlayReady` routes for unverified users, redirecting to
  `/account?reason=verify-email` (the page scrolls to the red banner).
- **Auth envelope** — adds `email_verified` so the SPA can render the
  red dot on first login without an extra fetch.

## CI status

- Frontend: `frontend/.github/workflows/ci.yml` — `npm ci`, `npm run lint`,
  `npm run build` on every push and PR. New in Phase 0.
- Backend: `backend/.github/workflows/main.yml` — `lint` job runs
  `php -l` over `wp-content/themes/pc`; `deploy` job (FTP) is gated by
  `needs: lint` and only runs on `push` to `main`. Updated in Phase 0
  (was deploy-only before).
