# SPRINT 1 — Events arrive and pay the right player (2026-09-18 → open)

<!-- playbook: v1.21. Written by discovery (Phase C) together with every
     other sprint of the plan — never by Claude Code. Rewritten by a
     re-planning chat only while no step is closed; afterwards steps may
     only be appended. This file
     describes the sprint and nothing else: goal, fixed decisions, steps.
     The only thing written here during the sprint is the tick in a step's
     heading, by /close-step. -->
**Branch:** `realtime/sprint-1` ← `main`, task branches `realtime/sprint-1-{short-name}` merged `--no-ff`. The same branch name in every repository a step touches; they merge together at the sprint boundary. The earlier `realtime/sprint-1` and `realtime/sprint-1-spike-transport` branches are abandoned (`DECISIONS.md` 2026-09-18) — start fresh from `main`.
**Goal:** The machine is on, as it always is now, and somebody at the venue is playing it. A bonus lands, and the player who holds the turn sees coins on their balance — with nobody running `wp pc machine-ingest`, and with the same event arriving twice crediting exactly once. Two rooms can no longer claim one machine, so "who held the turn" has a single answer. Behind that, one transport carries events from Home Assistant into WordPress, chosen by a spike whose `DECISIONS.md` entry names it, names its `event_key` formula, and states the latency measured; and the code's model of the sensors — what the coin counter counts, which relay entity is the contact, which way its polarity goes — matches what the machine actually reports, not what the documentation guessed. Every commit passes `backend/bin/check` and `frontend/bin/check` from `main`.

## Fixed decisions
- **Machine power is manual and the machine is assumed on** — `DECISIONS.md` 2026-09-18 "Machine power is a manual operation". No step, test or verification may switch it or depend on `turn_on` / `turn_off`.
- **Session A facts are established** — `DECISIONS.md` 2026-09-18 "What the abandoned spike established is kept as fact". Step 2 does not re-verify the token, the socket, admin rights or the idle values.
- **The inbound transport is not chosen yet; Step 2 chooses it** — `DECISIONS.md` 2026-09-15 "How machine events reach WordPress is deferred to a timeboxed spike". Step 5 builds what that entry names; a build that contradicts it stops and raises it.
- **The browser channel is Ably** — `DECISIONS.md` 2026-09-15. Not used here; do not re-litigate it while building the ingest side.
- **`realtime` writes `wp_pc_machine_events` only through `Machine_Ingest_Service`** — `FEATURE.md` → Invariants. Never a direct insert.
- **Machine payouts credit coin lots, never the ledger** — `DECISIONS.md` 2026-07-24. A payout in the player's history is a bug.
- **`event_key` is the idempotency guard and every transport supplies one** — `DECISIONS.md` 2026-07-24.
- **`core` stays frozen** — `DECISIONS.md` 2026-09-15 "Realtime machine events become the feature `realtime`". Contradictions between the code and the machine that Step 2 confirms are settled inside this feature's steps or through `/adhoc`, never by editing `core`'s docs as if they had always said so.

## Steps

### [x] Step 1 — Delta-audit
- **Tasks:**
  - Read `docs/features/core/FEATURE.md` and `docs/features/stripe/FEATURE.md` (Interfaces, Invariants) and the shared code this feature will touch: `app/utils/machine-ingest-service.php`, `machine-events.php`, `queue-service.php`, `machine-service.php`; on the SPA side `stores/queue.js`, `stores/chat.js`, `components/PlaceBet.vue`. Write the touchpoints into `FEATURE.md` → Fit into the host → Shared code it depends on, and list any conflict with either sibling's invariants.
  - Confirm `app/realtime/` will register exactly the way `app/stripe/` does — one `require_once … bootstrap.php` line in `functions.php` — and note it in `FEATURE.md` → Entry point.
  - Confirm the check commands on `main` cover what these three sprints touch: `backend/bin/check` and `frontend/bin/check` exist; `admin/` has no script, and no step of Sprints 1–3 edits `admin/`. Record that in `FEATURE.md` if the audit finds otherwise.
  - Read `docs/BACKEND-REVIEW.md` §10, §12 and §15 and note which step of this feature settles each.
- **Tests:** —
- **Verification (manual):** Read the updated `FEATURE.md`: the shared-code list names concrete files, and each carries a one-line "why it matters here". `backend/bin/check` and `frontend/bin/check` exit 0 on the untouched sprint branch.
- **Docs to update:** `docs/features/realtime/FEATURE.md` → Fit into the host.
- **Depends on:** —

### [x] Step 2 — Spike: how machine events leave Home Assistant
- **Tasks:**
  - Spike code is throwaway and is not merged into `main`. Timebox: one working session at the desk plus one venue day of passive logging.
  - **Desk, first:** sample `GET /api/states/` for `sensor.coin`, `sensor.lc01_12`, `sensor.relay_on` and `sensor.sw_b_t_relay` twice, 60 s apart, while nothing happens, and compare `last_reported` with `last_changed`. If `last_reported` moves while `state` does not, the integration re-reports unchanged values and a repeated bonus number is observable through `last_reported` or `context.id`; if it does not move, a repeat is invisible to every transport and Step 4 must not price payouts from the bonus number alone.
  - **Venue hours, passive:** the machine is on and loaded because the venue runs it; nobody is asked to be there for us. Leave a socket subscription (`state_changed`, all four sensors) and a periodic REST sample running into timestamped logs for a full opening day. Fire `input_button.toss_a_coin` through the API only on the user's explicit go-ahead, announced before each press.
  - From the logs: what a real bonus looks like on `sensor.lc01_12` and how long after the coins fall it appears; what `sensor.coin` does across one payout (it resets — confirm what it counts and when it resets); which of `sensor.relay_on` / `sensor.sw_b_t_relay` is the contact and which value means "closed"; the end-to-end latency of each.
  - Write one `docs/DECISIONS.md` entry naming the transport (HA automation → webhook is available — admin rights are confirmed — but is not pre-chosen), the exact `event_key` formula built from data the logs actually contain, the latency measured, and the corrected sensor model (coin counter semantics, relay entity, relay polarity). If a full day shows no bonus, the entry says so as a recorded unknown and names what the ingest may and may not price from.
