# Sprint 2 — step plans (`realtime`)

<!-- Written by /plan-step, one section per step. A section's status moves
     awaiting approval → approved, in progress → implemented, awaiting close →
     closed. Never edit another step's section. -->

## Plan — Sprint 2, Step 1: Publish to Ably, and hand the SPA a token   (status: approved, in progress)

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
