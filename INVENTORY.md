# Pusher Coin — Phase 0 Inventory

Frozen snapshot captured during Phase 0. This is a checklist Phase 1+ can
diff against, not a design document. Source files referenced by `path:line`.

Status legend (matching `ROADMAP.md`):

- `[done]` — implemented and exercised by the SPA.
- `[partial]` — scaffolding exists, behaviour incomplete.
- `[todo]` — not started.

---

## REST surface (`pc/v1`)

All five endpoints are registered through `backend/wp-content/themes/pc/app/rest-api.php`
and live in two controllers. Every route currently uses
`__return_true` / `check_permission` returning `true` — there is no
permission gating today. Phase 1 hardens this.

| Method | Path | Callback | Status |
| --- | --- | --- | --- |
| `POST` | `/pc/v1/user/sign-up/` | `UserController::create_user` (`UserController.php:55`) | `[done]` |
| `POST` | `/pc/v1/user/request-verification/` | `UserController::request_verification_code` (`UserController.php:199`) | `[done]` |
| `POST` | `/pc/v1/user/verify-code/` | `UserController::verify_code` (`UserController.php:270`) | `[done]` |
| `POST` | `/pc/v1/google-auth/authentication` | `GoogleAuthController::authenticate_with_google` (`GoogleAuthController.php:63`) | `[partial]` — `verify_google_token` instantiates `Google_Client` before checking `$google_client_id` is set (`GoogleAuthController.php:285-295`) |
| `POST` | `/pc/v1/google-auth/verify-code` | `GoogleAuthController::verify_google_code` (`GoogleAuthController.php:176`) | `[done]` |

JWT issuance/validation delegates to the bundled
`jwt-authentication-for-wp-rest-api` plugin and the `firebase/php-jwt`
library. The custom theme hooks `jwt_auth_expire`,
`jwt_auth_token_before_sign`, and `jwt_auth_token_before_dispatch` filters
in the verification endpoints.

Apple sign-in: not implemented. Logout, refresh, terms-acceptance,
account-page endpoints, rooms, wallet, transactions, support: not
implemented.

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

Future keys (terms acceptance, email confirmation, Apple ID, etc.) must be
added to `User_Meta_Keys` and to `DATA-MODEL.md`.

## Frontend stub inventory

Pages and components in `frontend/src/` that render placeholder data and
are not yet wired to the API. Phase 4–7 owners will replace these with
real API calls.

- `views/AccountView.vue` — hardcoded nickname/balance, forms have no
  submission handlers, no `/me`-style fetch.
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

## CI status

- Frontend: `frontend/.github/workflows/ci.yml` — `npm ci`, `npm run lint`,
  `npm run build` on every push and PR. New in Phase 0.
- Backend: `backend/.github/workflows/main.yml` — `lint` job runs
  `php -l` over `wp-content/themes/pc`; `deploy` job (FTP) is gated by
  `needs: lint` and only runs on `push` to `main`. Updated in Phase 0
  (was deploy-only before).
