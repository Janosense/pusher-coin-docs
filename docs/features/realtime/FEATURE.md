# Feature — realtime

<!-- playbook: v1.21. Lightweight ARCHITECTURE + DATA-MODEL for one
     feature. Lives at docs/features/realtime/FEATURE.md next to its
     sprints/. Root ARCHITECTURE.md holds only one row + a link here.
     Keep ≤80 lines. -->

## Purpose & scope
Makes the physical machine's events reach the right player by themselves. Today
nothing carries a coin drop, a bonus or a relay close from Home Assistant into
WordPress — the only producer is `wp pc machine-ingest` run by hand — so
settlement is wired and idle, and the room keeps three 3-second polls alive to
fake liveness. This feature builds the inbound transport, the outbound channel to
the browser, the attribution guarantees that make a payout land on the player who
held the turn, and the alerts that tell an operator when the machine misbehaves.
The machine itself is switched on and off by hand at the venue and is assumed on
throughout (`DECISIONS.md` 2026-09-18); nothing here controls its power.

**Out of scope:** the operations dashboard that aggregates machine, withdrawals,
pricing and tickets into one screen; all Phase 8 launch readiness; the parked
Google, Apple and captcha integrations; the admin **Machine** screen's On/Off
control, which is a `core` product question; and the rest of `BACKEND-REVIEW.md`
beyond §10, §12 and §15, which goes through `/adhoc`. See Roadmap for where each lands.

## Fit into the host
- **Code location:** `backend/wp-content/themes/pc/app/realtime/`, `frontend/src/services/realtime.js`, `admin/src/services/realtime.js`
- **Host area:** the repository root — the project's single code area, governed by the root `CLAUDE.md`
- **Entry point:** one `require_once TEMPLATE_DIR . '/app/realtime/bootstrap.php'` line in `functions.php`, the same pattern `app/stripe/bootstrap.php` uses; in each SPA, one import from the store that consumes the channel.
  Confirmed 2026-09-18: the line goes in the `Features` block next to `stripe`'s
  (`functions.php:14-17`) — after `app/utils.php` (`:12`), so `core`'s services and the
  two hooks (`queue-service.php:496-497`) exist, and before `app/rest-api.php` (`:23`).
  Routes register on `rest_api_init` inside the bootstrap (`app/stripe/bootstrap.php:23-26`),
  never through `app/rest-api.php`. SPA side: `frontend/` only, from Sprint 2.
- **Check coverage** (audited 2026-09-18): `backend/bin/check` lints every PHP file under the
  theme, so `app/realtime/` from its first file, and runs any new `tests/*.php` unedited;
  `frontend/bin/check` lints and builds all of `src/`. Both exit 0 on `main`. Gaps: the money
  checks run only with DDEV up (else a boxed `SKIPPED`, exit 0) and CI never runs them; a
  S1.5 artefact outside the theme — a worker, a cron line, an HA automation kept as
  documentation — is covered by no check, so S1.5's plan names its own.
- **`admin/`:** no step of Sprints 1–3 edits it, so its missing check script does not bite.
  S1.3's check runs on the **Room form** (`RoomFormView.vue`, not Room list), which already
  shows the server's `message` (`admin/src/stores/rooms.js`). Still stale or unbuilt, with no
  step: `MachineView.vue:156,160,168` names `sensor.coin` / "Relay closed (`sensor.relay_on`)"
  and promises push from "Phase 5 Step 4" — wrong if S1.2 moves the relay entity (`/adhoc`);
  `admin/src/services/realtime.js`, a code path above, is the Roadmap's "left for later".
- **`BACKEND-REVIEW.md` items:** §12 (two rooms, one machine) → S1.3, named; its citation
  `machine-service.php:38` is now `power_on()` — the attribution site is `queue-service.php:435`.
  §15 (duplicate sessions) → S2.5, **settled**. §10, bullet by bullet: a database error reported as a
  duplicate (`machine-events.php:85`) → S1.4 depends on it, does not name it; a failed refund
  loses the coin (`RoomQueueController.php:187`) → **unassigned**; the 2-s timeout makes a slow
  toss free (`machine-service.php:30`) → **unassigned**. Purpose & scope puts all of §10 here, so
  the two unassigned bullets need re-planning (an appended step) or `/adhoc`.
