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

### `GET /pc/v1/rooms/{id}/queue`

Bearer + play-ready. The room's queue, and the turn it confers.

**Doubles as the heartbeat.** The backend drops entries whose player
stopped calling this for `pc_queue_idle_timeout_seconds` (default 60),
so a client that wants to hold its place must keep polling — the SPA
does, every 3s. Phase 5 Step 7's push channel will replace the poll;
the endpoint stays as the state-of-truth read.

Response (`200`):
```json
{
  "queue": [ { "user_id": 4, "nickname": "jano3", "coins": 2 } ],
  "current_turn_user_id": 4,
  "online_count": 2,
  "session": {
    "id": 1, "user_id": 4, "room_id": 11,
    "started_at": "2026-07-28 20:45:00", "ended_at": null,
    "coins_played": 1, "coins_won": 0, "money_won": "0.00"
  },
  "idle_timeout_seconds": 60
}
```

`queue` is FIFO — index 0 holds the turn. `coins` is what that player
has *left* to play, not what they declared. `online_count` counts queued
players, not everyone watching the room. `session` is the head's open
bet session, or `null` for an empty queue.

Errors: `room_not_found` 404, `room_unavailable` 409.

### `POST /pc/v1/rooms/{id}/queue/join`

Bearer + play-ready. Join, or re-declare if already queued.

Request:
```json
{ "coins": 5 }
```

`coins` is an intent, not a reservation — it is capped at the wallet
balance at join time, and coins are debited one at a time by `play`.
Re-declaring keeps the original position: changing your mind about the
count must not let you jump your own place in the queue.

Response (`200`): the queue envelope, as above.

Errors: `invalid_coin_qty` 400, `insufficient_balance` 409,
`room_not_found` 404, `room_unavailable` 409.

### `POST /pc/v1/rooms/{id}/queue/leave`

Bearer + play-ready. Leave the queue; closes the bet session if the
caller held the turn.

Response (`200`): the queue envelope.

### `POST /pc/v1/rooms/{id}/play`

Bearer + play-ready. Toss exactly one coin. Empty request body.

The order of operations is the contract:

1. Refuse unless the caller holds the turn (`not_player_turn` 403).
2. Refuse while `sensor.relay_on` reads closed (`relay_closed` 423) —
   the machine is mid-payout.
3. Debit one coin FIFO (`insufficient_balance` 409).
4. Call the machine. **Only HTTP 200 counts as a toss.**
5. On any machine failure, re-credit the exact lot price consumed and
   return the machine's error. A player is never charged for a toss that
   did not happen.

Machine failures map to gateway statuses (502 / 503), never 401 — a 401
would trip the SPA's token-refresh interceptor and log the player out
mid-turn.

Response (`200`):
```json
{
  "toss_id": 9,
  "coins_remaining": 1,
  "balance_coins": 2,
  "queue": { "...": "the queue envelope" }
}
```

`toss_id` is the `wp_pc_machine_events` row for the toss.

