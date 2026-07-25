# Pusher Coin — Data Model

Catalogue of the persistence surfaces the project will need. This is **not a
migration plan** — it is the decision register that says *which storage form*
each entity uses (custom table vs CPT vs user meta vs WP option), the owning
phase, and key fields. Phase 0 produces the inventory; later phases produce
the migrations.

Existing storage already in use is documented in the first section. Future
storage is grouped by owning phase.

---

## Already in use (Phase 0 baseline)

### `wp_users` (built-in)

Standard WordPress users table. Sign-up writes here via `wp_insert_user`
(`UserController.php:117`, `GoogleAuthController.php:346`). Custom role:
`player`, registered in `app/utils/role-player.php`.

### `wp_usermeta` (built-in) — registered keys

Single source of truth: `backend/wp-content/themes/pc/app/utils/user-meta-keys.php`
(`User_Meta_Keys` class). Controllers reference constants only; no string
literals.

| Constant | Storage key | Type | Set by | Notes |
| --- | --- | --- | --- | --- |
| `PHONE` | `phone` | string | sign-up | Optional. |
| `VERIFICATION_CODE` | `verification_code` | string (6 digits) | request-verification | Cleared on success/expiry. |
| `VERIFICATION_CODE_EXPIRY` | `verification_code_expiry` | int (unix timestamp) | request-verification | 15-minute TTL. |
| `GOOGLE_ID` | `google_id` | string | google-auth/authentication | Google `sub` claim. |
| `GOOGLE_VERIFICATION_CODE` | `google_verification_code` | string (6 digits) | google-auth/authentication | Cleared on success/expiry. |
| `GOOGLE_VERIFICATION_CODE_EXPIRY` | `google_verification_code_expiry` | int (unix timestamp) | google-auth/authentication | 15-minute TTL. |
| `TERMS_ACCEPTED_AT` | `terms_accepted_at` | int (unix timestamp) | sign-up, accept-terms | Phase 1. Required before play / top-up. |
| `TERMS_ACCEPTED_VERSION` | `terms_accepted_version` | string | sign-up, accept-terms | Phase 1. Compared against `pc_terms_current_version` option. |
| `NICKNAME_CHOSEN` | `nickname_chosen` | string `'1'` | sign-up, set-nickname | Phase 1. Absent for first-time social-login users until they pick a nickname. |
| `APPLE_ID` | `apple_id` | string | apple-auth (stub) | Phase 1. Apple `sub` claim. Used when Apple is enabled. |
| `APPLE_VERIFICATION_CODE` | `apple_verification_code` | string | apple-auth (stub) | Phase 1. Mirrors Google flow. |
| `APPLE_VERIFICATION_CODE_EXPIRY` | `apple_verification_code_expiry` | int | apple-auth (stub) | Phase 1. |
| `EMAIL_VERIFIED_AT` | `email_verified_at` | int (unix timestamp) | confirm-email | Phase 2. Required for `Permissions::require_play_ready`. |
| `EMAIL_CONFIRMATION_TOKEN` | `email_confirmation_token` | string | request-email-confirmation | Phase 2. URL-safe base64; cleared on confirm/expiry. |
| `EMAIL_CONFIRMATION_EXPIRY` | `email_confirmation_expiry` | int | request-email-confirmation | Phase 2. 24-hour TTL. |
| `PASSWORD_CHANGE_CODE` | `password_change_code` | string (6 digits) | request-password-change | Phase 2. Cleared on confirm/expiry. |
| `PASSWORD_CHANGE_CODE_EXPIRY` | `password_change_code_expiry` | int | request-password-change | Phase 2. 15-minute TTL. |

### WP options (Phase 1+2)