- **Shared code it depends on** — all owned by `core`; editing any of it is a plan task
  marked **"touches shared code"**. Delta-audit 2026-09-18 (Sprint 1 Step 1), file:line
  on `main` (backend `5ebe9610`, frontend `7210c59`); "S1.4" = Sprint 1 Step 4.
  - Backend, `backend/wp-content/themes/pc/`:
    - `app/utils/machine-ingest-service.php` — the only door to `wp_pc_machine_events` and the credit. `settle()` reports any failed `record()` as `duplicate: true` (`:137-145`), and writes the row before crediting with no transaction and `mark()` unchecked (`:129-173`): a crash in between leaves a row every replay skips. `ingest_coins_dropped:79` expects a delta. S1.4, S1.5.
    - `app/utils/machine-events.php` — `record()` returns 0 for a duplicate key *and* for a failed insert (`:82-85`); S1.4's "already recorded" answer has to tell them apart.
    - `app/utils/queue-service.php` — attribution. `room_id_for_machine:435` takes the first `publish`/`draft` room with the machine id, ignoring `available` (S1.3). `resolve_player_for_machine:386` runs `sync_turn` before answering, and a last declared coin closes the turn on the spot (`consume_coin:237-240`), so a payout landing after it goes to the **next** head, or to nobody. `sync_turn` was the §15 race; **settled S2.5** — `open_room_id` under `UNIQUE KEY open_room` makes the database refuse a second open session and a lost race adopt the winner's. Hooks `:496-497`.
    - `app/utils/machine-service.php` — sensor reads. `get_coin_count:61` documented cumulative; `get_relay_closed:76` reads `sensor.relay_on` via `normalise_truthy:248`, which takes the idle `1` as "closed"; `HTTP_TIMEOUT:30`; `is_online:115`. S1.2 settles the model; S1.4, S2.3, S3.1.
    - `app/utils/cli/machine-ingest.php` — today's only producer; calls all three ingest methods (`:50-58`), so S1.4's change to `ingest_coins_dropped` reaches it.
    - `app/rest-api/RoomQueueController.php` — the toss: relay check → 423 (`:129-135`), debit, `toss_coin`, `refund():187` with its result ignored, toss logged `:155`. S2.3, S3.2.
    - `app/rest-api/AdminRoomController.php` — `create_room:105` and `update_room:135` both write status and machine id in `write_room_meta:352`: where S1.3's refusal goes.
    - `app/utils/install-schema.php` — `DB_VERSION:15` (`1.9.0`), `install_default_options:253`. Seeds **no** `pc_machine_*` option: the bonus map, relay count and entity ids are code defaults, though TECH-STACK → ANTI-PATTERNS says they are seeded here. S1.4, S2.1, Sprint 3.
    - `app/utils/rate-limiter.php` — `check:21`; `client_ip:42` trusts `X-Forwarded-For` (review item 5, open), so S1.4 must not key the ingest limit on the caller's IP alone.
    - `app/utils/audit-log.php` — `record:21` into `wp_pc_auth_audit_log`, insert unchecked. S1.4 audits every call; Sprint 3 reads it.
    - `app/utils/room-schedule-calculator.php` — `compute:43`, site timezone: how S3.1 tells the daily power-off from an outage.
    - `app/utils/support-service.php` — `notify_support:271` mails `pc_support_email` but is **private**; S3.1 reusing it changes shared code.
    - `functions.php` — the entry line (above).
  - Player SPA, `frontend/src/`:
    - `stores/queue.js` — **rewritten S2.2**: subscribes to the room channel, re-reads on a version it has not seen, heartbeats at a third of the idle timeout, falls back to the 3-s poll only when the channel cannot be established. Winnings are still the open session's `coinsWon`, now also incremented optimistically from a pushed `credit` and reconciled by the next read.
    - `stores/wallet.js` — the balance. The Room screen fetches it on entry (`views/RoomView.vue:54`) and sets it after a toss (`stores/queue.js:113`), never on a poll: a machine credit shows at once as winnings, but in the balance only after the next toss or a reload. Bears on S1.5's check and S2.2.
    - `stores/chat.js` — 3-s poll on the `after` cursor (`:22`, `:82`), started from `components/RoomChat.vue:64,74`. S2.4.
    - `components/PlaceBet.vue` — the toss button (`:176-182`), the 423 message (`:31`). S2.3.
    - `components/UserControls.vue` — balance `:25`, winnings `:31`. S2.2.
    - `components/RoomQueue.vue`, `components/RoomChat.vue` — render the stores; no step changes them.
