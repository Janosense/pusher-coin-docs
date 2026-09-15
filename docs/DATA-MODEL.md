# Data model — Pusher Coin

<!-- This file MUST match the actual schema at all times. A schema change
     without updating this file in the same commit = unfinished task
     (CLAUDE.md core rule 5). -->

## Conventions

**Prefix and naming.** Custom tables are `{$wpdb->prefix}pc_*` (`wp_pc_…` on a
default install). WP options are `pc_*`. Rate-limiter transients are
`pc_rl_<md5 of the bucket key>`. Columns are `snake_case`; timestamps are
`created_at` / `updated_at` / `settled_at` / `…_at`; a nullable `…_at` doubles as a
state flag (`revoked_at`, `ended_at`, `settled_at`).

**Ids.** `BIGINT` auto-increment primary keys, except `wp_pc_wallets`, whose PK *is*
`user_id` (one row per user). Foreign keys are logical, not declared — WordPress
tables are MyISAM/InnoDB-mixed and `dbDelta` does not manage constraints. One
deliberate consequence: `wp_pc_support_tickets.subject_id` has no FK, so a trashed
subject still resolves its label through `get_post()`.

**Timestamps.** `DATETIME` by default; `DATETIME(6)` (microsecond) where ordering
between rows written in the same second matters — `wp_pc_auth_audit_log`,
`wp_pc_machine_events`, `wp_pc_room_messages`.

**Money.** UAH. `DECIMAL(12,2)` for balances and amounts, `DECIMAL(8,2)` for a unit
price. Values are read and serialised as decimal strings end to end to avoid PHP and
JavaScript float drift. Option defaults for prices are stored as decimal strings too.

**Soft delete.** Nothing that carries evidence is deleted. A chat message flips
`status` to `hidden`; a retired support subject is trashed, not removed; a drained
coin lot lingers at `qty = 0` and is filtered out by readers; a revoked refresh token
keeps its row with `revoked_at` set. A lifted chat mute is the one exception — the
`chat_muted_until` meta is deleted rather than zeroed.

**`VARCHAR` over `ENUM`, always.** `dbDelta` cannot diff an `ENUM` reliably, so
adding a member later would silently skip the migration. Every status/type column is
`VARCHAR(16)`–`VARCHAR(32)` with the allowed values as class constants
(`Wallet_Service`, `Machine_Event_Log`, `Support_Service`, `Chat_Service`) and
validated on write in the controller.

**Migrations.** `backend/wp-content/themes/pc/app/utils/install-schema.php`
(`Install_Schema::maybe_install`) runs `dbDelta` and seeds default options. The
installed version is the `pc_db_version` option. Every schema change bumps it in the
same commit as this file. History:

| Version | What it added |
|---|---|
| 1.0.0 | `wp_pc_refresh_tokens`, `wp_pc_auth_audit_log` |
| 1.1.0 | Phase 2 account columns / options |
| 1.2.0 | `wp_pc_room_schedules` |
| 1.3.0 | `wp_pc_wallets`, `wp_pc_coin_lots`, `wp_pc_transactions` |
| 1.4.0 | `consumed_lots LONGTEXT NULL` on `wp_pc_transactions` |
| 1.5.0 | `wp_pc_machine_events` |
| 1.6.0 | `wp_pc_support_tickets` |
| 1.7.0 | `wp_pc_bet_sessions`, `wp_pc_room_queues` |
| 1.8.0 | `wp_pc_room_messages` |

**Meta-key registries.** A meta key is never a string literal. User meta comes from
`User_Meta_Keys` (`app/utils/user-meta-keys.php`), `pc_room` meta from
`Post_Meta_Keys` (`app/utils/post-meta-keys.php`). A new key is a constant there, a
row in this file, and nothing else.

**Storage decision matrix.** When introducing a new entity, pick storage by these
rules:

1. **WP options** — singleton config, low write rate, fits in a `LONGTEXT` value.
   Examples: coin price bounds, machine endpoint, bonus map.