| Option key | Type | Default | Notes |
| --- | --- | --- | --- |
| `pc_db_version` | string | `'1.4.0'` | Tracks installed schema version; `Install_Schema::maybe_install` reads / writes it. Phase 2 bumped 1.0.0 → 1.1.0; Phase 3 bumped 1.1.0 → 1.2.0 (adds `wp_pc_room_schedules`); Phase 4 Step 1 bumped 1.2.0 → 1.3.0 (adds `wp_pc_wallets`, `wp_pc_coin_lots`, `wp_pc_transactions`); Phase 4 Step 4 bumped 1.3.0 → 1.4.0 (adds `consumed_lots LONGTEXT NULL` to `wp_pc_transactions` so rejected withdrawals can re-credit at original prices); Phase 5 Step 5 bumped 1.4.0 → 1.5.0 (adds `wp_pc_machine_events`); Phase 7 bumped 1.5.0 → 1.6.0 (adds `wp_pc_support_tickets`). |
| `pc_terms_current_version` | string | `'2026-05'` | Bump when T&Cs change to force re-acceptance. |
| `pc_access_token_ttl_seconds` | int | `900` | 15 minutes. Read by `AuthController::issue_access_token` and the `jwt_auth_expire` filter. |
| `pc_refresh_token_ttl_seconds` | int | `604800` | 7 days. Read by `Refresh_Tokens`. |
| `pc_email_confirmation_ttl_seconds` | int | `86400` | 24h. Phase 2. Read by `request-email-confirmation`. |
| `pc_password_change_ttl_seconds` | int | `900` | 15min. Phase 2. Read by `request-password-change`. |
| `pc_spa_base_url` | string | `home_url()` | Phase 2. Operator-set; used to build outbound email links such as `/confirm-email?token=…`. |

### `wp_pc_refresh_tokens` (custom table — Phase 1)

Active refresh tokens. One row per issued token; `revoked_at` is set on
explicit logout, on rotation, and on reuse-detection cascade. The
plaintext token is never stored — only its SHA-256.

```
id            BIGINT       PK, AUTO_INCREMENT
user_id       BIGINT       INDEX
token_hash    CHAR(64)     UNIQUE   -- SHA-256
issued_at     DATETIME
expires_at    DATETIME     INDEX
revoked_at    DATETIME     NULL
replaced_by   CHAR(64)     NULL     -- token_hash of the rotation successor
user_agent    VARCHAR(512)
ip            VARBINARY(16) NULL
```

Replaces the placeholder `wp_pc_jwt_blacklist` reserved in earlier
drafts of this doc — we model active refresh tokens directly rather
than blacklisting access JWTs.

### `wp_pc_auth_audit_log` (custom table — Phase 1)

Append-only event log for auth events. Used by Phase 7's ops dashboard;
no admin viewer ships in Phase 1.

```
id            BIGINT       PK, AUTO_INCREMENT
event_type    VARCHAR(64)  INDEX
user_id       BIGINT       NULL INDEX
email         VARCHAR(255) NULL
ip            VARBINARY(16) NULL
user_agent    VARCHAR(512)
metadata      LONGTEXT     NULL    -- JSON
created_at    DATETIME(6)  INDEX
```

Event types written today: `signup`, `request_verification`,
`request_verification_failed`, `verify_success`, `verify_failure`,
`refresh`, `refresh_reuse`, `logout`, `accept_terms`, `set_nickname`,
`rate_limited`.

---

## Phase 2 — avatar (no persistence)

Faceless / abstract avatar tiles are rendered client-side from a
deterministic FNV-1a hash of the user ID. No DB column or meta key — see
`frontend/src/components/FacelessAvatar.vue`.

---

## Phase 3 — rooms & schedules

### CPT `pc_room`

Room is a content-y entity (name, description, theme song, stream URL)
that admins author one at a time — CPT semantics fit. Stored under
`wp_posts` / `wp_postmeta`.

Registered in `backend/wp-content/themes/pc/app/utils/cpt-room.php`. The
CPT is **not** `public` and **not** exposed via the default WP REST
namespace (`show_in_rest=false`) — rooms are read through the custom
`pc/v1/rooms` controllers so the response shape stays in
`API-CONTRACT.md`'s control.

Post fields:

- `post_title` → room name.
- `post_status` → `publish` for available, `draft` for unavailable;
  the `status` post-meta below carries the more specific state.

Post meta (constants in `Post_Meta_Keys`,
`backend/wp-content/themes/pc/app/utils/post-meta-keys.php`):

| Constant | Storage key | Type | Notes |
| --- | --- | --- | --- |
| `ROOM_STATUS` | `pc_room_status` | enum (`available`, `maintenance`, `unavailable`) | Enum values exposed as `ROOM_STATUS_*` constants. |
| `ROOM_THEME_SONG_URL` | `pc_room_theme_song_url` | string (URL) | Optional. |
| `ROOM_STREAM_URL` | `pc_room_stream_url` | string (URL) | HLS / WebRTC / LL-HLS endpoint. Phase 3 picks the transport. |
| `ROOM_MACHINE_ID` | `pc_room_machine_id` | string | Maps the room to a Home Assistant machine. |

