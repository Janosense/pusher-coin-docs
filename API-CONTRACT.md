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
- Tokens are issued by either `POST /pc/v1/user/verify-code/` or
  `POST /pc/v1/google-auth/verify-code`. Both use HS256, signed with
  `JWT_AUTH_SECRET_KEY` (server config), 7-day TTL.
- Token validation is delegated to the
  `jwt-authentication-for-wp-rest-api` plugin (`/wp-json/jwt-auth/v1/*`).
  The custom theme hooks the plugin's `jwt_auth_expire` /
  `jwt_auth_token_before_sign` / `jwt_auth_token_before_dispatch` filters.
- The SPA stores the token under `localStorage['pusher_coin_auth_token']`
  and the user shape under `localStorage['pusher_coin_user_data']`.

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
  request-verification, verify-code, google-auth/*). Phase 1 adds
  rate-limiting / captcha here.
- Authenticated routes: `permission_callback` calls
  `is_user_logged_in()` (the JWT plugin populates this when a valid
  bearer token is present).
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

All five endpoints are registered in
`backend/wp-content/themes/pc/app/rest-api/UserController.php` and
`GoogleAuthController.php`. Bodies and responses are quoted faithfully.

### `POST /pc/v1/user/sign-up/`

Create a new account. Public.

Request:
```json
{ "email": "...", "nickname": "...", "phone": "...", "password": "..." }
```
`phone` optional; the others required.

Response (`201`):
```json
{ "id": 42, "email": "...", "nickname": "...", "phone": "..." }
```

Errors: `missing_required_fields` 400, `invalid_email` 400,
`email_exists` 409, `username_exists` 409, `weak_password` 400 (<6 chars),
`user_creation_failed` 500.

### `POST /pc/v1/user/request-verification/`

Step 1 of email/password 2FA: validate credentials, mail a 6-digit code
(15-minute TTL). Public.

Request:
```json
{ "login": "...", "password": "..." }
```

Response (`200`):
```json
{ "success": true, "message": "Verification code has been sent to your email address." }
```

Errors: `missing_required_fields` 400, `authentication_failed` 401,
`email_send_failed` 500.

### `POST /pc/v1/user/verify-code/`

Step 2: redeem the code, return a JWT. Public.

Request:
```json
{ "login": "...", "password": "...", "code": "123456" }
```

Response (`200`):
```json
{
  "token": "<jwt>",
  "user_email": "...",
  "user_nicename": "...",
  "user_display_name": "..."
}
```

Errors: `missing_required_fields` 400, `authentication_failed` 401,
`no_verification_code` 404, `verification_code_expired` 401,
`invalid_verification_code` 401, `jwt_auth_bad_config` 403.

### `POST /pc/v1/google-auth/authentication`

Step 1 of Google OAuth: verify ID token, create/find user, mail a
6-digit code. Public.

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
`user_creation_failed` 500, `email_send_failed` 500.

### `POST /pc/v1/google-auth/verify-code`

Step 2: redeem the code, return a JWT. Public.

Request:
```json
{ "id_token": "<google jwt>", "verification_code": "123456" }
```

Response (`200`):
```json
{
  "token": "<jwt>",
  "user_id": 42,
  "user_email": "...",
  "user_nicename": "...",
  "user_display_name": "..."
}
```

Errors: `missing_required_fields` 400, `invalid_token_data` 400,
`user_not_found` 404, `no_verification_code` 404,
`verification_code_expired` 401, `invalid_verification_code` 401,
`jwt_not_configured` 500, `jwt_library_missing` 500,
`jwt_encoding_failed` 500.

---

## Endpoints — planned (by phase)

Stub shapes only. These are not implemented; they are the contract Phase
1+ work will satisfy.

### Phase 1 — auth hardening

- `POST /pc/v1/apple-auth/authentication` — request `{ id_token }`,
  response `{ requires_verification, success, message }` (mirrors Google).
- `POST /pc/v1/apple-auth/verify-code` — request
  `{ id_token, verification_code }`, response token bundle.
- `POST /pc/v1/auth/logout` — Bearer auth required. Response
  `{ success: true }`. Server invalidates the token (blacklist or
  refresh-token rotation).
- `POST /pc/v1/auth/refresh` — Bearer auth required. Response
  `{ token, token_expires }`.
- `POST /pc/v1/user/accept-terms` — Bearer auth required. Response
  `{ terms_accepted_at }`. Required before any play / top-up endpoint.

Errors introduced: `terms_not_accepted` 403, `token_revoked` 401,
`apple_not_configured` 500, `apple_token_invalid` 401.

### Phase 2 — account & verification gates

- `GET /pc/v1/user/me` — Bearer. Response: `{ id, email, email_verified,
  nickname, phone, terms_accepted_at, google_2fa_enabled,
  balance: { coins, money } }`.
- `PATCH /pc/v1/user/me` — Bearer. Editable fields: `nickname` (unique).
- `POST /pc/v1/user/request-email-confirmation` — Bearer. Response
  `{ success, message }`. 24-hour token TTL.
- `POST /pc/v1/user/confirm-email` — request `{ token }`. Response
  `{ email_verified: true }`.
- `POST /pc/v1/user/change-password` — request
  `{ current_password, new_password, verification_code }`. Bearer.

Errors introduced: `email_not_verified` 403, `nickname_taken` 409,
`token_invalid` 400, `token_expired` 401, `password_mismatch` 401.

### Phase 3 — rooms & schedules

- `GET /pc/v1/rooms` — public. Paginated. Item: `{ id, name, status,
  theme_song_url, stream_url, current_window, next_window }`.
- `GET /pc/v1/rooms/{id}` — public.
- `GET /pc/v1/rooms/{id}/schedule` — public. Response: `{ rules: [{
  weekday, start_time, end_time, recurrence }], next_window }`.
- `PUT /pc/v1/admin/rooms/{id}/schedule` — admin. Replaces the rules set.

### Phase 4 — wallet & transactions

- `GET /pc/v1/wallet` — Bearer. Response: `{ balance_money,
  balance_coins, lots: [{ qty, unit_price }] }`.
- `POST /pc/v1/wallet/topup` — Bearer. Request `{ amount, unit_price,
  payment_method_id }`. Response: `{ transaction_id, status }`.
- `POST /pc/v1/wallet/withdraw` — Bearer. Request `{ amount,
  destination }`. Response: `{ transaction_id, status }`.
- `GET /pc/v1/transactions` — Bearer. Paginated. Filters: `type`
  (`topup`, `withdraw`), `from`, `to`. Item: `{ id, type, amount,
  unit_price, status, created_at }`. Top-ups and withdrawals only —
  game results stay out.

Errors introduced: `insufficient_balance` 409, `payment_failed` 502,
`coin_price_out_of_bounds` 400.

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
| `missing_required_fields` | 400 | sign-up, request-verification, verify-code, google-auth/verify-code, planned auth/* |
| `missing_id_token` | 400 | google-auth/authentication |
| `invalid_email` | 400 | sign-up |
| `invalid_token_data` | 400 | google-auth/* |
| `weak_password` | 400 | sign-up, change-password (planned) |
| `coin_price_out_of_bounds` | 400 | wallet/topup (planned) |
| `token_invalid` | 400 | confirm-email (planned) |
| `authentication_failed` | 401 | request-verification, verify-code |
| `invalid_verification_code` | 401 | verify-code, google-auth/verify-code |
| `verification_code_expired` | 401 | verify-code, google-auth/verify-code |
| `invalid_id_token` | 401 | google-auth/* |
| `token_verification_failed` | 401 | google-auth/* |
| `token_expired` | 401 | confirm-email (planned) |
| `token_revoked` | 401 | auth/refresh (planned) |
| `password_mismatch` | 401 | change-password (planned) |
| `apple_token_invalid` | 401 | apple-auth/* (planned) |
| `captcha_failed` | 401 | support/tickets (planned, guest path) |
| `email_not_verified` | 403 | google-auth/authentication, gated endpoints (planned) |
| `terms_not_accepted` | 403 | gated endpoints (planned) |
| `not_player_turn` | 403 | rooms/{id}/play (planned) |
| `jwt_auth_bad_config` | 403 | verify-code |
| `no_verification_code` | 404 | verify-code, google-auth/verify-code |
| `user_not_found` | 404 | google-auth/verify-code |
| `subject_not_found` | 404 | support/tickets (planned) |
| `email_exists` | 409 | sign-up |
| `username_exists` | 409 | sign-up |
| `nickname_taken` | 409 | user/me PATCH (planned) |
| `insufficient_balance` | 409 | wallet, rooms/play (planned) |
| `queue_locked` | 409 | rooms/queue (planned) |
| `relay_closed` | 423 | rooms/play (planned) |
| `user_creation_failed` | 500 | sign-up, google-auth/authentication |
| `email_send_failed` | 500 | request-verification, google-auth/authentication |
| `google_not_configured` | 500 | google-auth/* |
| `apple_not_configured` | 500 | apple-auth/* (planned) |
| `jwt_not_configured` | 500 | google-auth/verify-code, planned |
| `jwt_library_missing` | 500 | google-auth/verify-code |
| `jwt_encoding_failed` | 500 | google-auth/verify-code |
| `payment_failed` | 502 | wallet/topup (planned) |
| `machine_call_failed` | 502 | admin/machine/* (planned) |
| `machine_offline` | 503 | admin/machine/* (planned) |