2. **User meta** — per-user scalars, low cardinality, no need to query across users.
   Examples: phone, terms-accepted timestamp, OAuth ids, verification codes.
   **Always add to `User_Meta_Keys`.**
3. **CPT (`wp_posts` + `wp_postmeta`)** — editorial content with admin authoring UI,
   ordering, status. Examples: rooms, support subjects.
4. **Custom table** — relational data, high write rate, or shapes that don't fit
   `wp_postmeta`'s key-value model. Examples: schedules, wallets, coin lots,
   transactions, sessions, queues, messages, machine events, tickets.

Default to (4) for anything that smells like a ledger. CPTs are tempting but
`wp_postmeta` is a key-value store and queries get expensive fast.

## Tables

### `wp_users` (built-in)

Standard WordPress users. Sign-up writes here via `wp_insert_user`
(`UserController`, `GoogleAuthController`). Custom role `player` registered in
`app/utils/role-player.php` with caps `{ read: false, play: true }`; `play` is also
granted to `administrator`. No code checks `play` today — every gate goes through
`Permissions::*` — so the role is effectively a label.

### `wp_usermeta` (built-in) — registered keys

| Constant | Storage key | Type | Set by | Notes |
| --- | --- | --- | --- | --- |
| `PHONE` | `phone` | string | sign-up | Optional. |
| `VERIFICATION_CODE` | `verification_code` | string (6 digits) | request-verification | Cleared on success/expiry. |
| `VERIFICATION_CODE_EXPIRY` | `verification_code_expiry` | int (unix timestamp) | request-verification | 15-minute TTL. |
| `GOOGLE_ID` | `google_id` | string | google-auth/authentication | Google `sub` claim. |
| `GOOGLE_VERIFICATION_CODE` | `google_verification_code` | string (6 digits) | google-auth/authentication | Cleared on success/expiry. |
| `GOOGLE_VERIFICATION_CODE_EXPIRY` | `google_verification_code_expiry` | int (unix timestamp) | google-auth/authentication | 15-minute TTL. |
| `TERMS_ACCEPTED_AT` | `terms_accepted_at` | int (unix timestamp) | sign-up, accept-terms | Required before play / top-up. |
| `TERMS_ACCEPTED_VERSION` | `terms_accepted_version` | string | sign-up, accept-terms | Compared against `pc_terms_current_version`. |
| `NICKNAME_CHOSEN` | `nickname_chosen` | string `'1'` | sign-up, set-nickname | Absent for first-time social-login users until they pick a nickname. |
| `APPLE_ID` | `apple_id` | string | apple-auth (stub) | Apple `sub` claim. |
| `APPLE_VERIFICATION_CODE` | `apple_verification_code` | string | apple-auth (stub) | Mirrors the Google flow. |
| `APPLE_VERIFICATION_CODE_EXPIRY` | `apple_verification_code_expiry` | int | apple-auth (stub) | |
| `EMAIL_VERIFIED_AT` | `email_verified_at` | int (unix timestamp) | confirm-email | Required by `Permissions::require_play_ready`. |
| `EMAIL_CONFIRMATION_TOKEN` | `email_confirmation_token` | string | request-email-confirmation | URL-safe base64; cleared on confirm/expiry. |
| `EMAIL_CONFIRMATION_EXPIRY` | `email_confirmation_expiry` | int | request-email-confirmation | 24-hour TTL. |
| `PASSWORD_CHANGE_CODE` | `password_change_code` | string (6 digits) | request-password-change | Cleared on confirm/expiry. |
| `PASSWORD_CHANGE_CODE_EXPIRY` | `password_change_code_expiry` | int | request-password-change | 15-minute TTL. |
| `CHAT_MUTED_UNTIL` | `chat_muted_until` | int (unix timestamp) | admin/chat/mute | Account-wide chat mute; absent or in the past means not muted. Deleted rather than zeroed when lifted. |

### CPT `pc_room`

