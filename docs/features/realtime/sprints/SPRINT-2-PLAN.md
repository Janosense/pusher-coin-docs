# Sprint 2 — step plans (`realtime`)

<!-- Written by /plan-step, one section per step. A section's status moves
     awaiting approval → approved, in progress → implemented, awaiting close →
     closed. Never edit another step's section. -->

## Plan — Sprint 2, Step 1: Publish to Ably, and hand the SPA a token   (status: closed)

### Branch
`realtime/sprint-2-channel` ← `realtime/sprint-2` ← `main`
(git model in root `CLAUDE.md`. The sprint branch does not exist yet and is cut from
`main`, which now carries Sprint 1. This step touches `backend/` and the docs
repository only — `frontend/` gets the same branch name when Step 2 first edits it.)

### What is settled before this step starts
- **Ably, free tier** (`DECISIONS.md` 2026-09-15): the key is a wp-config constant, the SPA gets a scoped token and never the key, and the ceilings to watch are 200 concurrent connections and 200 channels.
- **Publishing is fire-and-forget** (`FEATURE.md` → Invariants #2): a failed publish is logged and swallowed and may never fail, delay or roll back the money path that triggered it.
- **No SDK.** `vendor/` is git-ignored and CI FTP-syncs the tree without `composer install` (`DECISIONS.md` 2026-09-17, the Stripe precedent), so an Ably PHP SDK cannot reach production. Ably is called over `wp_remote_*` and its token request is signed by hand — the same shape as the Stripe webhook's HMAC. **No new dependency is added in this step**, so core rule 1 is not engaged.

### Tasks (ordered)
- [x] **1. The key, the channel names and the TTL.** `PC_ABLY_KEY` documented (commented out, **no value committed**) in `wp-config-sample.php` and `wp-config-ddev.php`, next to the other secrets and with the note that it is `name:secret` and the SPA never sees it. Two option defaults in `Install_Schema::install_default_options()` — `pc_realtime_channel_prefix` (`'pc'`) and `pc_realtime_token_ttl_seconds` (3600) — and `DB_VERSION` 1.11.0 → 1.12.0, which is what makes them seed. `DATA-MODEL.md` in the same commit. → `feat(realtime): the Ably key, channel prefix and token TTL` — **touches shared code (`core`) — may affect other features** (`app/utils/install-schema.php` and both wp-config templates; additive only)
- [x] **2. The channel names, in one place.** `app/realtime/channels.php` — `Realtime_Channels::room( int $room_id )` → `{prefix}:room:{id}`, `::machine()` → `{prefix}:machine`, `::room_pattern()` → `{prefix}:room:*`. One room channel, one machine channel, never one per viewer (the free tier caps channels at 200, and a per-viewer channel would blow that at 201 players). Written into `FEATURE.md` → Interfaces as the convention both SPAs must follow. → `feat(realtime): the channel naming convention`
- [x] **3. The publisher.** `app/realtime/publisher.php` — `Realtime_Publisher::publish( string $channel, string $name, array $data ): bool`, a hand-written `wp_remote_post` to `https://rest.ably.io/channels/{channel}/messages` with HTTP Basic auth from `PC_ABLY_KEY`, a 2-second timeout, and **every failure caught, logged through `Audit_Log` and swallowed** — it returns `false` and never throws, never returns a `WP_Error` to its caller, and an unconfigured key is a silent no-op rather than an error. Hooked to `pc_machine_event_credited`, which resolves machine id → room through `Queue_Service::room_id_for_machine()` and publishes one compact `credit` message to that room's channel. **The payload carries `user_id`, `coins`, `event_id` and `at` — no money.** A room channel is readable by everyone watching the room, and another player's coin purchase price is not theirs to see; the winning player's own money totals stay behind the authenticated endpoints they already come from. → `feat(realtime): publish machine credits to the room channel` — the hook is added in `app/realtime/`, so **no shared file is edited**
- [x] **4. The token endpoint.** `app/realtime/RealtimeTokenController.php` — `GET /pc/v1/realtime/token`, `permission_callback` `Permissions::require_logged_in`. It answers a **signed Ably token request**, not a token: `{keyName, ttl, capability, clientId, timestamp, nonce, mac}`, the `mac` an HMAC-SHA256 over the canonical field order, base64-encoded, keyed with the secret half of `PC_ABLY_KEY`. Ably's `authUrl` flow expects exactly this, and it means WordPress makes **no outbound call** to mint a token — no latency and no failure mode on a player-facing request. Capability: every signed-in caller gets `subscribe` on `{prefix}:room:*` (rooms are public — `DOMAIN.md`), and **only** an administrator additionally gets `subscribe` on `{prefix}:machine`. The response also names the resolved channels, so neither SPA hardcodes them. With no key configured it answers `realtime_not_configured` 503, the shape `stripe_not_configured` already uses. → `feat(realtime): GET /realtime/token hands the SPA a scoped token request` — registered from `app/realtime/bootstrap.php`, so **no shared file is edited**
- [x] **5. The documentation of the flow.** `CONTRACTS.md` gains the endpoint in "current" and retires the planned `POST /pc/v1/realtime/auth` bullet under the name and provider that actually shipped; `ARCHITECTURE.md` gains the Ably integration row and the publish leg of the data flow. → `docs(realtime): the Ably channel, its token endpoint and the publish leg`

### Why the relay half of this step is not built here
Step 1's task list says the publisher fires "on `pc_machine_event_credited` **and on a
relay-state transition**". There is no relay-state transition to hook:

- Nothing in the codebase calls `Machine_Service::relay_close()` / `relay_open()` — the only relay consumer is `RoomQueueController.php:129`, which *reads* `get_relay_closed()` for the 423 gate.
- The Sprint 1 Step 2 spike found the machine reports none either: `sensor.relay_on` follows only the buttons the software presses, stayed `1` through all 28 rises of the coin counter, and `sensor.sw_b_t_relay` never moved in ten days (`DECISIONS.md` 2026-09-18). That entry states in as many words that Sprint 2 Step 3's relay lock has no signal to follow.
- Detecting transitions means polling `sensor.relay_on`'s history — which is **Step 3's first task**, where the relay entity and polarity are read from the spike entry, as this sprint's Fixed decisions require.

So this step builds a publisher that can carry any message on any channel, and wires the
one trigger that exists. Step 3 adds the relay trigger together with the button state it
exists for. This is Question 1 below.

### Files to create/change
**`backend/` (repository `backend`)**
- `wp-content/themes/pc/app/utils/install-schema.php` — **shared (`core`)**: two `add_option()` lines, `DB_VERSION` → `'1.12.0'`. Additive.
- `wp-config-sample.php`, `wp-config-ddev.php` — **shared (`core`)**: the documented, commented-out `PC_ABLY_KEY`. **No value committed.**
- `wp-content/themes/pc/app/realtime/channels.php` — **new**, feature-owned.
- `wp-content/themes/pc/app/realtime/publisher.php` — **new**, feature-owned.
- `wp-content/themes/pc/app/realtime/RealtimeTokenController.php` — **new**, feature-owned.
- `wp-content/themes/pc/app/realtime/bootstrap.php` — three `require_once` lines, the route registration and the `pc_machine_event_credited` hook.
- `wp-content/themes/pc/tests/realtime-channel.php` — **new**.

**Docs repository**
- `docs/CONTRACTS.md`, `docs/DATA-MODEL.md`, `docs/ARCHITECTURE.md`, `docs/features/realtime/FEATURE.md`.

**Not touched:** `frontend/`, `admin/`, `functions.php`, the poller, the ingest endpoint,
`machine-ingest-service.php`, `queue-service.php`, `RoomQueueController.php`.

### Tests to write
All in `backend/wp-content/themes/pc/tests/realtime-channel.php`, the WP-CLI `eval-file`
shape `DECISIONS.md` 2026-09-15 fixed: the WP-CLI + DDEV guard, a per-run key prefix, a
`finally` that removes every fixture, option, transient and filter it touched **and the
`wp_pc_machine_events` rows the code under test writes** (`LEARNINGS.md` 2026-09-21).
Ably is never called: `pre_http_request` answers. The token half needs no HTTP at all.
Money zone — a publish sits on the credit path — so these ship with the code.

1. **A failed publish cannot touch the credit (task 3).** With the stub answering a transport error, then HTTP 401, then HTTP 500: in each case the player's wallet still moved by exactly the credited coins, the `wp_pc_machine_events` row still reads `credited` and names the player, the session's `coins_won` still rose, and `publish()` returned `false` — with the failure recorded in the audit log. The same again with **no `PC_ABLY_KEY` configured**: the credit is identical and nothing is attempted.
2. **A publish that succeeds sends the right thing (task 3).** One POST, to `https://rest.ably.io/channels/{prefix}:room:{id}/messages`, with an `Authorization: Basic` header; the body names `credit`, the crediting player, the coin count and the event id. **It carries no unit price and no money field** — asserted by name, so a later widening is a deliberate change.
3. **A credit with no room behind its machine id publishes nothing** and still credits (the unattributed path already covered in Sprint 1 keeps working).
4. **The token endpoint refuses an anonymous caller (task 4)** — 401 through the real route, and the body carries no `keyName`, no `mac` and nothing resembling the key.
5. **The key never leaves the server (task 4).** For a signed-in player the whole serialised response is searched for the secret half of `PC_ABLY_KEY` and for the key in full; neither appears. `keyName` (the public half) and `mac` do.
6. **The signature is real (task 4).** The test recomputes the HMAC independently from the returned fields and compares — a `mac` that is merely present is not a `mac` that works.
7. **A non-admin cannot reach the machine channel (task 4).** A player's capability names `{prefix}:room:*` with `subscribe` and **no** `{prefix}:machine` entry; an administrator's names both. Asserted on the parsed capability, not on a substring.
8. **An unconfigured server (task 4)** answers `realtime_not_configured` 503 and reveals nothing.
9. **The channel names (task 2).** `room()`, `machine()` and `room_pattern()` follow the prefix option, and changing the option changes all three together.
10. **The TTL and prefix come from the options (tasks 1, 4)**, so an operator change reaches the issued token request.

Each new behaviour's checks must **fail on the pre-step code** and pass after, reverting
only the file under test so the failure is the behaviour and not a missing class
(`LEARNINGS.md` 2026-09-21); the proofs go in the execution notes.

### Docs to update
- `docs/CONTRACTS.md` — `GET /pc/v1/realtime/token` in the current section: request, the token-request body, the capability rules, `realtime_not_configured` 503 in the error registry; and the planned `POST /pc/v1/realtime/auth` bullet retired under the name and provider that shipped.
- `docs/DATA-MODEL.md` — `PC_ABLY_KEY` alongside the other wp-config secrets, the two new options with their defaults, and `pc_db_version` 1.11.0 → 1.12.0.
- `docs/ARCHITECTURE.md` — an Ably row in Integrations (what it is used for, auth model, failure/fallback, credentials) and the publish leg in the machine-event data flow.
- `docs/features/realtime/FEATURE.md` — Interfaces: the channel naming convention as the fixed contract both SPAs follow, and the token endpoint's real shape; Data: the two new options and the constant.
- At close, by `/close-step`: `docs/WORKLOG.md`, `docs/features/realtime/verification/sprint-2-step-1.md`, and `docs/DECISIONS.md` for the decisions this step makes on top of the 2026-09-15 entry (a signed token request rather than a fetched token; a money-free room payload; the capability split).

### Checks
- **ANTI-PATTERNS: none violated.** The ones close enough to name:
  - *"Do not put a secret in a WP option or in the repository."* `PC_ABLY_KEY` is a wp-config constant, documented with no value committed, never logged, and never sent to a SPA — the token request is signed *with* it and does not contain it.
  - *"Do not register a REST route without an explicit `permission_callback`,"* and do not invent a new gate inline. The token route uses `Permissions::require_logged_in`.
  - *"Do not hardcode an operator-tunable value."* The channel prefix and the token TTL are options with defaults in `Install_Schema` — and TTLs are named in that rule explicitly.
  - *No new dependency*, so core rule 1 is not engaged: Ably is called over `wp_remote_post` and the token request is signed by hand, because `vendor/` never reaches production (`DECISIONS.md` 2026-09-17).
  - Also holding: no `ENUM`, no money column touched, no ledger row, no float, no meta-key literal, and the publish never runs before the wallet move it reports.
- **Docs vs reality:**
  - **`CONTRACTS.md` still lists the channel endpoint as planned under a different name and "provider unpicked"** (`POST /pc/v1/realtime/auth`). `FEATURE.md`, this sprint and `DECISIONS.md` 2026-09-15 all say `GET /pc/v1/realtime/token` on Ably. Resolved in favour of those three; task 5 retires the bullet.
  - **Nothing produces a relay transition** — see the section above. It is Question 1, not a silent omission.
  - **The 423 relay bug is still open** (`get_relay_closed()` reads the normal `1` as "closed", so every toss is refused while the machine is on — `DECISIONS.md` 2026-09-18, carried for `/adhoc`). This step does not touch the toss path, but Step 3 cannot deliver a working button until it is fixed. Recorded, unchanged here.
  - **Guests are public viewers, but the token endpoint is signed-in only.** `DOMAIN.md` says anyone may watch a broadcast and read a room's chat; this step's own Tests require an anonymous caller to be refused. So Steps 2 and 4 must keep a non-channel path for guests, or the sprint goal "no 3-second poll remains in `frontend/src/`" will collide with guest viewers. Changes nothing here; named so those steps plan for it.
  - `FEATURE.md` → Data already reserved "channel name" as a `pc_realtime_*` option; task 1 honours that rather than hardcoding the prefix.
- **Design: n/a** — no screen, no token, no component. `FEATURE.md` → UI introduces nothing and `design/` stays empty (`DECISIONS.md` 2026-09-15). The one design-visible change of this sprint is `PlaceBet`'s disabled state, which is Step 3's.
- **Check command:** `backend/bin/check` (`docs/TECH-STACK.md` → Check command) before every commit, with DDEV up so stage 2 executes — a boxed `SKIPPED` is not a green run for this step. `frontend/bin/check` run once at the end to confirm the untouched SPA still exits 0.
- **Not locally verifiable:**
  - **A real message arriving on a real Ably channel** — the step's own manual verification watches it in Ably's dashboard, which needs an **Ably account and an app key that do not exist yet**. Creating the account and putting `PC_ABLY_KEY` in the local `wp-config.php` is a prerequisite the user supplies; the code and all ten checks are written and gated without it.
  - **`PC_ABLY_KEY` in production `wp-config.php`** — no deploy writes that file, exactly as with the ingest secret. Verified by the operator's edit after the sprint reaches `main`.

### Questions / ambiguities
1. **The relay trigger this step names does not exist yet — build it here or leave it to Step 3?**
   - **(a) Leave it to Step 3 — recommended.** This step ships a publisher that can carry any message on any channel, plus the one trigger that exists (`pc_machine_event_credited`). Step 3 adds the relay trigger alongside the `PlaceBet` state it exists for, reading the relay entity and polarity from the spike entry as the sprint's Fixed decisions require. Tasks as written above.
   - **(b) Build the relay trigger here too.** Task 3 would additionally extend `Machine_Poller` to read `sensor.relay_on`'s history and fire transitions, and the checks would grow a relay half. That duplicates Step 3's first task, imports the relay's unresolved semantics into a step whose verification is about Ably, and still produces no visible behaviour until Step 3 ships the button.
   - Recommendation **(a)**: the spike already recorded that the relay has no signal behind it, Step 3 is where that is confronted, and (b) would have this step build something no one can see working.
   - **Resolved: approved as recommended — (a).** The publisher carries any message on any channel and this step wires only `pc_machine_event_credited`; the relay trigger belongs to Step 3, with the button it exists for.

### Execution notes (Step 1)
- **Branch:** `realtime/sprint-2-channel` ← `realtime/sprint-2` (cut fresh from `main`, which carries Sprint 1) in both repositories. Backend `a6355141` `09c26513` `6215f446` `73da9d01` `071f07ef`; docs `f5972d8` `8678e90` `c0bcbb1` `3f9acc1`.
- **Question 1 resolved as recommended (a).** The publisher carries any message on any channel; only `pc_machine_event_credited` is wired. No relay trigger was built, and `Machine_Poller` was not touched.
- **Ably's token request is signed here, not fetched.** `sign()` implements Ably's canonical order (`keyName`, `ttl`, `capability`, `clientId`, `timestamp`, `nonce`, each followed by `\n`), HMAC-SHA256, base64 of the raw digest. The check recomputes it independently rather than asserting the field is non-empty.
- **Before/after proofs**, each reverting only the behaviour under test (`LEARNINGS.md` 2026-09-21):
  - a publisher that lets a transport failure escape → the run dies mid-check; in production it would have thrown *after* the wallet moved, inside a poller pass. This is the whole reason invariant #2 exists.
  - `unit_price` added to the room payload → exactly the check that forbids money on a room channel fails.
  - the key placed in the token response → both checks that hunt for it fail.
  - the machine channel granted to everyone → exactly the check that forbids it fails.
  - signing with the wrong secret → the independent signature verification fails, and nothing else does.
- **A test bug, and the same trap as last step.** `null === ( $body['channels']['machine'] ?? 'x' )` can never pass: `??` treats the very `null` being asserted as absent (`LEARNINGS.md` 2026-09-21). The endpoint was correct; the check was not. Read once, then `array_key_exists`.
- **Test hygiene, and the same lesson as last step.** The cleanup deleted `machine_id = $machine_id`, but the orphan check deliberately credits `…-nobody`, so 15 rows survived in `wp_pc_machine_events`. Now a LIKE on the run prefix; the 15 were purged from the local install. Both leftovers-checks (`rc-%` users, `Channel test%` rooms) return nothing.
- **A Sprint 1 Step 5 omission, corrected here.** Neither `ARCHITECTURE.md`'s nor `PROJECT-TREE.md`'s directory map listed `machine-poller.php`, `machine-poll-command.php` or `machine-poll.php`: that step's tree edit matched a pattern the file did not contain and silently applied nothing. Both maps were wrong from Step 5's merge until this commit. `PROJECT-TREE.md` was not in this step's approved docs list — it is edited because this step adds files to the same map (core rule 5), and both corrections are disclosed rather than folded in quietly.
- **`pc_db_version` 1.11.0 → 1.12.0** ran on the local install; `pc_realtime_channel_prefix` and `pc_realtime_token_ttl_seconds` are seeded there.
- **Not done, deliberately:** no Ably account exists, so nothing was published to a real channel and no dashboard was seen. Every check answers Ably through `pre_http_request`; the token half makes no HTTP call at all. No SPA was opened signed in.
- **Gate:** `backend/bin/check` exit 0 before every commit (62 php files; stage 2 executed, `realtime-channel.php` among them) and `frontend/bin/check` exit 0 at the end — `lint: OK`, `build: OK`.

---

## Plan — Sprint 2, Step 2: The queue subscribes instead of polling   (status: closed)

### Branch
`realtime/sprint-2-queue` ← `realtime/sprint-2`
(git model in root `CLAUDE.md`. **The first step of this sprint to touch `frontend/`**, so
the task branch is created there too, with the same name; `backend/` and the docs
repository carry it as well. `admin/` is not touched.)

### What is settled before this step starts
- **Ably, free tier; the SPA gets a scoped pass and never the key** — `DECISIONS.md` 2026-09-15 and 2026-09-21. `GET /pc/v1/realtime/token` and `{prefix}:room:{id}` shipped in S2.1.
- **Publishing is fire-and-forget** — a failed publish is logged and swallowed and may never fail, delay or roll back what triggered it (`FEATURE.md` → Invariants #2).
- **The queue is persisted and pruned on read** — `DECISIONS.md` 2026-07-30. Removing the poll does not remove the heartbeat; this step changes what the heartbeat *is*, and says so in a new entry rather than editing the old one.
- **`stores/queue.js` and `RoomQueueController.php` belong to `core`** — every task touching them is marked accordingly.

### Two things the step text assumes that the code does not yet do
Both are derived from the step's own **Verification**, which §5 makes part of the step —
neither is borrowed from another step:

1. **Nothing publishes a queue change.** S2.1 publishes only `credit`, on
   `pc_machine_event_credited`. The manual check for this step is "one joins the queue —
   the other sees it without a three-second wait", which is impossible until the server
   announces joins, leaves and turn changes. Task 1 adds that.
2. **A turn can end without anyone acting.** `prune_stale()` drops an idle player and
   `sync_turn()` promotes the next one, and today both run *because* somebody polled.
   With the poll gone, a player promoted by someone else's idleness would never hear
   about it. The heartbeat in task 2 is what keeps that working.

### The permission line this step does not cross
`GET /rooms/{id}/queue` is gated `require_play_ready` (terms + nickname + confirmed
email). The room channel's pass, by contrast, is granted to **any signed-in caller**
(S2.1). So **a queue message may not carry queue state**: doing so would let a
signed-in but not-yet-play-ready account read, off the channel, what the API refuses
them.

Therefore the queue message is a **change ping** — `{room_id, version}` and nothing
else — and every client answers it by re-reading `GET /rooms/{id}/queue`, which is
still gated exactly as it is today. Push decides *when* to read; the existing
permission still decides *what* may be read. This also keeps the payload tiny on a
free tier metered by message, and means a missed message costs one stale second rather
than a wrong queue.

### Tasks (ordered)
- [x] **1. The server announces that the queue changed.** `Realtime_Publisher::announce_queue( int $room_id )` publishes `{room_id, version}` to that room's channel, `version` being a short fingerprint of the room's current queue state (`md5` of the serialised entries + turn holder, truncated) — enough to tell "changed" from "unchanged", and carrying nothing. `RoomQueueController` calls it after a successful `join`, `leave` and `play`. → `feat(realtime): announce queue changes on the room channel` — **touches shared code (`core`) — may affect other features** (`app/rest-api/RoomQueueController.php`; three added calls, no existing behaviour altered, and the publish is fire-and-forget so a dead Ably cannot fail a join)
- [x] **2. The heartbeat becomes a cheap write.** `POST /pc/v1/rooms/{id}/queue/heartbeat`, registered alongside the other queue routes and gated the same (`require_play_ready`): it runs `prune_stale()`, `touch()` and `sync_turn()` — the three things the poll did for free — and answers `{version}` only. No queue read, no serialisation, no entries in the body. When the version differs from the one the client holds, the client does one full `GET /rooms/{id}/queue`; when it matches, nothing happens. That is what makes a promotion caused by *another* player's idleness still reach the promoted player. `CONTRACTS.md` in the same commit. → `feat(realtime): a heartbeat that writes instead of reading` — **touches shared code (`core`) — may affect other features** (`app/rest-api/RoomQueueController.php`, `app/utils/queue-service.php` gains one small public `version()` helper; existing methods unchanged)
- [x] **3. The browser's channel client.** `frontend/src/services/realtime.js` — fetch the pass from `GET /realtime/token`, connect, subscribe to the room channel, and expose `subscribe(roomId, handlers)` / `unsubscribe()` plus a `connected` flag to the stores. Reconnect with backoff, and **on every (re)connect run the catch-up** rather than assume nothing was missed. It degrades quietly: if the token endpoint answers `realtime_not_configured`, or the connection never comes up, the service reports itself unavailable and the caller keeps working without it. → `feat(realtime): the browser's channel client` — new file, `realtime` owns it
- [x] **4. The queue store subscribes, and keeps its place.** `stores/queue.js`: `startPolling()` becomes `subscribe()` (the name `RoomView` calls is kept working), the 3-second full read is gone, and in its place — a subscription that re-reads on a `queue` ping, a heartbeat timer at `pc_queue_idle_timeout_seconds / 3` (20s by default, so a place survives two lost beats), and one full `refresh()` on every connect and reconnect. **If the channel cannot be reached at all, the store falls back to the old interval** so a player without Ably is not left with a frozen room. → `feat(realtime): the queue subscribes instead of polling` — **touches shared code (`core`) — may affect other features** (`frontend/src/stores/queue.js`; `RoomView.vue` keeps calling the same entry point, `RoomQueue.vue` and `PlaceBet.vue` read the same derived state and are not edited)
- [x] **5. Winnings arrive with the credit.** On a `credit` message naming the signed-in player, the store adds the coins to the turn's winnings immediately, so `UserControls` shows them without waiting; the next `refresh()` — on the following ping, heartbeat mismatch or reconnect — is what makes it authoritative, so a missed or duplicated message self-corrects rather than drifting. `UserControls.vue` itself is **not edited**: it already renders `myWinningsCoins`. → `feat(realtime): winnings update from the pushed credit` — **touches shared code (`core`)** (`frontend/src/stores/queue.js` again)
- [x] **6. The documentation of the change.** `ARCHITECTURE.md` (the queue flow stops saying "polled every 3s"), `FEATURE.md` → Interfaces, `TECH-STACK.md` → ANTI-PATTERNS (the cron rule's *justification* changes — the heartbeat is now an explicit cheap write, not the 3s poll — while the rule itself stands), and a **new** `DECISIONS.md` entry superseding 2026-07-30 on what the heartbeat is now. → `docs(realtime): the queue rides the channel, and what the heartbeat became`

### Files to create/change
**`backend/`**
- `wp-content/themes/pc/app/rest-api/RoomQueueController.php` — **shared (`core`)**: three publish calls, one new route.
- `wp-content/themes/pc/app/utils/queue-service.php` — **shared (`core`)**: one new public `version()` helper. Nothing existing changes.
- `wp-content/themes/pc/app/realtime/publisher.php` — `announce_queue()`.
- `wp-content/themes/pc/tests/realtime-queue.php` — **new**.

**`frontend/`**
- `src/services/realtime.js` — **new**, `realtime` owns it.
- `src/stores/queue.js` — **shared (`core`)**: the poll becomes a subscription plus a heartbeat.
- `package.json` / `package-lock.json` — the channel client dependency, **subject to Question 1**.

**Docs repository** — `docs/ARCHITECTURE.md`, `docs/CONTRACTS.md`, `docs/TECH-STACK.md`, `docs/features/realtime/FEATURE.md`.

**Not touched:** `admin/`, `RoomView.vue`, `RoomQueue.vue`, `PlaceBet.vue`, `UserControls.vue`, `stores/chat.js` (Step 4), `stores/wallet.js`, the poller, the ingest endpoint, the token endpoint.

### Tests to write
**Backend** — `backend/wp-content/themes/pc/tests/realtime-queue.php`, the WP-CLI
`eval-file` shape, with the guard, a per-run prefix, and a `finally` that cleans up **by
prefix** (`LEARNINGS.md` 2026-09-21).

1. **A join, a leave and a play each announce the room** — one message per action, on that room's channel, carrying `room_id` and `version` **and no queue entries, no nicknames, no coin counts** (asserted by field name: this is the permission line above, so a later widening must be deliberate).
2. **The version changes when the queue changes and not otherwise** — two reads with nothing happening in between give the same version; a join, a leave and a turn change each give a different one.
3. **A dead Ably cannot fail a queue action.** With the stub erroring, `join`, `leave` and `play` all still succeed, return their normal envelopes, and move the same rows; the failure is in the audit log.
4. **The heartbeat writes and does not read.** It updates `last_seen_at`, prunes a stale entry, promotes the next player when the head has gone, answers `{version}` and **no entries**; and an entry that heartbeats is still there after the idle timeout while one that does not is pruned.
5. **The heartbeat is gated exactly as the queue read is** — `require_play_ready`: an anonymous caller and a signed-in caller who has not confirmed their email are both refused, with the same codes the existing queue routes answer.

**Frontend** — there is no test runner in `frontend/` and this step does not add one
(that would be a dependency and a decision of its own). The gate stays
`frontend/bin/check` (lint + build), and the store's behaviour is covered by the manual
verification below, which is written to exercise exactly the paths the backend checks
cannot reach: a real reconnect, a real promotion, and the fallback when the channel is
unavailable. **This is a known gap and is named in Checks → Not locally verifiable**,
not papered over with an invented assertion.

### Docs to update
- `docs/CONTRACTS.md` — `POST /pc/v1/rooms/{id}/queue/heartbeat` in the current section: the body it answers, its gate, and that it deliberately returns no state. Plus a line on the `queue` channel message being a ping, with the reason.
- `docs/ARCHITECTURE.md` — the queue data flow stops saying "polled every 3s"; the overview line about polling narrows to what is still polled (chat, until Step 4).
- `docs/TECH-STACK.md` → ANTI-PATTERNS — the cron rule's justification is rewritten: pruning still happens on a request, but that request is now an explicit heartbeat rather than a 3-second full read. The rule itself does not change.
- `docs/features/realtime/FEATURE.md` — Interfaces (the `queue` message and the heartbeat), and the `frontend/src/` shared-code notes that describe the 3-s poll.
- At close, by `/close-step`: `docs/DECISIONS.md` (a **new** entry superseding 2026-07-30 — what the heartbeat is now, and why the queue message carries no state), `docs/WORKLOG.md`, `docs/features/realtime/verification/sprint-2-step-2.md`.

### Checks
- **ANTI-PATTERNS: none violated.** The ones worth naming:
  - *"Do not add a cron job for queue housekeeping."* Nothing here adds a cron. Pruning still happens on a request; the request is now a heartbeat. The rule's wording is updated in task 6 because its *reason* moves, not its verdict.
  - *"Do not register a REST route without an explicit `permission_callback`."* The heartbeat reuses `Permissions::require_play_ready` — the same gate as the queue read it replaces. No new gate is invented.
  - *"Do not paginate chat with `LIMIT/OFFSET`."* Untouched — chat is Step 4.
  - **Core rule 1** is engaged by the browser channel client: Question 1.
- **Docs vs reality:**
  - **Guests were never polling the queue.** At S2.1's close I flagged that rooms are public while the token endpoint is not, and that Steps 2 and 4 would have to keep a path for guests. For *this* step that turns out to be wrong: `GET /rooms/{id}/queue` is already `require_play_ready`, and `RoomView.vue:46` starts the queue only for an authenticated, email-verified viewer. A guest has no queue poll to remove. **The concern stands for Step 4**, where chat reads are genuinely public.
  - **S2.1's `credit` payload carries `user_id` and `coins`** on a channel any signed-in caller may join, which is slightly more than the play-ready-gated API reveals to that same caller. It shipped and is out of this step's scope; the tightening — narrowing the room capability to play-ready callers, or dropping `user_id` — belongs in a later step or `/adhoc`. Recorded, no task here.
  - **The balance still lags.** `stores/wallet.js` is read on room entry and after a toss, never on a poll, so a machine credit shows as winnings at once and in the balance only after a reload or the next toss. This step does not change that and no step of this sprint names it.
  - `ARCHITECTURE.md`'s overview and queue sections say "polled every 3s" in several places; task 6 narrows them rather than deleting them, because chat still polls until Step 4.
- **Design:** unchanged. `docs/DESIGN.md` → Components lists **Room queue** (`empty, waiting, you-are-next, your-turn`) and **User controls** (`balance, per-turn winnings, theme-song toggle`); this step changes how those states *arrive*, not what they are, and edits neither component's DOM. `PlaceBet`'s new disabled state is Step 3's.
- **Check command:** `backend/bin/check` and `frontend/bin/check` before every commit; the backend one with DDEV up, so a boxed `SKIPPED` is not a green run.
- **Not locally verifiable:**
  - **Everything the browser does** — the subscription, the reconnect, the fallback and the winnings update. `frontend/` has no test runner and this step does not add one; the manual guide is the only thing that exercises them, and the agent does not sign in to the SPA. This is the step's real risk and it is stated rather than hidden.
  - **A real Ably connection** — still no account (carried from S2.1). Section 7 of S2.1's guide and the whole of this step's two-browser check wait on one.

### Questions / ambiguities
1. **How does the browser talk to Ably? This is a new dependency either way, so it needs your approval (core rule 1).**
   - **(a) The official `ably` npm package — recommended.** It does the token exchange against our `authUrl`, the reconnect with backoff, and the catch-up-on-reconnect that this step explicitly asks for. Roughly one runtime dependency in `frontend/`, and unlike the backend there is no obstacle to it reaching production — Vercel builds from `package.json`, which is exactly why `vendor/` ruled an SDK out on the server and does not here.
   - **(b) No dependency: native `EventSource` against Ably's SSE endpoint.** We hand-write the token exchange, the reconnect backoff and the catch-up. It keeps `frontend/` at six runtime dependencies, but it puts the reconnect and catch-up logic — the part this step is actually about, and the part with no automated test — in our own untested code.
   - Recommendation **(a)**: the reason the backend has no SDK does not exist here, and (b) would hand-write exactly the logic that the step's own tests cannot cover.
   - **Resolved: approved as recommended — (a).** The official `ably` package is added to `frontend/`; core rule 1 is satisfied by this approval, and `TECH-STACK.md` records it with the locked version.

### Execution notes (Step 2)
- **Branch:** `realtime/sprint-2-queue` in all three repositories; `frontend/` needed `realtime/sprint-2` cut from `main` first. Backend `31981dee` `bed83006`; frontend `a01cedd` `0af0f90`; docs `5c9d80c` `50ecffc`.
- **Question 1 resolved as recommended (a):** `ably` 2.28.0 added to `frontend/`, recorded in `TECH-STACK.md` with the locked version and the measured cost.
- **Tasks 4 and 5 became one commit.** Both change the same file and the same subscription wiring, and splitting them would have left task 4's branch with a subscription that ignored `credit` messages. Both checkboxes are ticked against `0af0f90`.
- **A real bug the checks caught.** The heartbeat pruned *before* touching, copying `state()`'s order — so a caller who had missed a couple of beats would be evicted by their own heartbeat. A backgrounded tab throttled by the browser is exactly that case. Now it touches first; `state()` is unchanged, and the comment says why the two differ.
- **A judgement call made while measuring.** Statically imported, `ably` took the main bundle from 92 kB to 151 kB gzipped. It is now a dynamic import — its own 58 kB chunk that only a room fetches, main bundle back to 92 kB. Same pattern and same reason as `hls.js`, which this codebase already lazy-loads.
- **Before/after proofs**, each reverting only the behaviour under test:
  - putting the queue in the channel message — the shortcut that saves the client a refetch — fails exactly the two checks guarding the permission line;
  - pruning before touching fails the two checks about holding a place;
  - answering the full state from the heartbeat fails the check that it carries no queue.
- **Out of scope, found and reported, not fixed:** `tests/machine-poll.php` (S1.5) and `tests/realtime-channel.php` (S2.1) delete their throwaway rooms but not the `wp_pc_room_queues` / `wp_pc_bet_sessions` rows those rooms accumulated — 25 orphaned queue rows and 57 orphaned session rows in the local install, purged by hand here. **This is the third instance of the cleanup class of bug** (`LEARNINGS.md` 2026-09-21). Both files belong to closed steps and neither is in this plan's file list, so they were left alone: the fix belongs in `/adhoc`. This step's own script cleans both tables and leaves nothing.
- **Not done, deliberately:** nothing was pointed at real Ably (still no account), and no SPA was opened signed in. Everything the browser does in this step — subscribing, reconnecting, the fallback, the winnings — is therefore unverified until the manual guide is run, exactly as the plan's Not-locally-verifiable section said.
- **Gate:** `backend/bin/check` exit 0 before every backend commit (63 php files; stage 2 executed, `realtime-queue.php` among them) and `frontend/bin/check` exit 0 before every frontend commit and at the end — `lint: OK`, `build: OK`.

---

## Plan — Sprint 2, Step 3: The relay lock the player can see   (status: closed)

### Branch
`realtime/sprint-2-relay` ← `realtime/sprint-2`
(git model in root `CLAUDE.md`. The same branch name in `backend/`, `frontend/` and the
docs repository; `admin/` is not touched — the step text says the admin **Machine**
screen has no relay control and none is added.)

### What is settled before this step starts
- **Relay semantics are whatever Sprint 1 Step 2 recorded** (`SPRINT-2.md` → Fixed decisions). That entry is `DECISIONS.md` 2026-09-18 "Spike: machine events reach WordPress by polling Home Assistant's history", and it records three facts this plan does not re-derive:
  1. the contact sensor is **`sensor.relay_on`** — `sensor.sw_b_t_relay` read `0` for ten days including while the relay was switched, so it is not the contact;
  2. **`1` means closed, `0` means open**, by the documented button names (`input_button.relay_on` = "замкнуть"), confirmed 1.1 s after each press;
  3. **its normal state is `1` (closed), and it does not move during a payout** — it stayed `1` through all 28 rises of the coin counter in ten days of history.
- **The same entry records the defect this step runs into:** "`Machine_Service::get_relay_closed()` reads the normal `1` as 'closed', so `POST /rooms/{id}/play` answers 423 `relay_closed` to **every toss while the machine is on**. That is a defect in frozen `core`, for `/adhoc`. Sprint 2 Step 3's 'relay lock' has no payout signal to follow." See **Questions** — this is Question 1 and Question 2, and the tasks below are written under the recommended answers.
- **Publishing is fire-and-forget** — `FEATURE.md` → Invariants #2. A failed publish is logged and swallowed and may never fail, delay or roll back what triggered it.
- **`PlaceBet.vue`, `stores/queue.js`, `queue-service.php` and `RoomQueueController.php` belong to `core`** — every task touching them is marked accordingly.
- **Machine power is manual** — `DECISIONS.md` 2026-09-18. Nothing here switches it, and no task presses a relay button; the *verification* may, once, on the user's explicit go-ahead, which the step text itself sanctions.

### What the step text assumes that the machine does not do
The step says "publish relay open/closed transitions … using the relay entity and polarity
Sprint 1 Step 2 recorded" and "`PlaceBet` disables the toss control while the relay is
**closed**". Applied literally to the recorded polarity, the button is disabled whenever
`sensor.relay_on` reads `1` — which is every moment the machine is on. That is not a
reading of the step anyone intended; it is the same inversion the server already has, which
is why the server refuses every toss today. `DECISIONS.md` beats the sprint text
(`/plan-step` §5 precedence), so this plan takes "the relay lock" to mean **the state that
blocks a toss**, which by the recorded polarity is the relay being **open** (`0`), and says
so in every artefact it touches. Question 1 is where that reading can be refused.

### Tasks (ordered)
- [x] **1. The server stops refusing every toss.** `RoomQueueController::play()` refuses when the relay reads **open**, not closed: `if ( ! $relay_closed )`, error code **`relay_open`** (423) with the message "The machine is out of service. Try again shortly." `Machine_Service` is not touched — `get_relay_closed()` reports the contact correctly and the admin snapshot stays truthful; the defect is the controller's reading of it. Docblock and `docs/CONTRACTS.md` corrected in the same commit (the play order-of-operations, the error registry, and the "mid-payout" wording that the spike disproved). **Touches shared code — may affect other features** (`core`: `app/rest-api/RoomQueueController.php`; the only consumer of the error code is `frontend/src/components/PlaceBet.vue`, task 5). → `fix(realtime): the toss lock refuses an open relay, not a closed one`
- [x] **2. The relay is watched, and a change is announced.** New `app/realtime/relay-watch.php` (`Realtime_Relay_Watch`): one live read of `pc_machine_relay_sensor_entity` per poll pass through `Machine_Service::get_relay_closed()`, compared with the last known value cached in the `pc_realtime_relay_state` option; on a change, `Realtime_Publisher::announce_relay( $room_id, bool $locked )` publishes `relay` `{room_id, locked, at}` to that room's channel. The room comes from `Queue_Service::room_id_for_machine( Machine_Poller::machine_id() )`, exactly as `announce_credit()` resolves it; no room claiming the machine means nothing to tell. Called from `Machine_Poller::run()` **before** the coin work and outside its guards, so a missing ingest secret or a failed history read cannot stop it, and its own failure cannot stop them; `--dry-run` reads and reports but neither caches nor publishes. A read failure is audited (`machine_relay_read_failed`) and **leaves the cached state alone** — an unreadable relay is not a locked one, because the 423 is the authority. → `feat(realtime): watch the relay and announce a change to the room`
- [x] **3. The room knows the state before anything is published.** `Queue_Service::state()` carries `machine_locked` (bool) read from the cached option — no Home Assistant call on a queue read, ever. This is what makes the button correct at first paint rather than only after the next transition. **Touches shared code — may affect other features** (`core`: `app/utils/queue-service.php`; consumers are `GET /rooms/{id}/queue`, join, leave and play, all of which return this envelope). `docs/CONTRACTS.md` queue envelope updated in the same commit. → `feat(realtime): the queue envelope carries the machine lock`
- [x] **4. The browser listens for it.** `frontend/src/services/realtime.js` subscribes to `relay` and exposes `onRelay`; `frontend/src/services/queueService.js` maps `machine_locked`; `frontend/src/stores/queue.js` gains a `machineLocked` ref set from every envelope, from a pushed `relay` message, and — because the server is the authority — set to `true` when `play()` is refused with `relay_open` and to `false` on a successful toss. **Touches shared code — may affect other features** (`core`: `stores/queue.js`, `services/queueService.js`). → `feat(realtime): the room follows the relay lock`
- [x] **5. The button greys out, with the reason.** `frontend/src/components/PlaceBet.vue`: the toss button is disabled while `machineLocked`, with a visible line under it ("The machine is out of service — you can't toss right now."), and `PLAY_ERRORS` gains `relay_open` and drops `relay_closed`. The join and leave controls are untouched: a player may still queue for a machine that is being serviced. `docs/DESIGN.md` → Components (the `Place bet` row's state list) and `docs/ROADMAP.md` Phase 5 §5 → `[done]` in the same commit. **Touches shared code — may affect other features** (`core`: `components/PlaceBet.vue`). → `feat(realtime): the toss button shows the relay lock`

### Execution notes
- **Tasks 1–5 ran as written**, one commit each in `backend/` or `frontend/` plus its docs commit. One thing was added inside task 2's scope rather than beyond it: the relay watch's outcome also goes into `pc_realtime_poll_last_run`, because the plan promised `wp pc machine-poll --status` would show it and `--status` reads that option rather than a live pass.
- **The bug the checks name.** Ten of the first sixteen checks fail on the pre-step code, including "a closed relay is the machine working — the toss goes through". That is the production defect `DECISIONS.md` 2026-09-18 recorded: every toss refused with 423 while the machine is on.
- **Found while writing the tests, and fixed in place:** two audit-log checks first counted rows in the whole table, so they would have passed on debris from an earlier run. They now count only rows above a floor taken at start-up, and the cleanup deletes exactly those.
- **Out of scope, found and reported, not fixed:** the local install still carries orphaned rows from earlier steps' scripts (90 wallets, 276 coin lots, 2 queue rows, 4 sessions) — the `/adhoc` item `LEARNINGS.md` 2026-09-21 already names. This step's own script was measured before and after a run and leaves nothing: every table count was identical. `PROJECT-TREE.md` was also missing `realtime-queue.php` from S2.2; one line, added with this step's own.
- **Gate:** `backend/bin/check` exit 0 before every backend commit (65 php files; `realtime-relay.php` in the executed list) and `frontend/bin/check` exit 0 before every frontend commit — `lint: OK`, `build: OK`.

### Files to create/change
**`backend/` (theme `pc`)**
- `app/realtime/relay-watch.php` — **new**, `Realtime_Relay_Watch`.
- `app/realtime/publisher.php` — `announce_relay()`.
- `app/realtime/machine-poller.php` — one call into the watch at the top of `run()`, its outcome in the returned array under `relay` (so `wp pc machine-poll --status` and `--dry-run` show it).
- `app/rest-api/RoomQueueController.php` — **shared `core`**: the inverted refusal, the new code, the docblock.
- `app/utils/queue-service.php` — **shared `core`**: `machine_locked` in `state()`.
- `tests/realtime-relay.php` — **new**.

**`frontend/`**
- `src/services/realtime.js` — the `relay` subscription.
- `src/services/queueService.js` — **shared `core`**: `machine_locked` in `mapState`.
- `src/stores/queue.js` — **shared `core`**: `machineLocked`, and the 423 as the authority.
- `src/components/PlaceBet.vue` — **shared `core`**: the disabled state and its reason.

**No new dependency**, so core rule 1 is not engaged. `admin/` and `wp-config*` are untouched. No table, no column, no `pc_db_version` bump: `pc_realtime_relay_state` is written at runtime and never seeded, exactly as `pc_realtime_poll_last_run` is (`FEATURE.md` → Data), so `Install_Schema` does not change.

### Tests to write
`backend/wp-content/themes/pc/tests/realtime-relay.php`, run by `backend/bin/check` under DDEV, with Home Assistant stubbed through `pre_http_request` and Ably stubbed the same way (the pattern `tests/realtime-channel.php` and `tests/realtime-queue.php` already use). Each check that describes new behaviour must fail against the current code:

1. **The lock, both ways.** `sensor.relay_on` = `1` → `play()` does **not** refuse (this check fails on today's code, which is the bug); `0` → `play()` refuses `relay_open` 423 **before any debit**, and the caller's balance and coin lots are unchanged.
2. **The 423 in the gap** (the step's own test). Cached state says unlocked while the live relay reads open: the toss is still refused 423 and no coin is lost — the cache is a courtesy, the read is the gate.
3. **A refusal costs nothing.** After a refused toss the wallet, the coin lots, the queue row's `coins_remaining` and `wp_pc_machine_events` are all exactly as before.
4. **The watch publishes on a change and only on a change.** `1` → `0` publishes exactly one `relay` message with `locked: true`; a second pass at `0` publishes nothing; `0` → `1` publishes `locked: false`.
5. **The payload is `{room_id, locked, at}` and nothing else** — no entity id, no sensor value, no machine id.
6. **A dead Ably changes nothing.** A publish that fails leaves the cached state updated, the pass's result unchanged and a `realtime_publish_failed` audit row behind (fire-and-forget, `FEATURE.md` → Invariants #2).
7. **An unreadable relay is not a locked one.** `get_relay_closed()` returning a `WP_Error` leaves the cached state untouched, publishes nothing, audits `machine_relay_read_failed`, and does not stop the coin pass.
8. **The coin pass and the relay watch do not block each other.** A missing ingest secret stops the coin pass (`stopped: unconfigured`) and the relay is still watched; a failed history read likewise.
9. **`--dry-run` neither caches nor publishes.**
10. **No room, nothing to tell.** A machine id no available room carries: no publish, no error, cached state still updated.
11. **The envelope carries the lock.** `GET /rooms/{id}/queue`, join, leave and play all return `machine_locked`, read from the option with **no HTTP call to Home Assistant** (asserted by counting `pre_http_request` hits).
12. **Cleanup.** The script deletes every row it created — rooms, `wp_pc_room_queues`, `wp_pc_bet_sessions`, `wp_pc_machine_events`, wallet/coin-lot rows and audit rows — and restores `pc_realtime_relay_state` and every option it touched (`LEARNINGS.md` 2026-09-21, three instances of this class).

**Test-critical zones touched:** the toss path is a money zone (`CLAUDE.md` → Test-critical zones: machine-event idempotency and crediting; every `Permissions::*` callback is unchanged here). Checks 1–3 are the money checks and ship in task 1 with the code.

**The browser half has no automated coverage.** `frontend/` has no test runner and this step does not add one, so tasks 4 and 5 are covered only by the manual guide — the same gap Step 2 closed with words rather than tests, restated here so the close report does not discover it.

### Docs to update
- `docs/ROADMAP.md` — Phase 5 §5 `[partial]` → `[done]`, and its "while the machine is mid-payout" rewritten: the spike disproved it. The tracking matrix row with it.
- `docs/DESIGN.md` → Components — the `Place bet` row's states: `relay closed` becomes `machine out of service`.
- `docs/CONTRACTS.md` — `POST /rooms/{id}/play` order of operations and error registry (`relay_closed` 423 → `relay_open` 423, with what changed and why); the queue envelope's `machine_locked`; a `relay` row in **Room channel messages**.
- `docs/ARCHITECTURE.md` — the poll pass now also watches the relay and publishes it; the outbound leg.
- `docs/DATA-MODEL.md` — the `pc_realtime_relay_state` option (runtime-written, not seeded).
- `docs/features/realtime/FEATURE.md` — Interfaces (the `relay` message and the watch), Data (the new option), and the shared-code note on `RoomQueueController.php` / `machine-service.php`.
- `docs/DECISIONS.md` — a new entry recording what the relay lock means now. The 2026-09-18 spike entry said the old meaning was wrong and left the replacement open; this states it, and does not edit that entry.
- `docs/features/realtime/verification/sprint-2-step-3.md` — written by `/close-step`.

### Checks
- **ANTI-PATTERNS:** none violated. Home Assistant is called only through `Machine_Service` (the watch calls `get_relay_closed()`); no operator-tunable value is hardcoded (the entity is `pc_machine_relay_sensor_entity`, the pass interval is `pc_realtime_poll_interval_seconds`); no cron job is added for queue housekeeping (the watch rides the poll schedule that already exists); no secret moves; no route is added at all, so no `permission_callback` question arises; no money column is touched outside `Wallet_Service`; no `ENUM`, no literal meta key.
- **Docs vs reality:** **four mismatches, all corrected here.** (1) `CONTRACTS.md:612` and `ROADMAP.md` §5 both say the 423 fires "while the machine is mid-payout" — the spike showed the relay does not move during a payout at all. (2) `ROADMAP.md` §5 calls the server half `[done]`; it is inverted and refuses every toss. (3) `SPRINT-2.md`'s goal sentence and Step 3's task 2 say the button greys out "because the room knows the relay is closed" — closed is the normal state; see *What the step text assumes* above. (4) `DESIGN.md` → Components already lists `relay closed` as a `Place bet` state, so the design record expects this button state; only its wording changes. **Not corrected, reported only:** `admin/src/views/MachineView.vue:156,160,168` still promises push "from Phase 5 Step 4" and labels `sensor.relay_on` "Relay closed" — the label stays true as a fact display, no step edits `admin/`, and the stale promise wants `/adhoc` (already carried in `FEATURE.md` → Fit into the host).
- **Design:** matches `Place bet` in `docs/DESIGN.md` → Components — the row already carries a disabled-on-relay state. The wording of the state changes with the meaning; no token, no new component, no artboard (this feature has no `design/`, `DECISIONS.md` 2026-09-15).
- **Check command:** `backend/bin/check` and `frontend/bin/check` (`docs/TECH-STACK.md` → Check command). Both exist and exit 0 on the base.
- **Not locally verifiable:**
  - **A real relay transition on the machine.** Local checks stub Home Assistant. Verified by one announced `input_button.relay_off` press, on the user's explicit go-ahead, with `input_button.relay_on` immediately after to restore service — the step text sanctions exactly this. Nothing in the shipped code ever presses either button.
  - **Whether a toss with the relay open would physically fail.** Nobody has observed it; the lock is a precaution, not a measurement (see Question 1).
  - **Anything on a real Ably channel** — there is still no Ably account, so the two-browser half of the check waits on one, exactly as Steps 1 and 2 left it.
  - **The 60-second worst case.** The button greys out up to one poll interval (`pc_realtime_poll_interval_seconds`, 60) plus Home Assistant's 1.1 s after the relay opens. Inside that window the player's own toss is refused 423 and the button greys out from that. Only a real transition times it.

### Questions / ambiguities

**Question 1 — What does the relay lock mean, now that it is not a payout?**
**Resolved: approved as recommended — (a) the open relay is the lock.** `play()` refuses `relay_open` 423 while `sensor.relay_on` reads `0`; the transition is published and the button greys out with the reason.
`DECISIONS.md` 2026-09-18 settles the entity and the polarity and states plainly that the
relay carries no payout signal, but it does not say what should replace the rule it
disproved. Nobody has observed what happens to a toss while the relay is open, so this is a
product decision, not a measurement.
- **(a) — recommended. The open relay is the lock.** The relay's normal state is closed; an
  operator opening it takes the machine out of service. `play()` refuses `relay_open` 423,
  the transition is published, the button greys out with the reason. *This is what the tasks
  above are written to.* It keeps a real interlock against the failure mode that actually
  costs a player money — the coin is debited, Home Assistant answers 200 to the button press,
  and nothing happens physically — and it uses only the entity and polarity the spike
  recorded. Its weakness is honest: that an open relay would swallow a toss is an inference.
- **(b) The lock goes away.** No payout signal, no evidence a toss depends on the relay:
  delete the pre-check from `play()`, retire `relay_closed` from `CONTRACTS.md` and the
  `Place bet` state from `DESIGN.md`, mark ROADMAP §5 withdrawn rather than done. Tasks 1
  and 3–5 shrink to deletions, task 2 disappears, and the step ships no button at all. It
  also removes one Home Assistant round-trip (and one 2-second timeout) from every toss.
  Choose this if you know the machine takes a coin regardless of the relay.

**Question 2 — Does the server correction land in this step, or in `/adhoc` first?**
**Resolved: approved as recommended — (a) here, as task 1.**
The 2026-09-18 entry routed it to `/adhoc`, and three WORKLOG entries have carried it as
`/adhoc`-wanted. It has not been run, and Step 3 cannot deliver a button that means anything
until it is.
- **(a) — recommended. Here, as task 1.** The button is a mirror of the server rule, and
  mirroring a broken rule ships a permanently disabled button; the sprint's own goal names
  the 423 as "the authority", so making it correct is this step's subject, not a hotfix
  beside it. The `/adhoc` routing was written when Step 3 looked empty ("no payout signal to
  follow") — answer 1(a) gives it a signal, which is the premise that changed. Splitting it
  would also mean two sessions editing the same three lines with this step's own money
  checks unable to run in between.
- **(b) `/adhoc` first, then re-run `/plan-step realtime 2 3`.** One extra round trip, and it
  buys one thing: the fix can reach `main` on its own instead of waiting for Steps 3, 4 and 5
  to close. Choose this if you want the toss unblocked on the host before the sprint merges.