### `wp_pc_room_schedules` (custom table)

Weekly recurring rules. Relational: each room has many rules. Not a CPT
because rule rows are pure data with no editorial content.

```
id            BIGINT   PK
room_id       BIGINT   FK → wp_posts.ID
weekday       TINYINT  -- 0=Mon..6=Sun (ISO)
start_time    TIME
end_time      TIME
recurrence    ENUM('always','once')
once_date     DATE     NULL     -- only for recurrence='once'
created_at    DATETIME
```

Computed `next_window` is derived at query time — not stored.

---

## Phase 4 — wallet, coin lots, transactions

### `wp_pc_wallets` (custom table — Phase 4 Step 1)

One row per user. Custom table because `wp_usermeta` cannot atomically
update two fields. Row is lazy-created by `Wallet_Service::credit_lot`
on the first top-up settlement; readers fall back to `0.00 / 0` if
absent.

```
user_id        BIGINT   PK, FK → wp_users.ID
balance_money  DECIMAL(12,2)  DEFAULT 0
balance_coins  INT            DEFAULT 0
updated_at     DATETIME
```

### `wp_pc_coin_lots` (custom table — Phase 4 Step 1)

Required by `ROADMAP.md` §4.2: coins purchased carry their price; a
winning coin pays back at the price it was bought at. Modelled as a
stack of `(qty, unit_price)` lots, FIFO consumption per toss /
withdrawal. `Wallet_Service::debit_fifo` decrements `qty` on the oldest
lot first; rows with `qty = 0` linger but are filtered out by readers.

```
id            BIGINT   PK
user_id       BIGINT   FK → wp_users.ID
qty           INT
unit_price    DECIMAL(8,2)
acquired_at   DATETIME
source_txn_id BIGINT   FK → wp_pc_transactions.id
```

### `wp_pc_transactions` (custom table — Phase 4 Step 1)

Append-only ledger for top-ups and withdrawals only. Game results stay
out of this table — they are derived from `wp_pc_bet_sessions` (Phase 6)
and never expose individual coin tosses to the history view.

`type` / `status` are stored as `VARCHAR(16)` (not `ENUM`) because
`dbDelta`'s diff logic mishandles `ENUM` definitions; constants live on
`Wallet_Service` (`TYPE_TOPUP` / `TYPE_WITHDRAW`, `STATUS_PENDING` /
`STATUS_COMPLETED` / `STATUS_FAILED` / `STATUS_REFUNDED`).

A free-form `notes` column was added beyond the original sketch to hold
admin reasons on `refunded` / `failed` withdrawals.

`consumed_lots` (added Phase 4 Step 4) is a JSON array of
`[{ qty, unit_price }, ...]` written when a withdrawal is requested.
On reject, the values are re-credited as new lots — preserving the
player's value even though the original lot rows may have been
deleted (qty=0). Null for top-up rows.

```
id              BIGINT   PK
user_id         BIGINT
type            VARCHAR(16)    -- 'topup' | 'withdraw'
amount_money    DECIMAL(12,2)
amount_coins    INT
unit_price      DECIMAL(8,2)
status          VARCHAR(16)    -- 'pending' | 'completed' | 'failed' | 'refunded'
external_ref    VARCHAR(128)   -- payment provider id (LiqPay order_id)
notes           TEXT
consumed_lots   LONGTEXT       -- JSON [{qty,unit_price},...], withdrawals only
created_at      DATETIME
settled_at      DATETIME
```

### WP options — coin pricing & LiqPay (Phase 4 Step 1)

Singleton config; WP options are sufficient. Stored as decimal strings
to avoid PHP float drift.