A room is content-y (name, description, theme song, stream URL) and admins author
one at a time, so CPT semantics fit. Registered in `app/utils/cpt-room.php`. The CPT
is **not** `public` and **not** exposed via the default WP REST namespace
(`show_in_rest=false`) — rooms are read through `pc/v1/rooms` so the response shape
stays under `CONTRACTS.md`'s control.

- `post_title` → room name.
- `post_status` → `publish` for available, `draft` for unavailable; the `status` post
  meta below carries the more specific state.

| Constant | Storage key | Type | Notes |
| --- | --- | --- | --- |
| `ROOM_STATUS` | `pc_room_status` | `available` \| `maintenance` \| `unavailable` | Values exposed as `ROOM_STATUS_*` constants. |
| `ROOM_THEME_SONG_URL` | `pc_room_theme_song_url` | string (URL) | Optional. |
| `ROOM_STREAM_URL` | `pc_room_stream_url` | string (URL) | HLS / LL-HLS / embed / direct-video endpoint. |
| `ROOM_MACHINE_ID` | `pc_room_machine_id` | string | Maps the room to a Home Assistant machine. The link that makes a machine event attributable. |

### CPT `pc_support_subject`

Admin-editable subject lines for the support form. CPT because admins author one at
a time and ordering matters.

- `post_title` → subject label shown in the dropdown.
- `menu_order` → sort order.
- `post_status` → `publish` to expose, `draft` to hide.

### `wp_pc_refresh_tokens`

Active refresh tokens, one row per issued token. The plaintext token is never
stored — only its SHA-256. `revoked_at` is set on explicit logout, on rotation, and
on the reuse-detection cascade.

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

Replaces the placeholder `wp_pc_jwt_blacklist` reserved in earlier drafts — active
refresh tokens are modelled directly rather than blacklisting access JWTs.

### `wp_pc_auth_audit_log`

Append-only event log. Despite the name, **every** part of the product writes here —
it is the single operator-facing audit trail, not just an auth log.

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

Event types written today, by owning area:

| Area | Event types |
| --- | --- |
| auth | `signup`, `request_verification`, `request_verification_failed`, `verify_success`, `verify_failure`, `refresh`, `refresh_reuse`, `logout`, `accept_terms`, `set_nickname`, `rate_limited` |
| account | `request_email_confirmation`, `confirm_email`, `request_password_change`, `password_change`, `password_change_failure` |
| payments | `liqpay_payload_invalid`, `liqpay_signature_invalid`, `liqpay_callback_misconfigured`, `liqpay_callback_unknown_order`, `liqpay_topup_settled`, `liqpay_topup_settle_failed`, `liqpay_topup_failed` |
| withdrawals | `withdrawal_approved`, `withdrawal_rejected` |
| machine (admin actions) | `machine_power_changed`, `machine_bonus_map_updated` |
| chat | `chat_message_moderated`, `chat_user_muted` |
| support | `support_ticket_created`, `support_ticket_updated`, `support_subjects_updated`, `support_captcha_updated` |

Machine *events* (tosses, drops, bonuses) do not go here — they have their own table.
Adding an event type is a code change in the owning controller plus a row above.

### `wp_pc_room_schedules`

Weekly recurring rules; each room has many. Not a CPT because rule rows are pure
data with no editorial content.

```
id            BIGINT   PK
room_id       BIGINT   -> wp_posts.ID
weekday       TINYINT  -- 0=Mon..6=Sun (ISO)
start_time    TIME
end_time      TIME
recurrence    VARCHAR(16)  DEFAULT 'always'  -- 'always' | 'once'
once_date     DATE     NULL     -- only for recurrence='once'
created_at    DATETIME
```

`next_window` / `current_window` are derived at query time by
`Room_Schedule_Calculator`, never stored. A schedule save is an atomic replace
(`PUT /admin/rooms/{id}/schedule`), not a per-row edit.

### `wp_pc_wallets`

