# Pusher Coin — API Contract

Single source of truth for the `pc/v1` REST namespace. Anything new the SPA,
admin SPA, or background workers consume must be specified here before it
ships.

The contract has two halves: **conventions** (the rules every endpoint
follows) and **endpoint catalogues** (the current 5 endpoints, plus
phase-by-phase stubs for what's coming).

---

## Conventions

### Base URL & versioning

- Base URL: `https://<host>/wp-json/pc/v1`.
- The version segment (`v1`) is part of the URL. Breaking shape changes go
  to a new namespace (`pc/v2`); additive changes stay on `v1`.

### Authentication

- Bearer JWT in the `Authorization` header: `Authorization: Bearer <token>`.
- Tokens come in pairs (Phase 1):
  - **Access JWT** — HS256, signed with `JWT_AUTH_SECRET_KEY`, TTL controlled
    by the `pc_access_token_ttl_seconds` WP option (default 900 = 15min).
  - **Refresh token** — opaque 256-bit random string, rotates on every
    `/auth/refresh`, stored server-side hashed in `wp_pc_refresh_tokens`,
    TTL controlled by `pc_refresh_token_ttl_seconds` (default 7 days).
- Token pairs are issued by `verify-code`, `google-auth/verify-code`,
  `apple-auth/verify-code` (when configured), and `/auth/refresh`. All
  return the same envelope (see "Auth response envelope" below).
- Access-token validation is delegated to the
  `jwt-authentication-for-wp-rest-api` plugin (`/wp-json/jwt-auth/v1/*`).
  The custom theme hooks the plugin's `jwt_auth_expire` /
  `jwt_auth_token_before_sign` / `jwt_auth_token_before_dispatch` filters.
- The SPA stores the access token under
  `localStorage['pusher_coin_auth_token']` and a JSON bundle (user,
  refresh token, expiry, terms-accepted, nickname-required) under
  `localStorage['pusher_coin_user_data']`.

### Success envelope

```php
return new WP_REST_Response( $payload, $status );
```

`$payload` is a flat JSON object with snake_case keys. Use `200 OK` for
read / mutation success, `201 Created` only when a new entity is created
(currently just sign-up).

### Error envelope

```php
return new WP_Error( $code, $message, [ 'status' => $http_status ] );
```

WordPress renders this as:

```json
{
  "code": "machine_readable_code",
  "message": "Human-readable string",
  "data": { "status": 401 }
}
```

This is already consistent across all 5 existing endpoints — Phase 0
codifies it as the rule for every new endpoint.

**Status code conventions:**

- `400` — request shape invalid (`missing_required_fields`,
  `invalid_email`, `weak_password`).
- `401` — auth failed or credential rejected (`authentication_failed`,
  `invalid_verification_code`, `verification_code_expired`,
  `invalid_id_token`). The SPA's axios interceptor treats every 401 as
  token-expiry: it clears `pusher_coin_auth_token` /
  `pusher_coin_user_data` and dispatches the `auth:token-expired`
  `CustomEvent` (`frontend/src/services/api.js:48-56`). **New endpoints
  must not reuse 401 for non-auth failures** — use 403 for permission and
  409 for state conflicts.
- `403` — caller is authenticated but not permitted (`email_not_verified`,
  `terms_not_accepted`, `jwt_auth_bad_config`).
- `404` — entity does not exist (`no_verification_code`, `user_not_found`).
- `409` — state conflict (`email_exists`, `username_exists`).
- `5xx` — server-side failure (`user_creation_failed`, `email_send_failed`,
  `jwt_not_configured`, `token_verification_failed`).

### Pagination

For collection endpoints (transaction history is the first one in
Phase 4):

- Query: `?page=N&per_page=M`. `page` is 1-indexed, `per_page` defaults
  to 20, capped at 100.
- Response:

  ```json
  {
    "items": [ ... ],
    "total": 123,
    "page": 2,
    "per_page": 20
  }
  ```

- Filtering parameters are flat snake_case query params
  (`?type=topup&from=2026-01-01`).

### Permission callbacks

- Public routes: `permission_callback` returns `true` (sign-up,
  request-verification, verify-code, google-auth/*, apple-auth/*,
  auth/refresh). All public auth endpoints carry `Rate_Limiter` checks
  (5 / 15min per IP; 10 / 24h per IP for sign-up).
- Authenticated routes use `Permissions::require_logged_in` (auth/logout,
  user/accept-terms, user/set-nickname, plus future Phase 2+ endpoints).
- Play / top-up routes will use `Permissions::require_play_ready` —
  composes logged-in + terms-accepted + nickname-chosen.
- Admin routes (under `pc/v1/admin/...`): `permission_callback` calls
  `current_user_can( 'manage_options' )`.
- The custom `play` capability gates room/play endpoints in Phase 6.

### Naming

- Paths: kebab-case (`/google-auth/verify-code`).
- Request fields: snake_case (`id_token`, `verification_code`).
- Response fields: snake_case (`user_id`, `user_email`, `requires_verification`).
- Error codes: snake_case, scoped by failure mode
  (`missing_required_fields`, `email_not_verified`).

---

## Endpoints — current

Endpoints are registered across
`backend/wp-content/themes/pc/app/rest-api/UserController.php`,
`GoogleAuthController.php`, `AppleAuthController.php` (Apple is stubbed
behind `apple_not_configured` until Apple Developer enrollment), and
`AuthController.php` (logout / refresh and the shared token-pair helpers).
Bodies and responses are quoted faithfully.

### Auth response envelope

`verify-code`, `google-auth/verify-code`, `apple-auth/verify-code` (when
configured), and `auth/refresh` all return the same shape:

```json
{
  "access_token": "<jwt, ~15min TTL>",
  "access_token_expires_in": 900,
  "refresh_token": "<opaque, rotates on every refresh>",
  "refresh_token_expires_in": 604800,
  "user_id": 42,
  "user_email": "...",
  "user_nicename": "...",
  "user_display_name": "...",
  "terms_accepted": true,
  "nickname_required": false,
  "email_verified": true
}
```

`terms_accepted` is `true` only when the stored
`terms_accepted_version` matches the current `pc_terms_current_version`
WP option. `nickname_required` is `true` when `nickname_chosen` user
meta is unset (i.e. the social-login auto-generated `User-<n>` is still
in place). `email_verified` is `true` when `email_verified_at > 0` (set
by `/user/confirm-email`).

### `POST /pc/v1/user/sign-up/`

Create a new account. Public. Rate limit: 10 / 24h per IP.

Request:
```json
{ "email": "...", "nickname": "...", "phone": "...", "password": "...", "terms_accepted": true }
```
`phone` optional; the others required. `terms_accepted` must be `true`
or the request is rejected with `terms_not_accepted`.

Response (`201`):
```json
{ "id": 42, "email": "...", "nickname": "...", "phone": "..." }
```

On success the server stores `terms_accepted_at`,
`terms_accepted_version`, and `nickname_chosen=1` user meta.

Errors: `missing_required_fields` 400, `invalid_email` 400,
`weak_password` 400 (<6 chars), `terms_not_accepted` 403,
`email_exists` 409, `username_exists` 409, `rate_limited` 429,
`user_creation_failed` 500.

### `POST /pc/v1/user/request-verification/`

Step 1 of email/password 2FA: validate credentials, mail a 6-digit code
(15-minute TTL). Public. Rate limit: 5 / 15min per IP.

Request:
```json
{ "login": "...", "password": "..." }
```

Response (`200`):
```json
{ "success": true, "message": "Verification code has been sent to your email address." }
```

Errors: `missing_required_fields` 400, `authentication_failed` 401,
`rate_limited` 429, `email_send_failed` 500.

### `POST /pc/v1/user/verify-code/`

Step 2: redeem the code, return the auth envelope. Public.

Request:
```json
{ "login": "...", "password": "...", "code": "123456" }
```

Response (`200`): the canonical auth envelope (see top of this section).

Errors: `missing_required_fields` 400, `authentication_failed` 401,
`no_verification_code` 404, `verification_code_expired` 401,
`invalid_verification_code` 401, `jwt_not_configured` 500,
`jwt_library_missing` 500, `jwt_encoding_failed` 500.

### `POST /pc/v1/user/accept-terms`

Record acceptance of the current Terms & Conditions version.
Bearer auth.

Request:
```json
{ "version": "2026-05" }
```

Response (`200`):
```json
{ "terms_accepted_at": 1714905600, "terms_accepted_version": "2026-05" }
```

Errors: `rest_forbidden` 401, `invalid_terms_version` 400.

### `POST /pc/v1/user/set-nickname`

Pick a unique nickname (required after first social login). Bearer auth.

Request:
```json
{ "nickname": "..." }
```

Response (`200`):
```json
{ "nickname": "..." }
```

Server-side: 3–20 chars matching `^[A-Za-z0-9_]+$`, unique across
`wp_users.user_login` and `nickname` user meta. Sets `nickname_chosen=1`
on success.

Errors: `rest_forbidden` 401, `invalid_nickname` 400, `nickname_taken` 409.

### `GET /pc/v1/user/me`

Return the canonical user shape for the account page. Bearer auth.

Response (`200`):
```json
{
  "id": 42,
  "email": "...",
  "email_verified": true,
  "email_verified_at": 1714905600,
  "nickname": "...",
  "phone": "+1...",
  "phone_verified": false,
  "terms_accepted": true,
  "terms_accepted_version": "2026-05",
  "google_linked": true,
  "apple_linked": false,
  "balance_money": 0,
  "balance_coins": 0
}
```

`balance_money` and `balance_coins` are read from `wp_pc_wallets` (Phase
4, Step 1). Users with no wallet row yet get `"0.00"` / `0` —
`Wallet_Service` lazy-creates the row on first credit.

### `PATCH /pc/v1/user/me`

Update mutable profile fields. Bearer auth. Phase 2 ships only `phone`;
nickname mutations remain on `/user/set-nickname` for the uniqueness
check.

Request:
```json
{ "phone": "+1 555 010 1234" }
```

Empty string clears the phone meta. Response: same shape as `GET /user/me`.

Errors: `rest_forbidden` 401, `invalid_phone` 400.

### `POST /pc/v1/user/request-email-confirmation`

Mail a confirmation link to the user's email
(`{pc_spa_base_url}/confirm-email?token=...`) with a 24-hour TTL.
Bearer auth. Rate limit: 5 / 15min per user.

Response (`200`):
```json
{ "success": true, "message": "A confirmation link has been sent to your email address." }
```

Errors: `rest_forbidden` 401, `rate_limited` 429, `email_send_failed` 500.

### `POST /pc/v1/user/confirm-email`

Redeem the confirmation token from the emailed link. Public — the token
is the credential.

Request:
```json
{ "token": "..." }
```

Response (`200`):
```json
{ "success": true, "email_verified_at": 1714905600 }
```

Errors: `token_invalid` 401, `token_expired` 401.

### `POST /pc/v1/user/request-password-change`

Mail a 6-digit code to the user's email (15-minute TTL). Bearer auth.
Rate limit: 5 / 15min per user.

Response (`200`):
```json
{ "success": true, "message": "A 6-digit code has been sent to your email address." }
```

Errors: `rest_forbidden` 401, `rate_limited` 429, `email_send_failed` 500.

### `POST /pc/v1/user/confirm-password-change`

Validate the current password + 6-digit code, set the new password,
revoke every other active refresh token for the user, and return a
freshly-issued auth envelope. Bearer auth.

Request:
```json
{ "current_password": "...", "new_password": "...", "code": "######" }
```

Response (`200`): the canonical auth envelope.

Errors: `missing_required_fields` 400, `weak_password` 400,
`authentication_failed` 401 (wrong current password),
`no_verification_code` 404, `verification_code_expired` 401,
`invalid_verification_code` 401, `jwt_not_configured` 500,
`jwt_library_missing` 500, `jwt_encoding_failed` 500.

### `POST /pc/v1/google-auth/authentication`

Step 1 of Google OAuth: verify ID token, create/find user, mail a
6-digit code. Public. Rate limit: 5 / 15min per IP.

Request:
```json
{ "id_token": "<google jwt>" }
```

Response (`200`):
```json
{
  "requires_verification": true,
  "success": true,
  "message": "Verification code has been sent to your email address."
}
```

Errors: `missing_id_token` 400, `invalid_token_data` 400,
`email_not_verified` 403, `google_not_configured` 500,
`invalid_id_token` 401, `token_verification_failed` 401,
`rate_limited` 429, `user_creation_failed` 500, `email_send_failed` 500.

### `POST /pc/v1/google-auth/verify-code`

Step 2: redeem the code, return the auth envelope. Public.

Request:
```json
{ "id_token": "<google jwt>", "verification_code": "123456" }
```

Response (`200`): the canonical auth envelope (see top of this section).
First-time Google users get `nickname_required: true` and must POST
`/user/set-nickname` before any gated endpoint succeeds.

Errors: `missing_required_fields` 400, `invalid_token_data` 400,
`user_not_found` 404, `no_verification_code` 404,
`verification_code_expired` 401, `invalid_verification_code` 401,
`jwt_not_configured` 500, `jwt_library_missing` 500,
`jwt_encoding_failed` 500.

### `POST /pc/v1/apple-auth/authentication` (stub)

Mirror of `/google-auth/authentication`, gated on
`APPLE_CLIENT_ID` / `APPLE_TEAM_ID` / `APPLE_KEY_ID` / `APPLE_PRIVATE_KEY`
constants or matching options. Until those are set, every call returns
`apple_not_configured` (500). Same rate limit as Google.

When configured: request `{ id_token }`, response
`{ requires_verification, success, message }` (mirrors Google).

### `POST /pc/v1/apple-auth/verify-code` (stub)

Same configuration gate. When configured: request
`{ id_token, verification_code }`, response is the canonical auth envelope.

### `POST /pc/v1/auth/logout`

Revoke the supplied refresh token (and any descendants). Bearer auth.

Request:
```json
{ "refresh_token": "..." }
```

Response (`200`):
```json
{ "success": true }
```

`refresh_token` is optional — clients without one (e.g. the access token
already expired) can still call this to record a `logout` audit event.
Errors: `rest_forbidden` 401.

### `GET /pc/v1/rooms`

Paginated list of rooms. Public. Phase 3.

Query: `?page=N&per_page=M` (defaults 1 / 20; `per_page` capped at 100).

Response (`200`):
```json
{
  "items": [
    {
      "id": 42,
      "name": "Sunset Pusher",
      "status": "available",
      "theme_song_url": "https://...",
      "stream_url": "https://...",
      "current_window": { "start_at": "2026-05-13T18:00:00+00:00", "end_at": "2026-05-13T22:00:00+00:00" },
      "next_window":    { "start_at": "2026-05-14T18:00:00+00:00", "end_at": "2026-05-14T22:00:00+00:00" }
    }
  ],
  "total": 8,
  "page": 1,
  "per_page": 20
}
```

`status` is one of `available`, `maintenance`, `unavailable`. `stream_url`
is `null` unless `status === "available"`. `current_window` is `null`
when the room is not currently in a broadcast window. `next_window` is
`null` when no future window is scheduled (and is the *following* window
when `current_window` is set).

### `GET /pc/v1/rooms/{id}`

Single room. Public. Phase 3. Same item shape as `GET /pc/v1/rooms` (a
flat object, not wrapped in `items`).

Errors: `room_not_found` 404.

### `GET /pc/v1/rooms/{id}/schedule`

Weekly schedule rules for one room. Public. Phase 3.

Response (`200`):
```json
{
  "rules": [
    { "weekday": 0, "start_time": "18:00", "end_time": "22:00", "recurrence": "always", "once_date": null }
  ],
  "next_window": { "start_at": "2026-05-14T18:00:00+00:00", "end_at": "2026-05-14T22:00:00+00:00" }
}
```

`weekday` is 0=Mon..6=Sun (ISO). `recurrence` is `always` or `once`;
`once_date` is `null` unless `recurrence === "once"`.

Errors: `room_not_found` 404.

### `GET /pc/v1/wallet`

Bearer auth (`Permissions::require_logged_in`). Phase 4.

Response (`200`):
```json
{
  "balance_money": "0.00",
  "balance_coins": 0,
  "lots": [
    { "qty": 5, "unit_price": "40.00" },
    { "qty": 3, "unit_price": "45.00" }
  ]
}
```

`balance_money` is a decimal string (UAH) so JS doesn't introduce
floating-point rounding when totalling. `lots` are returned oldest
first — the same FIFO order they're consumed in.

A user with no wallet row yet returns the zero state. The row is
created lazily by `Wallet_Service::credit_lot` on the first top-up
settlement (Step 2).

Errors: `rest_forbidden` 401.

### `GET /pc/v1/admin/me`

Probe used by the admin SPA to verify the current session is both
authenticated and has `manage_options`. Bearer auth + admin gate.

Response (`200`):
```json
{
  "id": 1,
  "email": "admin@example.com",
  "display_name": "Ops Admin",
  "capabilities": { "manage_options": true }
}
```

Errors: `rest_forbidden` 401 (not logged in), `rest_forbidden` 403 (not admin).

### `GET /pc/v1/admin/rooms`

Paginated list including drafts. Bearer auth + admin gate. Same item
shape as the public `GET /pc/v1/rooms`, with two extra fields admins
need: `machine_id` and `post_status`.

Response (`200`): standard pagination envelope.

### `POST /pc/v1/admin/rooms`

Create a room. Bearer auth + admin gate.

Request:
```json
{
  "name": "Sunset Pusher",
  "status": "available",
  "stream_url": "https://...",
  "theme_song_url": "https://...",
  "machine_id": "sonoff_10024fb618"
}
```
`name` required; the rest are optional (status defaults to `unavailable`).

Response (`201`): the admin-room shape.

Errors: `invalid_room_name` 400, `invalid_room_status` 400,
`invalid_room_url` 400, `invalid_room_machine_id` 400,
`room_create_failed` 500.

### `GET /pc/v1/admin/rooms/{id}`

Single room, including drafts. Bearer auth + admin gate. Errors:
`room_not_found` 404.

### `PUT /pc/v1/admin/rooms/{id}`

Partial update — omitted fields are left untouched. Bearer auth + admin
gate.

Request: any subset of the create payload.

Response (`200`): the updated admin-room shape.

Errors: `room_not_found` 404, plus the create-time validation errors.

### `DELETE /pc/v1/admin/rooms/{id}`

Trashes (soft-deletes) the room. Bearer auth + admin gate. Reads stop
returning trashed rooms; restore via WP admin if needed.

Response (`200`): `{ "deleted": true, "id": 42 }`.

Errors: `room_not_found` 404.

### `PUT /pc/v1/admin/rooms/{id}/schedule`

Atomic replace of the room's schedule rules. The previous rule set is
deleted and the supplied set is inserted in a single transaction.
Bearer auth + admin gate.

Request:
```json
{
  "rules": [
    { "weekday": 0, "start_time": "18:00", "end_time": "22:00", "recurrence": "always" },
    { "weekday": 5, "start_time": "20:00", "end_time": "23:30", "recurrence": "once", "once_date": "2026-06-12" }
  ]
}
```
`weekday` 0=Mon..6=Sun. `start_time < end_time` (cross-midnight windows
must be split into two rules). `once_date` is required iff
`recurrence === "once"`.

Response (`200`):
```json
{
  "rules": [ ... ],
  "next_window": { "start_at": "...", "end_at": "..." }
}
```

Errors: `room_not_found` 404, `invalid_schedule_rule` 400,
`schedule_write_failed` 500.

### `POST /pc/v1/auth/refresh`

Rotate the refresh token, return a fresh auth envelope. Public (the
refresh token is the credential).

Request:
```json
{ "refresh_token": "..." }
```

Response (`200`): the canonical auth envelope. Each redeem rotates: the
presented token's row is marked `revoked_at` and `replaced_by`; a new
row is written. Reuse of an already-revoked token revokes the entire
descendant chain and returns `token_revoked`.

Errors: `missing_required_fields` 400, `token_invalid` 401,
`token_expired` 401, `token_revoked` 401, `user_not_found` 404,
`jwt_not_configured` 500, `jwt_library_missing` 500,
`jwt_encoding_failed` 500.

---

## Endpoints — planned (by phase)

Stub shapes only. These are not implemented; they are the contract Phase
1+ work will satisfy.

### Phase 3 — rooms & schedules

All Phase 3 endpoints (public read + admin CRUD + admin schedule
replace) ship in the current section. The admin SPA that consumes
the admin endpoints lands in Phase 3 too — see `ADMIN-DECISION.md`.

### Phase 4 — wallet & transactions

`GET /pc/v1/wallet` ships in the current section (Step 1). The rest of
the Phase 4 surface is still planned:

- `POST /pc/v1/wallet/topup` — Bearer. Request `{ coin_qty, unit_price }`.
  Response: `{ transaction_id, liqpay: { data, signature } }`. The SPA
  POSTs `{ data, signature }` to `https://www.liqpay.ua/api/3/checkout`.
- `POST /pc/v1/payments/liqpay/callback` — public, signature-validated,
  idempotent on `(order_id, status)`. On `success`: marks the
  transaction `completed`, creates a coin lot, increments the wallet.
- `POST /pc/v1/wallet/withdraw` — Bearer. Request `{ coin_qty }`.
  Debits FIFO into a pending transaction; admin approves out-of-band.
- `GET /pc/v1/transactions` — Bearer. Paginated. Filters: `type`
  (`topup`, `withdraw`), `from`, `to`. Item: `{ id, type, amount_money,
  amount_coins, unit_price, status, created_at }`. Top-ups and
  withdrawals only — game results stay out.
- `GET /pc/v1/admin/coin-pricing`, `PUT /pc/v1/admin/coin-pricing` —
  admin. Reads / writes `pc_coin_price_default`, `pc_coin_price_min`,
  `pc_coin_price_max` WP options.
- `GET /pc/v1/admin/withdrawals`,
  `POST /pc/v1/admin/withdrawals/{id}/approve|reject` — admin queue.

Errors planned: `insufficient_balance` 409, `coin_price_out_of_bounds`
400, `payment_failed` 502, `liqpay_signature_invalid` 401.

### Phase 5 — machine (admin)

All admin-gated. Body / response shapes mirror the Home Assistant
endpoints in `PUSHER-COIN-COMMANDS.txt`.

- `POST /pc/v1/admin/machine/power` — `{ on: bool }`.
- `GET /pc/v1/admin/machine/state` — `{ relay_closed, light_state,
  coin_count, last_bonus }`.
- `PUT /pc/v1/admin/machine/bonus-map` — `{ map: { "1": coins, ... }`.

Errors: `machine_offline` 503, `machine_call_failed` 502.

### Phase 6 — queue & play

- `GET /pc/v1/rooms/{id}/queue` — Bearer. Response: `{ queue: [{
  user_id, nickname, coins }], current_turn_user_id }`.
- `POST /pc/v1/rooms/{id}/queue/join` — Bearer. Request `{ coins }`.
- `POST /pc/v1/rooms/{id}/queue/leave` — Bearer.
- `POST /pc/v1/rooms/{id}/play` — Bearer. Request: `{ }` (single coin
  toss; the wallet decides which lot to deduct). Response: `{
  toss_id, coins_remaining }`.

A real-time channel (websocket / Pusher / Soketi — picked in Phase 5)
pushes `coin_dropped`, `bonus_won`, and `relay_closed` events. The HTTP
endpoints above are command-side only.

Errors: `not_player_turn` 403, `queue_locked` 409,
`relay_closed` 423, `insufficient_balance` 409.

### Phase 7 — support

- `GET /pc/v1/support/subjects` — public. Response: `{ items: [{ id, label }] }`.
- `POST /pc/v1/support/tickets` — public + captcha for guests; Bearer
  for logged-in users. Request: `{ email, subject_id, description,
  captcha_token? }`. Response: `{ ticket_id }`.
- `GET /pc/v1/admin/support/tickets` — admin. Paginated.
- `PATCH /pc/v1/admin/support/tickets/{id}` — admin. Status updates.
- `PUT /pc/v1/admin/support/subjects` — admin. Replaces the subjects list.

Errors: `captcha_failed` 401, `subject_not_found` 404.

---

## Error code registry

Every code emitted today plus the codes new endpoints will introduce.
One canonical code per failure mode — do not invent variants.

| Code | HTTP | Owning endpoint(s) |
| --- | --- | --- |
| `missing_required_fields` | 400 | sign-up, request-verification, verify-code, google-auth/verify-code, auth/refresh, confirm-password-change |
| `missing_id_token` | 400 | google-auth/authentication |
| `invalid_email` | 400 | sign-up |
| `invalid_token_data` | 400 | google-auth/* |
| `invalid_nickname` | 400 | user/set-nickname |
| `invalid_phone` | 400 | user/me PATCH |
| `invalid_room_name` | 400 | admin/rooms POST/PUT |
| `invalid_room_status` | 400 | admin/rooms POST/PUT |
| `invalid_room_url` | 400 | admin/rooms POST/PUT |
| `invalid_room_machine_id` | 400 | admin/rooms POST/PUT |
| `invalid_schedule_rule` | 400 | admin/rooms/{id}/schedule PUT |
| `invalid_terms_version` | 400 | user/accept-terms |
| `weak_password` | 400 | sign-up, confirm-password-change |
| `coin_price_out_of_bounds` | 400 | wallet/topup (planned) |
| `token_invalid` | 401 | auth/refresh, confirm-email |
| `authentication_failed` | 401 | request-verification, verify-code, confirm-password-change |
| `invalid_verification_code` | 401 | verify-code, google-auth/verify-code, confirm-password-change |
| `verification_code_expired` | 401 | verify-code, google-auth/verify-code, confirm-password-change |
| `invalid_id_token` | 401 | google-auth/* |
| `token_verification_failed` | 401 | google-auth/* |
| `token_expired` | 401 | auth/refresh, confirm-email |
| `token_revoked` | 401 | auth/refresh |
| `password_mismatch` | 401 | change-password (planned) |
| `apple_token_invalid` | 401 | apple-auth/* (when configured) |
| `rest_forbidden` | 401 | auth/logout, user/accept-terms, user/set-nickname, user/me, user/request-email-confirmation, user/request-password-change, user/confirm-password-change, admin/me, admin/rooms/* (when unauthenticated; 403 when authed but non-admin) |
| `captcha_failed` | 401 | support/tickets (planned, guest path) |
| `email_not_verified` | 403 | google-auth/authentication, play-ready gated endpoints (Permissions::require_play_ready) |
| `terms_not_accepted` | 403 | sign-up, play / top-up gated endpoints |
| `nickname_required` | 403 | gated play endpoints |
| `not_player_turn` | 403 | rooms/{id}/play (planned) |
| `jwt_auth_bad_config` | 403 | verify-code (legacy; replaced by `jwt_not_configured` in new endpoints) |
| `no_verification_code` | 404 | verify-code, google-auth/verify-code, confirm-password-change |
| `user_not_found` | 404 | google-auth/verify-code, auth/refresh |
| `room_not_found` | 404 | rooms/{id}, rooms/{id}/schedule |
| `subject_not_found` | 404 | support/tickets (planned) |
| `email_exists` | 409 | sign-up |
| `username_exists` | 409 | sign-up |
| `nickname_taken` | 409 | user/set-nickname, user/me PATCH (planned) |
| `insufficient_balance` | 409 | wallet, rooms/play (planned) |
| `queue_locked` | 409 | rooms/queue (planned) |
| `relay_closed` | 423 | rooms/play (planned) |
| `rate_limited` | 429 | sign-up, request-verification, google-auth/authentication, apple-auth/authentication, request-email-confirmation, request-password-change |
| `room_create_failed` | 500 | admin/rooms POST |
| `schedule_write_failed` | 500 | admin/rooms/{id}/schedule PUT |
| `user_creation_failed` | 500 | sign-up, google-auth/authentication |
| `email_send_failed` | 500 | request-verification, google-auth/authentication, request-email-confirmation, request-password-change |
| `google_not_configured` | 500 | google-auth/* |
| `apple_not_configured` | 500 | apple-auth/* |
| `jwt_not_configured` | 500 | verify-code, google-auth/verify-code, auth/refresh, confirm-password-change |
| `jwt_library_missing` | 500 | verify-code, google-auth/verify-code, auth/refresh, confirm-password-change |
| `jwt_encoding_failed` | 500 | verify-code, google-auth/verify-code, auth/refresh, confirm-password-change |
| `payment_failed` | 502 | wallet/topup (planned) |
| `machine_call_failed` | 502 | admin/machine/* (planned) |
| `machine_offline` | 503 | admin/machine/* (planned) |