- **Tests:** —
- **Verification (manual):** Read the `DECISIONS.md` entry. It names exactly one transport, gives an `event_key` formula computable from the logs, states one latency number, and answers three questions from observation rather than inference: does a repeated bonus number leave a trace, what does `sensor.coin` count, which relay entity and value mean "closed". "Probably" anywhere means the step is not done.
- **Docs to update:** `docs/DECISIONS.md` (the transport entry); `docs/PUSHER-COIN-COMMANDS.txt` where the machine contradicts it (the relay entity at least); `docs/ROADMAP.md` Phase 5 §6 — the walk-through it wanted is replaced by this observation.
- **Depends on:** —

### [x] Step 3 — One machine, one active room
- **Tasks:**
  - `AdminRoomController` refuses to set a room `available` when another non-trashed room carries the same `pc_room_machine_id` and is itself available. New error code `machine_already_in_use`.
  - `Queue_Service` refuses a join for a room whose machine is claimed by another available room, so existing bad data cannot start a second queue before the operator fixes it.
  - A `wp pc machine-rooms` report listing every machine id held by more than one room.
  - Register the error code in `docs/CONTRACTS.md`. This settles `BACKEND-REVIEW.md` §12.
- **Tests:** Two rooms with one machine id — the second cannot be made available. An already-available pair is reported by the CLI and refuses joins rather than silently breaking. A room with an empty machine id is unaffected.
- **Verification (manual):** In the admin SPA's **Room list**, create a second room with the same machine id as a live one and try to make it available — it refuses with a message that names the conflict. The existing room keeps working throughout.
- **Docs to update:** `docs/CONTRACTS.md` (error code); `docs/features/realtime/FEATURE.md` → Invariants; `docs/BACKEND-REVIEW.md` (§12 settled).
- **Depends on:** Step 1

### [x] Step 4 — The ingest endpoint
- **Tasks:**
  - `POST /pc/v1/machine/events` in `app/realtime/`, registered from `app/realtime/bootstrap.php`. Shared-secret authentication with a constant-time compare, rate-limited, every call audited. The secret is a wp-config constant.
  - Dispatch to `Machine_Ingest_Service::ingest_bonus` / `ingest_relay_closed` / `ingest_coins_dropped`. `event_key` is required; a request without one is rejected.
  - A replayed `event_key` answers 200 with an "already recorded" body and credits nothing. An unattributed event logs and answers 200, so the transport does not retry it forever.
  - Reconcile the ingest with Step 2's sensor model: `ingest_coins_dropped` and `get_coin_count()` assume a cumulative counter, and Session A showed a counter that resets per payout. The step plan lists this as Docs vs reality and settles it here — the endpoint credits from what the machine reports, not from a delta the machine never produces.
  - Any new option gets its default in `Install_Schema`.
- **Tests:** Money zone, so tests ship in this step. Replay of one `event_key` credits once. A missing key is refused. A wrong secret is 401 and reveals nothing. A bonus credits exactly the mapped coin count at the FIFO-head unit price. A coins-dropped event credits the count reported, once. An unattributed event moves no wallet and writes no ledger row.
- **Verification (manual):** With a player holding the turn in a room, POST a bonus event with `curl`. The player's coin balance rises by the number the bonus map gives, and their **History** screen shows no new transaction. POST the identical body again — nothing changes anywhere.
- **Docs to update:** `docs/CONTRACTS.md` (endpoint + error codes, in "current"); `docs/DATA-MODEL.md` (options, constants, and the corrected meaning of `wp_pc_machine_events.coins_credited` if it changed); `docs/ARCHITECTURE.md` (the ingest data flow, which currently says nothing pushes events in).
- **Depends on:** Step 2

### [x] Step 5 — The transport
- **Tasks:**
  - Build the transport the Step 2 `DECISIONS.md` entry names. The task list differs per option, so `/plan-step` produces it after Step 2 closes — writing it now would be guessing:
    - *HA automation → webhook:* the automation committed as documentation, the shared secret on both sides, `event_key` built Home-Assistant-side from `context.id` or `last_reported`.
    - *Cron poller:* a `wp pc` command plus a real cron entry, a cursor transient, and the `last_reported`-derived key.
    - *WebSocket worker:* a small always-on service, its deploy target, and a reconnect/backoff policy.
  - Common to every option: it survives its own restart without replaying or losing events, and it fires at most one ingest call per real machine event.
- **Tests:** A restart mid-stream credits nothing twice. A dropped connection or a missed poll is either recovered or recorded as an explicit gap — never silently lost.
- **Verification (manual):** During venue hours, a real player takes a turn and the machine pays out. That player's balance moves within the latency the Step 2 entry promised, visible on the **Room** screen, with nobody running a command. The ingest secret must exist in production `wp-config.php` before the sprint merges, or the first real event is lost — the next deploy from `main` is what proves the transport in production.
- **Docs to update:** `docs/ARCHITECTURE.md` (Integrations and the data flow); `docs/TECH-STACK.md` if a dependency or a deploy target is added; `docs/DECISIONS.md` if the build contradicted the spike; `docs/ROADMAP.md` Phase 5 §3 (crediting) and §7.
- **Depends on:** Step 2, Step 4