One row per user. A custom table because `wp_usermeta` cannot atomically update two
fields. Lazy-created by `Wallet_Service::credit_lot` on the first settlement;
readers fall back to `0.00 / 0` when absent.

```
user_id        BIGINT   PK  -> wp_users.ID
balance_money  DECIMAL(12,2)  DEFAULT 0
balance_coins  INT            DEFAULT 0
updated_at     DATETIME
```

### `wp_pc_coin_lots`

Purchased coins carry their price, so a winning coin pays back at the price it was
bought at. A stack of `(qty, unit_price)` lots consumed FIFO per toss / withdrawal.
`Wallet_Service::debit_fifo` decrements `qty` on the oldest lot first; rows at
`qty = 0` linger and are filtered out by readers.

```
id            BIGINT   PK
user_id       BIGINT   -> wp_users.ID
qty           INT
unit_price    DECIMAL(8,2)
acquired_at   DATETIME
source_txn_id BIGINT   -> wp_pc_transactions.id
```

### `wp_pc_transactions`

Append-only ledger for **top-ups and withdrawals only**. Game results stay out: they
are derived from `wp_pc_bet_sessions`, and individual tosses are never exposed to
the history view.

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

Constants live on `Wallet_Service` (`TYPE_TOPUP` / `TYPE_WITHDRAW`,
`STATUS_PENDING` / `STATUS_COMPLETED` / `STATUS_FAILED` / `STATUS_REFUNDED`).
`notes` holds the admin reason on `refunded` / `failed` withdrawals. `consumed_lots`
is written when a withdrawal is requested; on reject the values are re-credited as
new lots, preserving the player's value even though the original lot rows have since
been drained to `qty = 0`. Null for top-up rows.

### `wp_pc_machine_events`

Audit log of every coin-toss / coins-dropped / bonus-won / relay-closed event the
backend mediates. Written through `Machine_Event_Log`, never directly.

```
id             BIGINT   PK
machine_id     VARCHAR(64)  NOT NULL DEFAULT ''
event_type     VARCHAR(32)        -- toss|coins_dropped|bonus|relay_closed|offline
event_key      VARCHAR(191) NULL  -- UNIQUE; idempotency guard
user_id        BIGINT   NULL      -- player credited, NULL when unattributed
coins_credited INT      NOT NULL DEFAULT 0
unit_price     DECIMAL(8,2) NOT NULL DEFAULT 0
status         VARCHAR(16)        -- recorded|credited|unattributed|failed
payload        LONGTEXT NULL      -- JSON
correlation_id BIGINT   NULL      -- reserved for wp_pc_bet_sessions.id; always NULL today
created_at     DATETIME(6)        -- microsecond precision for ordering
```

- **`event_key` is the idempotency guard.** A retried webhook or an overlapping poll
  collides on the unique index and is recorded once, never credited twice. NULL is
  permitted (MySQL allows repeated NULLs in a unique index) so keyless events still
  log — but a transport that omits the key gets at-least-once delivery, which for a
  payout means double credits. **Transports must supply one.**
- **Machine credits do not write `wp_pc_transactions`.** They insert a coin lot and
  move `balance_coins`; this table is their audit trail. The ledger stays the money
  trail, which is what the player's history view shows. Payouts are priced at the
  player's FIFO-head lot price — the price of the next coin they would spend —
  falling back to `pc_coin_price_default` for an empty wallet
  (`Machine_Ingest_Service::payout_unit_price`).
- `correlation_id` is only passed through from the ingest `$context` and no caller
  supplies it, so it is NULL on every row. The event → session link is recorded on
  the *session* side instead. The column stays reserved for a transport that wants a
  row-level back-reference.

### `wp_pc_bet_sessions`

One row per turn — a player's stretch at the front of the queue. Closed when the
next player takes over or the player abandons. Written through `Queue_Service`.

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

Wins land here through the `pc_machine_event_credited` action, so the in-room
winnings counter is per-turn rather than lifetime.