| Option key | Type | Default | Notes |
| --- | --- | --- | --- |
| `pc_coin_price_default` | decimal string | `'40.00'` | Initial value approximates 1 USD in UAH for an operator who hasn't visited the admin SPA yet. |
| `pc_coin_price_min` | decimal string | `'10.00'` | Floor enforced by `/wallet/topup`. |
| `pc_coin_price_max` | decimal string | `'500.00'` | Ceiling enforced by `/wallet/topup`. |
| `pc_liqpay_public_key` | string | `''` | LiqPay merchant public key. Operator-set in the admin SPA (Step 7). The matching **private key** lives in `wp-config.php` as `PC_LIQPAY_PRIVATE_KEY` — never in the DB. |

---

## Phase 5 — machine integration

### WP options — machine settings

Singletons; admin-edited from the admin SPA. Step 1 reads each value
with the default below when the option is absent, so a fresh install
talks to the production HA endpoint without any explicit configuration
beyond the bearer token.

| Option key | Type | Default | Notes |
| --- | --- | --- | --- |
| `pc_machine_endpoint` | string (URL) | `https://developer-it.com/api` | Home Assistant base URL. |
| `pc_machine_power_switch_entity` | string | `switch.sonoff_10024fb618` | HA entity for the wall switch. |
| `pc_machine_toss_button_entity` | string | `input_button.toss_a_coin` | HA entity that fires a coin toss. |
| `pc_machine_coin_sensor_entity` | string | `sensor.coin` | Cumulative coin counter. |
| `pc_machine_bonus_sensor_entity` | string | `sensor.lc01_12` | Bonus wheel value 1–12. |
| `pc_machine_light_sensor_entity` | string | `sensor.light_b_t` | Status-light bitfield. |
| `pc_machine_relay_sensor_entity` | string | `sensor.relay_on` | Relay-contact state read. |
| `pc_machine_relay_close_entity` | string | `input_button.relay_on` | Closes the relay. |
| `pc_machine_relay_open_entity` | string | `input_button.relay_off` | Opens the relay. |
| `pc_machine_bonus_map` | JSON `{ "1": coins, ... "12": coins }` | _set by admin_ | Coins-per-bonus-id (Phase 5 Step 3). |
| `pc_machine_relay_coin_count` | int | _set by admin_ | Coins credited when the relay closes (Phase 5 Step 3). |
| `pc_support_email` | string (email) | site `admin_email` | Where new-ticket notifications are mailed (Phase 7). |
| `pc_captcha_provider` | string | `turnstile` | `turnstile` or `hcaptcha`; picks the siteverify endpoint. |
| `pc_captcha_site_key` | string | `''` | Public captcha key, handed to the SPA. Empty ⇒ captcha disabled. Set from the admin SPA; the matching secret is the `PC_CAPTCHA_SECRET` wp-config constant, never stored here (see `CAPTCHA_SETUP.md`). |

The Home Assistant **bearer token is not stored in the database**. Keep
it in `wp-config.php` (`PC_MACHINE_TOKEN`), read by `Machine_Service`
only. This keeps the secret out of DB backups, the admin UI, and
`wp db export`. Rotation is a wp-config edit.

### `wp_pc_machine_events` (custom table)

Audit log: every coin-toss / coins-dropped / bonus-won / relay-closed
event the backend mediates. Source for the operations dashboard alerts
in Phase 7. Installed by Install_Schema 1.5.0; written through
`Machine_Event_Log`, never directly.

```
id             BIGINT   PK
machine_id     VARCHAR(64)  NOT NULL DEFAULT ''
event_type     VARCHAR(32)        -- toss|coins_dropped|bonus|relay_closed|offline
event_key      VARCHAR(191) NULL  -- UNIQUE; idempotency guard, see below
user_id        BIGINT   NULL      -- player credited, NULL when unattributed
coins_credited INT      NOT NULL DEFAULT 0
unit_price     DECIMAL(8,2) NOT NULL DEFAULT 0
status         VARCHAR(16)        -- recorded|credited|unattributed|failed
payload        LONGTEXT NULL      -- JSON
correlation_id BIGINT   NULL      -- FK → wp_pc_bet_sessions.id when applicable
created_at     DATETIME(6)        -- microsecond precision for ordering
```

Three decisions worth keeping:

- **`VARCHAR` over `ENUM`** for `event_type` / `status`. `dbDelta` cannot
  diff an `ENUM` reliably, so adding a member later would silently skip
  the migration. The allowed values live on `Machine_Event_Log` constants.