Errors: `not_player_turn` 403, `relay_closed` 423,
`insufficient_balance` 409, `room_not_found` 404, `room_unavailable`
409, `machine_offline` 503, `machine_call_failed` 502,
`machine_unauthorized` 502, `machine_not_configured` 500.

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
  ],
  "coin_pricing": {
    "default": "40.00",
    "min": "10.00",
    "max": "500.00"
  }
}
```

`balance_money` is a decimal string (UAH) so JS doesn't introduce
floating-point rounding when totalling. `lots` are returned oldest
first — the same FIFO order they're consumed in. `coin_pricing`
mirrors the operator-tunable WP options so the top-up form can clamp
its slider client-side; the server enforces the same bounds on
`POST /wallet/topup`.

A user with no wallet row yet returns the zero state. The row is
created lazily on the first top-up settlement.

Errors: `rest_forbidden` 401.

### `POST /pc/v1/wallet/topup`

Begin a top-up. Bearer auth + `Permissions::require_play_ready`. Phase
4.

Request:
```json
{ "coin_qty": 5, "unit_price": "40.00" }
```
`coin_qty` is a positive integer; `unit_price` is a decimal string
within `[pc_coin_price_min, pc_coin_price_max]` (UAH). The amount sent
to LiqPay is `coin_qty * unit_price` (UAH).

Response (`200`):
```json
{
  "transaction_id": 17,
  "order_id": "pc-topup-17",
  "amount": "200.00",
  "checkout_url": "https://www.liqpay.ua/api/3/checkout",
  "liqpay": {
    "data": "<base64 params>",
    "signature": "<base64 sha1>"
  }
}
```

The SPA POSTs `liqpay.data` + `liqpay.signature` as form fields to
`checkout_url` (`<form method="POST" action="…">` works in any
browser — LiqPay's hosted page renders next).

A pending row is written to `wp_pc_transactions` immediately so the
LiqPay callback has something to look up via `external_ref = order_id`.
Settlement (status → `completed`, lot creation, wallet credit) happens
**only** from the callback, never from the redirect back to the SPA.

Errors: `rest_forbidden` 401 (not authed); `email_not_verified` /
`terms_not_accepted` / `nickname_required` 403; `invalid_coin_qty` 400;
`coin_price_out_of_bounds` 400; `liqpay_not_configured` 500.

### `POST /pc/v1/wallet/withdraw`

Request a coin withdrawal. Bearer auth + `Permissions::require_play_ready`.
Phase 4.

Request:
```json
{ "coin_qty": 5 }
```

The server FIFO-consumes `coin_qty` from the player's lots and parks a
`pending` withdrawal transaction with the consumed `(qty, unit_price)`
slices stored on `consumed_lots` (so a later rejection can re-credit
at the original prices). The actual payout is out-of-band — admins
approve in the admin SPA after paying the player by bank transfer.

A player can only have one pending withdrawal at a time; submitting a
second returns `withdrawal_already_pending` 409.

Response (`200`):
```json
{
  "transaction_id": 18,
  "status": "pending",
  "amount_coins": 5,
  "amount_money": "200.00"
}
```

Errors: `rest_forbidden` 401; `email_not_verified` / `terms_not_accepted`
/ `nickname_required` 403; `invalid_coin_qty` 400; `insufficient_balance`
409; `withdrawal_already_pending` 409.

### `GET /pc/v1/transactions`

Player transaction history. Bearer auth (`require_logged_in`).
Scoped to the current user. Phase 4.

Query:
- `?page=N&per_page=M` (default 1 / 20; per_page capped at 100).
- `?type=topup|withdraw` — optional, filters by type.
- `?from=YYYY-MM-DD` / `?to=YYYY-MM-DD` — optional, inclusive of both
  endpoints (`from` snapped to 00:00:00, `to` snapped to 23:59:59 in
  the site timezone).

Response (`200`):
```json
{
  "items": [
    {
      "id": 17,
      "type": "topup",
      "amount_money": "200.00",
      "amount_coins": 5,
      "unit_price": "40.00",
      "status": "completed",
      "created_at": "2026-05-13 18:00:00",
      "settled_at": "2026-05-13 18:00:30"
    }
  ],
  "total": 1,
  "page": 1,
  "per_page": 20
}
```

Top-ups and withdrawals only. Game results live on `wp_pc_bet_sessions`
(Phase 6) and are deliberately excluded from this view. `external_ref`
(LiqPay order_id) and `consumed_lots` are admin-only audit data and
are not exposed here.

Errors: `rest_forbidden` 401; `invalid_transaction_type` 400;
`invalid_date` 400.

### `POST /pc/v1/payments/liqpay/callback`

LiqPay webhook. Public route; the signed payload is the credential.
Phase 4.

Request (form-encoded by LiqPay):
```
data=<base64 params>
signature=<base64 sha1>
```

Always returns 200 on a valid signature even for unrecoverable
conditions (unknown `order_id`, already-settled txn). LiqPay treats
non-2xx as a delivery failure and retries indefinitely, so the
controller swallows recoverable surprises and writes them to
`wp_pc_auth_audit_log` instead. The body's `note` field disambiguates:
`already_settled`, `unknown_order`, or absent on first-time success.

State machine:
- LiqPay `success` / `sandbox` → `Wallet_Service::settle_topup` (atomic
  transaction-status flip + lot insert + wallet credit).
- LiqPay `failure` / `error` / `reversed` → mark transaction `failed`.
- Anything else (`processing`, `wait_secure`, `wait_accept`, …) →
  leave `pending`, wait for the next callback.
- Re-delivery of the same final status is a no-op (idempotent via the
  `pending`-status check).

Errors: `missing_required_fields` 400 (no `data`/`signature` in body);
`liqpay_signature_invalid` 401; `liqpay_payload_invalid` 400;
`liqpay_not_configured` 500.

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

### `GET /pc/v1/admin/withdrawals`

Admin queue. Bearer + admin gate. Phase 4.

Query: `?status=pending|completed|refunded|failed|all` (default
`pending`), `?page=N&per_page=M` (default 1 / 50).

Response (`200`):
```json
{
  "items": [
    {
      "id": 18,
      "user_id": 42,
      "user_email": "player@example.com",
      "user_nickname": "Coin Tosser",
      "amount_money": "200.00",
      "amount_coins": 5,
      "status": "pending",
      "notes": null,
      "created_at": "2026-05-14 18:00:00",
      "settled_at": null,
      "consumed_lots": [ { "qty": 5, "unit_price": "40.00" } ]
    }
  ],
  "total": 1,
  "page": 1,
  "per_page": 50
}
```

### `POST /pc/v1/admin/withdrawals/{id}/approve`

Mark a pending withdrawal `completed`. Bearer + admin gate. The actual
money payout happens out-of-band (bank transfer, etc.) — this endpoint
only records the decision and writes a `withdrawal_approved` audit log
entry.

Request:
```json
{ "notes": "Paid via Privat24 #ABC123" }
```
`notes` is optional, max 2000 characters.

Response (`200`): the updated withdrawal row (same shape as the list
item above).

Errors: `withdrawal_not_found` 404; `withdrawal_not_pending` 409.

### `POST /pc/v1/admin/withdrawals/{id}/reject`

Mark a pending withdrawal `refunded` and re-credit the originally-
consumed coin lots back to the player's wallet at their original unit
prices. Atomic. Bearer + admin gate.

Request:
```json
{ "notes": "KYC pending — please contact support." }
```

Response (`200`): the updated withdrawal row, now `status: refunded`
with `settled_at` populated.

Errors: `withdrawal_not_found` 404; `withdrawal_not_pending` 409;
`wallet_write_failed` 500.

### `GET /pc/v1/admin/coin-pricing`

Read the operator-tunable per-coin price bounds. Bearer + admin gate.
Phase 4.

Response (`200`):
```json
{ "default": "40.00", "min": "10.00", "max": "500.00" }
```

### `PUT /pc/v1/admin/coin-pricing`

Update the bounds. Bearer + admin gate. Phase 4.

Request:
```json
{ "default": "45.00", "min": "10.00", "max": "500.00" }
```

All three fields required. Validation:
- Each value must be a positive decimal (`> 0.00`, two decimals).
- `min <= max`.
- `min <= default <= max`.

Response (`200`): same shape as GET.

Errors: `invalid_coin_price` 400; `coin_price_bounds_invalid` 400.

### `GET /pc/v1/admin/machine/state`

Batched read of the physical machine's state. Bearer + admin gate.
Phase 5.

Response (`200`):
```json
{
  "online": true,
  "power_on": true,
  "coin_count": 1287,
  "last_bonus_number": 7,
  "light_state": 5,
  "relay_closed": false
}
```

`online` is a cheap connectivity probe against the HA root. The other
fields **soft-fail per sensor**: if one sensor is `"unavailable"` its
field comes back `null` instead of failing the whole call. This is
deliberate — the admin UI wants partial state; the toss flow (Phase 6)
will use individual sensor methods that hard-fail.

Errors: `machine_not_configured` 500.

### `POST /pc/v1/admin/machine/power`

Switch the machine on or off. Bearer + admin gate. Phase 5.

Request:
```json
{ "on": true }
```

Response (`200`):
```json
{ "ok": true, "on": true }
```

Writes a `machine_power_changed` audit log entry. Errors map HA
failures to gateway statuses (the **caller's** auth is fine; failures
mean the upstream HA is the problem):

- `machine_offline` 503
- `machine_unavailable_state` 503
- `machine_unauthorized` 502 (HA rejected our bearer token)
- `machine_call_failed` 502
- `machine_not_configured` 500
- `missing_required_fields` 400 (`on` not boolean)

### `GET /pc/v1/admin/machine/bonus-map`

Read the operator-configured coin payouts for each bonus number and
relay closure. Bearer + admin gate. Phase 5.

Response (`200`):
```json
{
  "map": {
    "1": 0, "2": 0, "3": 1, "4": 0, "5": 2, "6": 0,
    "7": 5, "8": 0, "9": 0, "10": 10, "11": 0, "12": 50
  },
  "relay_coin_count": 10
}
```

The map always has exactly 12 string-keyed entries (`"1"`..`"12"`).
Missing entries on a fresh install default to `0`.

### `PUT /pc/v1/admin/machine/bonus-map`

Replace the bonus payout configuration. Bearer + admin gate. Phase 5.

Request: same shape as GET. All 12 entries required (a partial map
returns `invalid_bonus_map`). Values are non-negative integers
(strings or ints accepted; coerced to int server-side).

Response (`200`): same shape as GET.

Writes a `machine_bonus_map_updated` audit log entry.

Errors: `invalid_bonus_map` 400.

### `GET /pc/v1/support/subjects`

Public. The support form's dropdown options plus the captcha challenge
the guest path will demand — both in one response so the SPA can render
the whole form after a single request.

Response (`200`):
```json
{
  "items": [ { "id": 7, "label": "Payment problem" } ],
  "captcha": { "provider": "turnstile", "site_key": "0x4AAA..." }
}
```

Only published subjects are listed, in `menu_order`. `captcha` is
`null` when the operator has not configured a provider — in that mode
`POST /support/tickets` accepts guests without a token. See
`Captcha_Verifier`.

### `POST /pc/v1/support/tickets`

Public route, two paths. Rate-limited to 5 per hour per IP
(`rate_limited` 429).

Request:
```json
{
  "email": "player@example.com",
  "subject_id": 7,
  "description": "My top-up did not arrive.",
  "captcha_token": "..."
}
```

- **Guest** (no Bearer) — `email` is required and `captcha_token` is
  required whenever `captcha` is non-null on the subjects response. The
  ticket records `email_verified: false`.
- **Logged in** (Bearer) — `email` and `captcha_token` are ignored. The
  account email is used, so a ticket cannot be filed under someone
  else's address, and the account must be email-verified.

`description` is bounded to 10–5000 characters.

Response (`201`):
```json
{ "ticket_id": 42 }
```

Writes a `support_ticket_created` audit log entry and mails
`pc_support_email` with the submitter as `Reply-To`.

Errors: `invalid_email` 400, `invalid_description` 400,
`subject_not_found` 404, `captcha_failed` 401, `email_not_verified`
403, `rate_limited` 429, `ticket_write_failed` 500.

### `GET /pc/v1/admin/support/tickets`

Admin. Paginated (`page`, `per_page`), filterable by `status` and by a
free-text `search` matched against email + description.

Response (`200`):
```json
{
  "items": [ {
    "id": 42,
    "user_id": null,
    "email": "player@example.com",
    "subject_id": 7,
    "subject_label": "Payment problem",
    "description": "My top-up did not arrive.",
    "email_verified": false,
    "status": "open",
    "ip": "203.0.113.9",
    "user_agent": "Mozilla/5.0 …",
    "created_at": "2026-07-24 19:46:18",
    "updated_at": "2026-07-24 19:46:18"
  } ],
  "total": 1,
  "page": 1,
  "per_page": 20
}
```

`subject_label` resolves even for a retired subject, so old tickets stay
readable after the list is edited.

Errors: `invalid_ticket_status` 400.

### `PATCH /pc/v1/admin/support/tickets/{id}`

Admin. Status transitions only — replies happen from the operator's mail
client, since the notification mail sets `Reply-To` to the player.

Request:
```json
{ "status": "in_progress" }
```

`status` ∈ `open` | `in_progress` | `resolved` | `closed`.

Response (`200`): the updated ticket, same shape as the list rows.
Writes a `support_ticket_updated` audit log entry.

Errors: `invalid_ticket_status` 400, `ticket_not_found` 404.

### `GET /pc/v1/admin/support/captcha`

Admin. Captcha configuration for the guest ticket path.

Response (`200`):
```json
{
  "provider": "turnstile",
  "site_key": "0x4AAA...",
  "secret_configured": true,
  "enabled": true
}
```

`secret_configured` reports only whether the `PC_CAPTCHA_SECRET`
constant exists in `wp-config.php` — the secret itself is never returned.
`enabled` is `site_key && secret_configured`; when false the guest path
accepts tickets with no challenge. See `CAPTCHA_SETUP.md`.

### `PUT /pc/v1/admin/support/captcha`

Admin. Sets the public half. The secret is a wp-config edit, by design.

Request:
```json
{ "provider": "turnstile", "site_key": "0x4AAA..." }
```

`provider` ∈ `turnstile` | `hcaptcha`.

Response (`200`): same shape as GET.
Writes a `support_captcha_updated` audit log entry.

Errors: `invalid_captcha_config` 400.

### `GET /pc/v1/admin/support/subjects`

Admin. Like the public list but includes hidden (draft) subjects and
carries `hidden` + `order`.

Response (`200`):
```json
{ "items": [ { "id": 7, "label": "Payment problem", "hidden": false, "order": 0 } ] }
```

### `PUT /pc/v1/admin/support/subjects`

Admin. Replaces the whole list in one call.

Request:
```json
{ "items": [ { "id": 7, "label": "Payment problem", "hidden": false }, { "label": "Something else" } ] }
```

Array position becomes `menu_order`. Items with an `id` are updated in
place so existing tickets keep pointing at a live subject; items without
one are created; anything absent from the payload is **trashed, not
deleted**, so an old ticket still resolves its label.

Response (`200`): the resulting list, same shape as GET.
Writes a `support_subjects_updated` audit log entry.

Errors: `invalid_subject` 400.

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

All Phase 4 endpoints ship in the current section.

### Phase 5 — machine (admin)

`GET /pc/v1/admin/machine/state`, `POST /pc/v1/admin/machine/power`,
`GET /pc/v1/admin/machine/bonus-map`, and
`PUT /pc/v1/admin/machine/bonus-map` ship in the current section. Still
planned:

- `POST /pc/v1/machine/webhook` — HA outbound webhook ingress
  (HMAC-signed payload). One of the two candidate transports for Step 7;
  the other is a backend poller and neither is picked until the Step 6
  walk-through establishes whether HA can push at all. Whichever wins
  calls `Machine_Ingest_Service`, which already ships — the endpoint is
  a thin signature-check + `event_key` wrapper over it.
- `POST /pc/v1/realtime/auth` — private-channel subscription auth for
  the Step 7 push channel (provider unpicked: Pusher / Ably / Soketi).

### Phase 6 — queue & play

All four queue/play endpoints ship in the current section. Still
planned: the real-time channel (Phase 5 Step 7) that pushes
`coin_dropped`, `bonus_won`, and `relay_closed` instead of the SPA's 3s
poll of `GET /rooms/{id}/queue`. `queue_locked` 409 is reserved for it —
nothing returns that code today, because with a persisted queue there is
no lock to contend.

Chat has no contract yet: the in-room `RoomChat.vue` still renders local
placeholder messages. It needs storage, moderation rules, and the same
transport decision, so it is deliberately out of the Phase 6 slice.

### Phase 7 — support

All Phase 7 support endpoints ship in the current section. Still
planned, from ROADMAP §7.4 (ops alerts):

- an alerting path for machine-offline / stalled-coin-sensor / withdrawal
  spikes. No endpoint shape yet — it needs the Phase 5 transport
  decision first, since "machine offline" is only observable once
  events (or polls) arrive.

---

## Error code registry

Every code emitted today plus the codes new endpoints will introduce.
One canonical code per failure mode — do not invent variants.

| Code | HTTP | Owning endpoint(s) |
| --- | --- | --- |
| `missing_required_fields` | 400 | sign-up, request-verification, verify-code, google-auth/verify-code, auth/refresh, confirm-password-change |
| `missing_id_token` | 400 | google-auth/authentication |
| `invalid_email` | 400 | sign-up, support/tickets |
| `invalid_description` | 400 | support/tickets |
| `invalid_subject` | 400 | admin/support/subjects PUT |
| `invalid_ticket_status` | 400 | admin/support/tickets (GET filter, PATCH) |
| `invalid_captcha_config` | 400 | admin/support/captcha PUT |
| `invalid_coin_qty` | 400 | rooms/{id}/queue/join |
| `invalid_token_data` | 400 | google-auth/* |
| `invalid_nickname` | 400 | user/set-nickname |
| `invalid_phone` | 400 | user/me PATCH |
| `invalid_room_name` | 400 | admin/rooms POST/PUT |
| `invalid_room_status` | 400 | admin/rooms POST/PUT |
| `invalid_room_url` | 400 | admin/rooms POST/PUT |
| `invalid_room_machine_id` | 400 | admin/rooms POST/PUT |
| `invalid_schedule_rule` | 400 | admin/rooms/{id}/schedule PUT |
| `invalid_terms_version` | 400 | user/accept-terms |
| `invalid_transaction_type` | 400 | transactions |
| `invalid_date` | 400 | transactions |
| `invalid_bonus_map` | 400 | admin/machine/bonus-map PUT |
| `weak_password` | 400 | sign-up, confirm-password-change |
| `invalid_coin_qty` | 400 | wallet/topup |
| `invalid_coin_price` | 400 | admin/coin-pricing PUT |
| `coin_price_out_of_bounds` | 400 | wallet/topup |
| `coin_price_bounds_invalid` | 400 | admin/coin-pricing PUT |
| `liqpay_payload_invalid` | 400 | payments/liqpay/callback |
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
| `captcha_failed` | 401 | support/tickets (guest path, when a provider is configured) |
| `liqpay_signature_invalid` | 401 | payments/liqpay/callback |
| `email_not_verified` | 403 | google-auth/authentication, support/tickets (logged-in path), play-ready gated endpoints (Permissions::require_play_ready) |
| `terms_not_accepted` | 403 | sign-up, play / top-up gated endpoints |
| `nickname_required` | 403 | gated play endpoints |
| `not_player_turn` | 403 | rooms/{id}/play |
| `jwt_auth_bad_config` | 403 | verify-code (legacy; replaced by `jwt_not_configured` in new endpoints) |
| `no_verification_code` | 404 | verify-code, google-auth/verify-code, confirm-password-change |
| `user_not_found` | 404 | google-auth/verify-code, auth/refresh |
| `room_not_found` | 404 | rooms/{id}, rooms/{id}/schedule |
| `withdrawal_not_found` | 404 | admin/withdrawals/{id}/approve, /reject |
| `subject_not_found` | 404 | support/tickets |
| `ticket_not_found` | 404 | admin/support/tickets PATCH |
| `room_not_found` | 404 | rooms/{id}/queue*, rooms/{id}/play |
| `email_exists` | 409 | sign-up |
| `username_exists` | 409 | sign-up |
| `nickname_taken` | 409 | user/set-nickname, user/me PATCH (planned) |
| `insufficient_balance` | 409 | wallet/withdraw |
| `room_unavailable` | 409 | rooms/{id}/queue*, rooms/{id}/play |
| `withdrawal_already_pending` | 409 | wallet/withdraw |
| `withdrawal_not_pending` | 409 | admin/withdrawals/{id}/approve, /reject |
| `insufficient_balance` | 409 | wallet, rooms/{id}/play, rooms/{id}/queue/join |
| `queue_locked` | 409 | reserved for the Phase 5 Step 7 push channel; unused today |
| `relay_closed` | 423 | rooms/{id}/play |
| `rate_limited` | 429 | sign-up, request-verification, google-auth/authentication, apple-auth/authentication, request-email-confirmation, request-password-change, support/tickets |
| `room_create_failed` | 500 | admin/rooms POST |
| `schedule_write_failed` | 500 | admin/rooms/{id}/schedule PUT |
| `wallet_write_failed` | 500 | wallet/withdraw, admin/withdrawals/{id}/reject |
| `ticket_write_failed` | 500 | support/tickets |
| `user_creation_failed` | 500 | sign-up, google-auth/authentication |
| `email_send_failed` | 500 | request-verification, google-auth/authentication, request-email-confirmation, request-password-change |
| `google_not_configured` | 500 | google-auth/* |
| `apple_not_configured` | 500 | apple-auth/* |
| `liqpay_not_configured` | 500 | wallet/topup, payments/liqpay/callback |
| `machine_not_configured` | 500 | admin/machine/state, admin/machine/power |
| `jwt_not_configured` | 500 | verify-code, google-auth/verify-code, auth/refresh, confirm-password-change |
| `jwt_library_missing` | 500 | verify-code, google-auth/verify-code, auth/refresh, confirm-password-change |
| `jwt_encoding_failed` | 500 | verify-code, google-auth/verify-code, auth/refresh, confirm-password-change |
| `payment_failed` | 502 | wallet/topup (planned) |
| `machine_call_failed` | 502 | admin/machine/power |
| `machine_unauthorized` | 502 | admin/machine/power |
| `machine_offline` | 503 | admin/machine/power |
| `machine_unavailable_state` | 503 | admin/machine/power |