### `wp_pc_room_queues`

Persisted, not in-memory: the turn decides who gets paid for a bonus, so it has to
survive a page reload, a backend restart, and the arrival of a machine event seconds
after the player's tab was backgrounded — none of which presence state guarantees.

```
id              BIGINT   PK
room_id         BIGINT
user_id         BIGINT
coins_declared  INT      -- what the player said they'd play
coins_remaining INT      -- what's left of it
session_id      BIGINT   NULL  -- wp_pc_bet_sessions.id, set at the head
joined_at       DATETIME       -- FIFO ordering key
last_seen_at    DATETIME       -- heartbeat; stale entries are pruned
UNIQUE KEY (room_id, user_id)
```

Entries are pruned when `last_seen_at` falls further behind than
`pc_queue_idle_timeout_seconds`. Pruning happens **on read** — every queue request
cleans the room before answering — so the queue heals on traffic alone and needs no
cron. A room nobody is watching may hold a stale head, but nothing can happen in it
either. `coins_declared` is an intent, not a reservation: coins are debited one at a
time by `POST /rooms/{id}/play`, so a player who tops up mid-turn isn't penalised and
one who spends elsewhere runs out early.

### `wp_pc_room_messages`

In-room chat. Written through `Chat_Service`, never directly.

```
id          BIGINT   PK
room_id     BIGINT
user_id     BIGINT
body        VARCHAR(500)
status      VARCHAR(16)     -- visible|hidden
ip          VARBINARY(16) NULL
created_at  DATETIME(6)
KEY room_status_id (room_id, status, id)
KEY user_created (user_id, created_at)
```

- **Reads are cursor-based on `id`, not offset-based.** The SPA polls
  `GET /rooms/{id}/messages?after=<last id>` every 3s. Under `LIMIT/OFFSET` a
  conversation that gains rows between two polls would re-send or skip messages; a
  cursor cannot. `room_status_id` is the index that serves it.
- **Moderation hides, it does not delete.** A hidden row keeps its author, body, IP
  and timestamp for whoever reviews the complaint. The player-facing read filters to
  `visible`.
- **No nickname column.** Authors are resolved live through `get_userdata`, so a
  player who renames themselves does not leave two names in one conversation. This
  is the opposite call to `wp_pc_support_tickets.email_verified`, and for the
  opposite reason: a ticket records what was true at submission time, a chat line
  just needs to say who is speaking now.

Muting is not stored here — it is `chat_muted_until` user meta, a per-user scalar.
The mute is account-wide rather than per-room: someone worth silencing in one room is
worth silencing in the next, and a per-room mute would need a table of its own for a
rule nobody has asked for.

### `wp_pc_support_tickets`

High-write, append-mostly. Not a CPT — tickets are not editorial content. Written
through `Support_Service`.

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

`email_verified` is a snapshot, not a join: it answers "was this address verified
when the ticket was filed?", which is what support weighs when judging a claim —
re-deriving it later would quietly rewrite history. Guests are always `0`.
`subject_id` has no FK: retiring a subject trashes the post rather than deleting it,
so an old ticket still resolves its label.

### WP options

**Core / auth / account**

| Option key | Type | Default | Notes |
| --- | --- | --- | --- |
| `pc_db_version` | string | `'1.8.0'` | Installed schema version; read/written by `Install_Schema::maybe_install`. |
| `pc_terms_current_version` | string | `'2026-05'` | Bump when T&Cs change to force re-acceptance. |
| `pc_access_token_ttl_seconds` | int | `900` | Read by `AuthController::issue_access_token` and the `jwt_auth_expire` filter. |
| `pc_refresh_token_ttl_seconds` | int | `604800` | 7 days. Read by `Refresh_Tokens`. |
| `pc_email_confirmation_ttl_seconds` | int | `86400` | Read by `request-email-confirmation`. |
| `pc_password_change_ttl_seconds` | int | `900` | Read by `request-password-change`. |
| `pc_spa_base_url` | string | `home_url()` | Operator-set; builds outbound email links such as `/confirm-email?token=…`. |