- **`event_key` is the idempotency guard.** A retried HA webhook or an
  overlapping poll collides on the unique index and is recorded once,
  never credited twice. NULL is permitted (MySQL allows repeated NULLs in
  a unique index) so keyless events still log — but a transport that
  omits the key gets at-least-once delivery, which for a payout means
  double credits. Transports must supply one.
- **Machine credits do not write `wp_pc_transactions`.** They insert a
  coin lot and move `balance_coins`, and this table is their audit trail.
  The ledger stays the money trail (top-ups / withdrawals), which is what
  ROADMAP §4.7 shows in the player's history view. Payouts are priced at
  the player's FIFO-head lot price — the price of the next coin they
  would spend — falling back to `pc_coin_price_default` for an empty
  wallet (`Machine_Ingest_Service::payout_unit_price`).

Attribution — which player a payout belongs to — is a Phase 6 concept
(`wp_pc_bet_sessions`). Until it lands, the `pc_machine_event_player`
filter returns null and events log as `unattributed` with no wallet
movement.

---

## Phase 6 — game session

### `wp_pc_bet_sessions` (custom table)

One row per turn (a player's stretch at the front of the queue). Closed
when the next player takes over or the player abandons.

```
id              BIGINT   PK
user_id         BIGINT
room_id         BIGINT
started_at      DATETIME
ended_at        DATETIME NULL
coins_played    INT      DEFAULT 0
coins_won       INT      DEFAULT 0
money_won       DECIMAL(12,2) DEFAULT 0
```

### `wp_pc_room_queues` (custom table) — optional

Volatile queue state. Could be implemented in-memory (Pusher / Soketi
presence channels) instead of persisting; Phase 6 decides. Reserve the
table name so the schema doesn't churn.

---

## Phase 7 — support

### CPT `pc_support_subject`

Admin-editable list of subject lines for the support form. CPT because
admins author one at a time and ordering matters (use `menu_order`).

Post fields:

- `post_title` → subject label shown in the dropdown.
- `menu_order` → sort order.
- `post_status` → `publish` to expose, `draft` to hide.

### `wp_pc_support_tickets` (custom table)

High-write, append-mostly. Not a CPT — tickets are not editorial content.

Installed by Install_Schema 1.6.0; written through `Support_Service`.

```
id             BIGINT   PK
user_id        BIGINT   NULL   -- nullable for guest submissions
email          VARCHAR(255)
subject_id     BIGINT          -- a pc_support_subject post ID
description    TEXT
ip             VARBINARY(16) NULL
user_agent     VARCHAR(512)
email_verified TINYINT(1)      -- captured at submission time
status         VARCHAR(16)     -- open|in_progress|resolved|closed
created_at     DATETIME
updated_at     DATETIME
```

`status` is a `VARCHAR`, not the `ENUM` this section originally drafted,
for the same reason as `wp_pc_machine_events`: `dbDelta` cannot diff an
`ENUM`, so adding a member later would silently skip the migration. The
allowed values live on `Support_Service` constants.

`email_verified` is a snapshot, not a join. It answers "was this address
verified when the ticket was filed?", which is what support weighs when
judging a claim — re-deriving it from the account later would quietly
rewrite history. Guests are always `0`.

`subject_id` has no FK constraint. Retiring a subject trashes the post
rather than deleting it, so an old ticket still resolves its label
through `get_post()`.

---

## Storage decision matrix

When introducing a new entity, pick storage by these rules:

1. **WP options** — singleton config, low write rate, fits in a
   `LONGTEXT` value. Examples: coin price bounds, machine endpoint,
   bonus map.
2. **User meta** — per-user scalars, low cardinality, no need to query
   across users. Examples: phone, terms-accepted timestamp, OAuth IDs,
   verification codes. **Always add to `User_Meta_Keys`.**
3. **CPT (`wp_posts` + `wp_postmeta`)** — editorial content with admin
   authoring UI, ordering, status. Examples: rooms, support subjects.
4. **Custom table** — relational data, high write rate, or shapes that
   don't fit `wp_postmeta`'s key-value model. Examples: schedules,
   wallets, coin lots, transactions, sessions, machine events,
   tickets, JWT blacklist.

Default to (4) for anything that smells like a ledger. CPTs are tempting
but `wp_postmeta` is a key-value store and queries get expensive fast.