- **Conflicts with the siblings' invariants:**
  - `core` 2 / root invariant 8 (`Machine_Service` is the only caller of Home Assistant): S1.5's WebSocket-worker option would be a second client outside WordPress; the HA-automation option (HA calls WordPress) is not. S1.2's entry answers it if it picks the worker.
  - `core` 3 (one open session per room makes a payout attributable): **enforced since S2.5** — `wp_pc_bet_sessions.open_room_id` under `UNIQUE KEY open_room`, plus a one-off cleanup of the sessions the race left open and `wp pc queue-sessions` to see the state of it. The head's `session_id` and the room's open session are now kept identical, which is the half that made the race cost money. Still true: the last-coin handover above sends a late payout to the next player, against the Sprint 1 goal "the player who holds the turn". **No step names it**; S1.2's latency says how often it bites.
  - `stripe`: none. `realtime` never writes the ledger or a transaction status; it only calls `Wallet_Service::credit_lot()` (`wallet-service.php:256`). `stripe`'s FEATURE.md expects `realtime` in `stores/wallet.js` in "their Sprint 2"; no `realtime` step names that file — the path to it is `stores/queue.js`.

## Data
Owns no table. It writes `wp_pc_machine_events` **only through
`Machine_Ingest_Service`** — the row shape and its `event_key` unique index belong
to `core` (`docs/DATA-MODEL.md`). Owns these keys, and no other feature writes them:
- WP options `pc_realtime_*` — channel name, alert thresholds and windows, any
  transport setting the spike's choice needs. Defaults seeded by `Install_Schema`.
  Shipped so far: `pc_realtime_ingest_rate_max` (120) and
  `pc_realtime_ingest_rate_window_seconds` (60), the ingest endpoint's ceiling
  (S1.4); `pc_realtime_poll_interval_seconds` (60),
  `pc_realtime_poll_machine_id` (`''`, **required** — the poller holds its cursor
  until it is set) and `pc_realtime_poll_backfill_seconds` (3600), the transport
  (S1.5), plus `pc_realtime_poll_last_run`, written at runtime rather than seeded,
  and `pc_realtime_relay_state` (S2.3), also written at runtime and only when the
  relay actually moves — the last known lock state the room screen paints with;
  and `pc_realtime_channel_prefix` (`'pc'`) and `pc_realtime_token_ttl_seconds`
  (3600), the push channel (S2.1).
- Transients `pc_realtime_cursor_*` — last-seen sensor state; the spike picked
  polling, so `pc_realtime_cursor_sensor_coin` holds the `last_updated` of the last
  row delivered. Rebuildable; never a source of truth for money — `event_key` is.
  `pc_realtime_poll_lock` keeps two passes from overlapping.