**Coin pricing & LiqPay** — stored as decimal strings to avoid PHP float drift.

| Option key | Type | Default | Notes |
| --- | --- | --- | --- |
| `pc_coin_price_default` | decimal string | `'40.00'` | Approximates 1 USD in UAH for an operator who hasn't visited the admin SPA yet. |
| `pc_coin_price_min` | decimal string | `'10.00'` | Floor enforced by `/wallet/topup`. |
| `pc_coin_price_max` | decimal string | `'500.00'` | Ceiling enforced by `/wallet/topup`. |
| `pc_liqpay_public_key` | string | `''` | Merchant public key, operator-set in the admin SPA. The private key is `PC_LIQPAY_PRIVATE_KEY` in wp-config — never in the DB. |

**Machine** — defaults match `PUSHER-COIN-COMMANDS.txt`, so a fresh install talks to
the production HA endpoint with no configuration beyond the bearer token.

| Option key | Type | Default | Notes |
| --- | --- | --- | --- |
| `pc_machine_endpoint` | string (URL) | `https://developer-it.com/api` | Home Assistant base URL. |
| `pc_machine_power_switch_entity` | string | `switch.sonoff_10024fb618` | Wall switch. |
| `pc_machine_toss_button_entity` | string | `input_button.toss_a_coin` | Fires a coin toss. |
| `pc_machine_coin_sensor_entity` | string | `sensor.coin` | Cumulative coin counter. |
| `pc_machine_bonus_sensor_entity` | string | `sensor.lc01_12` | Bonus wheel value 1–12. |
| `pc_machine_light_sensor_entity` | string | `sensor.light_b_t` | Status-light bitfield. |
| `pc_machine_relay_sensor_entity` | string | `sensor.relay_on` | Relay-contact state read. |
| `pc_machine_relay_close_entity` | string | `input_button.relay_on` | Closes the relay. |
| `pc_machine_relay_open_entity` | string | `input_button.relay_off` | Opens the relay. |
| `pc_machine_bonus_map` | JSON `{ "1": coins, … "12": coins }` | set by admin | Coins per bonus id. |
| `pc_machine_relay_coin_count` | int | set by admin | Coins credited when the relay closes. |

The HA **bearer token is not stored in the database** — `PC_MACHINE_TOKEN` in
wp-config, read by `Machine_Service` only. This keeps the secret out of DB backups,
the admin UI and `wp db export`. Rotation is a wp-config edit.

**Queue**

| Option key | Type | Default | Notes |
| --- | --- | --- | --- |
| `pc_queue_idle_timeout_seconds` | int | `60` | How long a queue entry survives without a heartbeat; the SPA's 3s queue poll is the heartbeat. `Queue_Service::idle_timeout` floors it at 10. Not exposed in the admin SPA yet. |

**Support & captcha**

| Option key | Type | Default | Notes |
| --- | --- | --- | --- |
| `pc_support_email` | string (email) | site `admin_email` | Where new-ticket notifications are mailed; the player's address goes in `Reply-To`. |
| `pc_captcha_provider` | string | `turnstile` | `turnstile` or `hcaptcha`; picks the siteverify endpoint in `Captcha_Verifier`. |
| `pc_captcha_site_key` | string | `''` | Public captcha key, handed to the SPA via `GET /support/subjects`. Empty ⇒ captcha disabled and guests submit unchallenged. The matching secret is `PC_CAPTCHA_SECRET` in wp-config. **Launch blocker while empty** — see `backend/wp-content/themes/pc/CAPTCHA_SETUP.md`. |

### Transients — rate limiting

`Rate_Limiter` keeps counters in WP transients named `pc_rl_<md5 of the bucket key>`.
The bucket key is the action plus the client IP (`signup`, `request_verification`,
`verify`, `google_auth`, `apple_auth`, `support_ticket`) or the action plus the user
id for the two per-account email flows (`request_email_confirmation`,
`request_password_change`). They expire with their window, are not options, and
nothing else reads them. Listed so the prefix is reserved.

