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

---

## Phase 1 — auth hardening

### User meta — additions

Add to `User_Meta_Keys`:

| Constant | Storage key | Type | Notes |
| --- | --- | --- | --- |
| `TERMS_ACCEPTED_AT` | `terms_accepted_at` | int (unix timestamp) | Set on sign-up + first social login + explicit accept. Required before play / top-up. |
| `APPLE_ID` | `apple_id` | string | Apple `sub` claim. |
| `APPLE_VERIFICATION_CODE` | `apple_verification_code` | string | Mirrors Google flow. |
| `APPLE_VERIFICATION_CODE_EXPIRY` | `apple_verification_code_expiry` | int | |

### `wp_pc_jwt_blacklist` (custom table)

Token revocation list for explicit logout. Storage form: custom table —
high write rate, append-only, no need for CPT semantics.

```
id              BIGINT   PK
jti             VARCHAR(64) UNIQUE  -- JWT id claim
user_id         BIGINT
revoked_at      DATETIME
expires_at      DATETIME            -- garbage collection
```

A daily WP-Cron job purges rows past `expires_at`.

---

## Phase 2 — account & verification gates

### User meta — additions

| Constant | Storage key | Type | Notes |
| --- | --- | --- | --- |
| `EMAIL_VERIFIED_AT` | `email_verified_at` | int (unix timestamp) | Null until confirm-email succeeds. |
| `EMAIL_CONFIRMATION_TOKEN` | `email_confirmation_token` | string | 24-hour TTL; cleared on confirm. |
| `EMAIL_CONFIRMATION_EXPIRY` | `email_confirmation_expiry` | int | |

### Avatar generation

Faceless / abstract tile. Stored as a deterministic seed on the user
(`avatar_seed`, user meta) and rendered client-side (no image stored).

---

## Phase 3 — rooms & schedules

### CPT `pc_room`

Room is a content-y entity (name, description, theme song, stream URL)
that admins author one at a time — CPT semantics fit. Stored under
`wp_posts` / `wp_postmeta`.

Post fields:

- `post_title` → room name.
- `post_status` → `publish` for available, `draft` for unavailable;
  the `status` post-meta below carries the more specific state.

Post meta:

| Key | Type | Notes |
| --- | --- | --- |
| `pc_room_status` | enum (`available`, `maintenance`, `unavailable`) | |
| `pc_room_theme_song_url` | string (URL) | Optional. |
| `pc_room_stream_url` | string (URL) | HLS / WebRTC / LL-HLS endpoint. Phase 3 picks the transport. |
| `pc_room_machine_id` | string | Maps the room to a Home Assistant machine. |

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

### `wp_pc_wallets` (custom table)

One row per user. Custom table because `wp_usermeta` cannot atomically
update two fields.

```
user_id        BIGINT   PK, FK → wp_users.ID
balance_money  DECIMAL(12,2)  DEFAULT 0
balance_coins  INT            DEFAULT 0
updated_at     DATETIME
```

### `wp_pc_coin_lots` (custom table)

Required by `ROADMAP.md` §4.2: coins purchased carry their price; a
winning coin pays back at the price it was bought at. Modelled as a
stack of `(qty, unit_price)` lots, FIFO consumption per toss.

```
id            BIGINT   PK
user_id       BIGINT   FK → wp_users.ID
qty           INT
unit_price    DECIMAL(8,2)
acquired_at   DATETIME
source_txn_id BIGINT   FK → wp_pc_transactions.id
```

### `wp_pc_transactions` (custom table)

Append-only ledger for top-ups and withdrawals only. Game results stay
out of this table — they are derived from `wp_pc_bet_sessions` (Phase 6)
and never expose individual coin tosses to the history view.

```
id              BIGINT   PK
user_id         BIGINT
type            ENUM('topup','withdraw')
amount_money    DECIMAL(12,2)
amount_coins    INT
unit_price      DECIMAL(8,2)
status          ENUM('pending','completed','failed','refunded')
external_ref    VARCHAR(128)   -- payment provider id
created_at      DATETIME
settled_at      DATETIME
```

### WP options — coin pricing rules

Singleton config; WP options are sufficient.

| Option key | Type | Notes |
| --- | --- | --- |
| `pc_coin_price_default` | decimal | Initial value: `1.00`. |
| `pc_coin_price_min` | decimal | |
| `pc_coin_price_max` | decimal | |

---

## Phase 5 — machine integration

### WP options — machine settings

Singletons; admin-edited from the admin SPA.

| Option key | Type | Notes |
| --- | --- | --- |
| `pc_machine_bonus_map` | JSON `{ "1": coins, ... "12": coins }` | Coins-per-bonus-id. |
| `pc_machine_relay_coin_count` | int | Coins credited when the relay closes. |
| `pc_machine_endpoint` | string (URL) | Home Assistant base URL. |

The Home Assistant **bearer token is not stored in the database**. Keep
it in `wp-config.php` (`PC_MACHINE_TOKEN`) or env, read by
`MachineService` only. This keeps the secret out of DB backups and
admin UI.

### `wp_pc_machine_events` (custom table)

Audit log: every coin-toss / coins-dropped / bonus-won / relay-closed
event the backend mediates. Source for the operations dashboard alerts
in Phase 7.

```
id            BIGINT   PK
machine_id    VARCHAR(64)
event_type    ENUM('toss','coins_dropped','bonus','relay_closed','offline')
payload       JSON
correlation_id BIGINT  NULL  -- FK → wp_pc_bet_sessions.id when applicable
created_at    DATETIME(6)    -- microsecond precision for ordering
```

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

```
id             BIGINT   PK
user_id        BIGINT   NULL   -- nullable for guest submissions
email          VARCHAR(255)
subject_id     BIGINT   FK → wp_posts.ID  (a pc_support_subject post)
description    TEXT
ip             VARBINARY(16)
user_agent     VARCHAR(512)
email_verified TINYINT(1)
status         ENUM('open','in_progress','resolved','closed')
created_at     DATETIME
updated_at     DATETIME
```

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