- wp-config constants `PC_MACHINE_INGEST_SECRET` (the ingest shared secret, S1.4)
  and `PC_ABLY_KEY` (the push-channel app key in Ably's `name:secret` form, S2.1).
  Never options, never logged, and never sent to a SPA — the token endpoint signs
  *with* the Ably key and does not contain it.

## Invariants
1. **Every accepted inbound event carries an `event_key`.** A transport that
   cannot produce a stable one is not accepted: `POST /machine/events` refuses a body
   without one (`missing_event_key`), and a key already on file answers
   `already_recorded` and credits nothing. A row that could not be written is a 500,
   never a duplicate — the two are opposite instructions to a transport.
2. **Publishing and alerting are fire-and-forget.** A failure is logged and
   swallowed; it never fails, rolls back or delays the money path that triggered it.
3. **The SPA never sees the Ably key** — it asks for a scoped token.
4. **The ingest endpoint is not public.** Shared secret
   (`PC_MACHINE_INGEST_SECRET`, `hash_equals`), rate-limited on one ceiling across
   all callers — never per IP, which `Rate_Limiter::client_ip()` takes from a
   spoofable header — and audited on every call. A bad secret, a missing header and
   an unconfigured server are one 401 that says nothing about why; the audit log
   tells them apart. The limit is checked before the secret, so an unauthenticated
   flood cannot fill the audit log.
5. **A machine id is carried by at most one available room.** The admin API
   refuses a second (`machine_already_in_use`), a queue join into a room caught in
   such a pair (old data) is refused with the same code, and `wp pc machine-rooms`
   reports shared ids; all three go through `Machine_Rooms`, as does attribution
   (`Queue_Service::room_id_for_machine()` resolves the available room, and nobody
   when two claim the id). Empty ids claim nothing. And
   `Machine_Service` drives one physical machine whatever the id, so the id must
   name the machine for this rule to protect it.
6. **Nothing here debits a wallet, and nothing here switches the machine.** It
   credits only, only through `Machine_Ingest_Service`; power is a hand at the venue.
7. **The transport's cursor never advances past an event the endpoint did not
   accept, and every gap is written down.** Re-reading a window is free —
   `event_key` makes a repeat cost nothing — and skipping one is a payout a player
   never got, so a refused delivery, an unreachable machine, a missing secret or
   machine id all hold `pc_realtime_cursor_sensor_coin` where it was and record a
   `machine_poll_*` row in `wp_pc_auth_audit_log`. A lost cursor costs a bounded
   backfill and an audit row, never silence. The poller reads Home Assistant only
   through `Machine_Service` and delivers only through `POST /machine/events` — it
   is a client of that door, not a way around it.

## Interfaces
- `POST /pc/v1/machine/events` — the ingest endpoint, whatever transport calls it.
  Shipped S1.4: `type` (`coins_dropped` | `bonus` | `relay_closed`), `event_key`,
  `machine_id`, plus `coins` or `bonus_number`; answers 200 with a `status`
  (`credited` / `recorded` / `unattributed` / `already_recorded` / `failed`) for
  everything a retry cannot fix. Full shape in `docs/CONTRACTS.md`. What a transport
  may *send* is narrower than what the endpoint accepts — only `sensor.coin`
  increments have a payout behind them (`DECISIONS.md` 2026-09-18).
- **`Machine_Poller`** (S1.5) — the inbound transport, and the only caller of that
  endpoint in production. Reads `sensor.coin`'s history through
  `Machine_Service::get_state_history()` and delivers each change with
  `rest_do_request()`. Driven by the WP-Cron event `pc_realtime_poll` and by
  `wp pc machine-poll`; both are safe on one host. `--dry-run` reads and credits
  nothing, which is how it is pointed at a live machine safely.
- **The room channel's messages** (S2.2, S2.3): `queue` carries `{room_id, version}` and
  **nothing else** — the channel is readable by any signed-in account while
  `GET /rooms/{id}/queue` is play-ready gated, so a client answers a new version by
  re-reading through that gate. `credit` carries `{room_id, user_id, coins, event_id,
  at}` and no money. `relay` carries `{room_id, locked, at}` and names neither the
  sensor nor the machine. `chat` (S2.4) carries the **whole message**, and `moderation`
  an id and a state and never a body. Widening any of them is a permission decision,
  not a convenience — and the rule behind all five is one rule: **what may travel is
  what the read already gives away.** `GET /rooms/{id}/messages` is public, so a chat
  body gives nothing away; `GET /rooms/{id}/queue` is play-ready gated, so a queue
  entry may not travel at all.
- **`Realtime_Relay_Watch`** (S2.3) — one read of `sensor.relay_on` per pass of the
  poll schedule, through `Machine_Service` like every other Home Assistant call. The
  relay carries no payout signal (`DECISIONS.md` 2026-09-18): it idles **closed** and
  follows the operator's own relay buttons, so an **open** relay means the machine has
  been taken out of service by hand. A change is cached in `pc_realtime_relay_state`
  and published as `relay` `{room_id, locked, at}`; an unreadable relay publishes
  nothing, changes nothing and audits `machine_relay_read_failed`, because an
  unreadable relay is not a locked one. It runs before the coin work and outside its
  guards, so neither half can stop the other.
- `POST /pc/v1/rooms/{id}/play` is the authority on that lock, not the channel: it
  reads the relay live and answers `relay_open` 423 (S2.3 — it previously read the
  normal state as "closed" and refused every toss while the machine was on). The
  queue envelope carries `machine_locked` from the cache, so the button is right at
  first paint without a Home Assistant call on a queue read.
- `POST /pc/v1/rooms/{id}/queue/heartbeat` (S2.2) — what is left of the 3-s poll: a
  cheap write, gated exactly as the queue read is, answering a version and no state.
- `GET /pc/v1/realtime/token` — the scoped pass a signed-in SPA needs (shipped S2.1).
  It answers an Ably **token request** signed with `PC_ABLY_KEY`, never the key:
  `subscribe` on `{prefix}:room:*` for anyone signed in (rooms are public), plus
  `{prefix}:machine` for `manage_options` only, and `publish` on nothing. It names the
  resolved channels so neither SPA hardcodes them, and answers
  `realtime_not_configured` 503 where the key is absent. Full shape in
  `docs/CONTRACTS.md`.
- **One connection, several listeners** (S2.4) — `frontend/src/services/realtime.js`
  keeps one Ably client and one channel per room and fans every message out to
  consumers registered by name (`queue`, `chat`). The free tier counts *connections*,
  not subscriptions, so a second socket per room would spend the 200 ceiling twice as
  fast; and a room has both stores live at once. A listener that registers while the
  connection is up is caught up immediately.
- **Guests do not ride the channel** (S2.4). The room page and the chat read are
  public, but `GET /realtime/token` requires a signed-in caller, so a guest keeps the
  3-second chat poll. Widening the pass to anonymous callers would spend one of the 200
  concurrent connections per guest — a decision for after the ceiling has been measured.
- **The channel naming convention** (fixed S2.1, `app/realtime/channels.php` is the
  only place it is built): `{prefix}:room:{id}` for a room, `{prefix}:machine` for the
  operator channel, `{prefix}:room:*` as the capability pattern — `prefix` from
  `pc_realtime_channel_prefix`, so one Ably app can host staging and production
  without them hearing each other. **One channel per room, one for the machine, never
  one per viewer:** the free tier caps an app at 200 channels, which a per-viewer
  channel would pass at the 201st player. Neither SPA hardcodes a name — the token
  endpoint returns the resolved ones — but the admin SPA reads the same channels, so
  changing the shape is still "touches shared surface".
- It consumes, and does not change, `core`'s `pc_machine_event_player` filter and
  `pc_machine_event_credited` action.

## UI
- **Screens:** none of its own. It changes two screens `core` owns, indexed in
  `docs/DESIGN.md` → Screens: **Room** (player SPA) and **Machine** (admin SPA).
- **Reuses:** `PlaceBet`, `UserControls`, `RoomQueue`, `RoomChat`. S2.3 gives `PlaceBet`
  one more disabled state — the machine out of service — indexed in `docs/DESIGN.md`
  → Components. No new component, no token, no screen.
- **Introduces:** — (no design; `design/` stays empty — `DECISIONS.md` 2026-09-15).

## Roadmap
- Sprint 1 — a bonus on the machine credits the player holding the turn, with nobody running a command; the sensor model matches the machine (`sprints/SPRINT-1.md`)
- Sprint 2 — the room reflects coins, bonuses and the relay lock without polling (`sprints/SPRINT-2.md`)
- Sprint 3 — the operator learns about a fault from the system, not from a support ticket; a hand-switched power-off is not a fault (`sprints/SPRINT-3.md`)
- Left for later, outside these sprints: the operations dashboard (a future `ops` feature); error tracking and request logging (ROADMAP Phase 8); out-of-hours escalation (a business decision nobody has made); the admin **Machine** screen's own 3-second poll, which may ride the machine channel when Sprint 2 makes it free but is not a goal.