### Not persisted

Faceless / abstract avatar tiles are rendered client-side from a deterministic FNV-1a
hash of the user id — no column, no meta key. See
`frontend/src/components/FacelessAvatar.vue`.

## Relations

```
wp_users 1 ──1  wp_pc_wallets            (PK = user_id)
wp_users 1 ──*  wp_pc_coin_lots          (user_id)
wp_users 1 ──*  wp_pc_transactions       (user_id)
wp_users 1 ──*  wp_pc_refresh_tokens     (user_id; replaced_by chains rotations)
wp_users 1 ──*  wp_pc_bet_sessions       (user_id)
wp_users 0/1 ─* wp_pc_support_tickets    (user_id NULL for guests)

wp_pc_transactions 1 ──* wp_pc_coin_lots (source_txn_id — which top-up bought the lot)

pc_room (wp_posts) 1 ──* wp_pc_room_schedules  (room_id)
pc_room            1 ──* wp_pc_room_queues     (room_id; UNIQUE (room_id, user_id))
pc_room            1 ──* wp_pc_bet_sessions    (room_id; at most one with ended_at IS NULL)
pc_room            1 ──* wp_pc_room_messages   (room_id)

pc_support_subject (wp_posts) 1 ──* wp_pc_support_tickets  (subject_id, no FK by design)

wp_pc_room_queues 0/1 ── 1 wp_pc_bet_sessions  (session_id, set only for the head)

wp_pc_machine_events ──  pc_room   via machine_id = pc_room_machine_id post meta
                     ──  wp_users  via the room's open bet session
                         (the `pc_machine_event_player` filter; correlation_id unused)
```

## Invariants

1. **`pc_db_version` and this file move together.** A schema change that does not
   bump `Install_Schema` and update this document in the same commit is unfinished.
2. **No `ENUM` columns.** Every status/type is `VARCHAR` with constants in PHP.
3. **Only `Wallet_Service` writes `wp_pc_wallets`, `wp_pc_coin_lots` and
   `wp_pc_transactions`**, and every mutation runs under `SELECT … FOR UPDATE`.
4. **Coin lots are FIFO and never deleted** — drained to `qty = 0` and filtered out
   by readers, so a refund can re-credit at the original price.
5. **`wp_pc_transactions` is the money trail only.** Top-ups and withdrawals. Machine
   payouts credit coin lots and are audited in `wp_pc_machine_events` instead.
6. **A transaction reaches `completed` only through the LiqPay callback**, which is
   idempotent on `(order_id, status)`.
7. **One pending withdrawal per player at a time**; `consumed_lots` must be populated
   before a withdrawal row is written, or a reject cannot refund.
8. **`wp_pc_machine_events.event_key` is unique and a transport must supply it.**
   Without it a retry double-credits.
9. **At most one open bet session per room** (`ended_at IS NULL`). This is what makes
   a machine event attributable; break it and payouts go to the wrong player.
10. **`wp_pc_room_queues` has one entry per (room, user)** and is pruned on read, not
    by cron.
11. **Nothing that carries evidence is hard-deleted** — chat rows flip `status`,
    support subjects are trashed, refresh tokens keep `revoked_at`.
12. **Meta keys come from `User_Meta_Keys` / `Post_Meta_Keys`.** A literal meta-key
    string is a defect.
13. **Secrets never reach the database**: `JWT_AUTH_SECRET_KEY`,
    `PC_LIQPAY_PRIVATE_KEY`, `PC_MACHINE_TOKEN`, `PC_CAPTCHA_SECRET`,
    `GOOGLE_CLIENT_ID`, `APPLE_*` are wp-config constants. Only their public
    counterparts are options.
14. **Money is `DECIMAL`, read and written as decimal strings.** A float anywhere in
    a money path is a defect.
