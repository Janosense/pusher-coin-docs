# SPRINT 1 — Events arrive and pay the right player (2026-09-15 → open)

**Branch:** `realtime/sprint-1` ← `main`, task branches `realtime/sprint-1-{short-name}` merged `--no-ff`. The same branch name is used in every repository a step touches; they merge together at the sprint boundary.
**Goal:** A bonus on the physical machine credits the player who holds the turn, by itself, with nobody running `wp pc machine-ingest`. The same event arriving twice credits once. Two rooms can no longer claim one machine, so "who held the turn" has a single answer. The project has a real check command for the first time, and every commit of this sprint goes through it.

## Fixed decisions
- The browser channel is **Ably**, free tier — `DECISIONS.md` 2026-09-15 "The browser channel is Ably, on its free tier". Not used in this sprint, but do not re-litigate it while building the ingest side.
- The inbound transport is **not chosen yet** and is decided by Step 2 — `DECISIONS.md` 2026-09-15 "How machine events reach WordPress is deferred to a timeboxed spike". Step 5 implements what that spike's entry names. If the build contradicts the spike, stop and raise it.
- `realtime` writes `wp_pc_machine_events` only through `Machine_Ingest_Service` — `FEATURE.md` → Invariants. Never a direct insert, however convenient.
- Machine payouts credit coin lots and never the ledger — `DECISIONS.md` 2026-07-24. A payout that shows up in the player's history is a bug.
- `event_key` is the idempotency guard and a transport must supply one — `DECISIONS.md` 2026-07-24.

## Steps

### [ ] Step 1 — Delta-audit and the check command
- **Tasks:**
  - Read `docs/features/core/FEATURE.md` (Interfaces, Invariants) and the shared code this feature will touch: `app/utils/machine-ingest-service.php`, `machine-events.php`, `queue-service.php`, `machine-service.php`; on the SPA side `stores/queue.js`, `stores/chat.js`, `components/PlaceBet.vue`. Write the touchpoints and any conflict with `core`'s invariants into `FEATURE.md` → Shared code it depends on.
  - Create `backend/bin/check`: `php -l` over every theme file, then the WP-CLI eval scripts in `wp-content/themes/pc/tests/` when a DDEV database is reachable (skip with a loud notice when it is not). Exit non-zero on the first failure.
  - Create `frontend/bin/check`: lint, then build. Exit non-zero on the first failure.
  - Split the SPA `lint` script: `lint` reports, `lint:fix` mutates. The current script runs `--fix`, and a gate that edits the tree it is judging is not a gate. Same change in `admin/` for symmetry.
  - Replace the "**There is none.**" section of `docs/TECH-STACK.md` → Check command with the two commands, and update `CLAUDE.md` → Commands.
- **Tests:** No new tests of its own — this step builds the harness that runs them. `backend/bin/check` must actually execute `tests/wallet-rollback.php` and surface its 53 checks rather than skipping silently.
- **Verification (manual):** Run `backend/bin/check` and `frontend/bin/check` on a clean tree — both print a summary and exit 0. Introduce a deliberate PHP syntax error, rerun the backend one — it exits non-zero and names the file. Revert. Run `npm run lint` in `frontend/` and confirm `git status` is clean afterwards.
- **Docs to update:** `docs/TECH-STACK.md` → Check command; `CLAUDE.md` → Commands; `docs/features/realtime/FEATURE.md` → Fit into the host, if the audit found a touchpoint not already listed.
- **Depends on:** —

### [ ] Step 2 — Spike: how machine events leave Home Assistant
- **Tasks:**
  - Timeboxed to one working session. Spike code is throwaway and is not merged into `main`.
  - Against the live machine, `GET /api/states/` for `sensor.coin`, `sensor.lc01_12` and `sensor.relay_on`; record the complete JSON. Confirm whether `last_changed` and `last_updated` are present, and whether they move independently of `state`.
  - Try the long-lived token against `/api/websocket`: `auth` handshake, then `subscribe_events` with `event_type: state_changed`. Record whether the token is accepted and what a `state_changed` payload for these entities contains.
  - Establish whether we can add an automation in that Home Assistant instance — that is, whether we have admin access, not just a token.
  - Observe real behaviour: one toss, one bonus, one relay close. Record what each sensor does, in what order, and with what delay. **The decisive question: when the same bonus number comes up twice in a row, does anything observable change?** If not, polling cannot see the second bonus and cannot be used for money.
  - Write the result as a `docs/DECISIONS.md` entry naming one transport, the exact `event_key` formula it will use, and the latency measured.
- **Tests:** —
- **Verification (manual):** Read the new `DECISIONS.md` entry. It names exactly one transport, gives an `event_key` formula that could be computed from data the spike actually saw, and answers the "same bonus twice" question with a recorded observation rather than a guess. If the entry says "probably", the step is not done.
- **Docs to update:** `docs/DECISIONS.md` (the transport entry); `docs/PUSHER-COIN-COMMANDS.txt` if the machine behaves differently from what it documents.
- **Depends on:** —

### [ ] Step 3 — One machine, one active room
- **Tasks:**
  - `AdminRoomController` refuses to set a room `available` when another non-trashed room carries the same `pc_room_machine_id` and is itself available. New error code `machine_already_in_use`.
  - `Queue_Service` refuses a join for a room whose machine is claimed by another available room, so existing bad data cannot start a second queue before the operator fixes it.
  - A `wp pc machine-rooms` report listing every machine id held by more than one room, so the operator can clean up before the rule bites.
  - Register the error code in `docs/CONTRACTS.md`.
- **Tests:** Two rooms with one machine id — the second cannot be made available. An already-available pair is reported by the CLI and refuses joins rather than silently breaking. A room with an empty machine id is unaffected.
- **Verification (manual):** In the admin SPA's **Room list**, create a second room with the same machine id as a live one and try to make it available — it refuses with a message that names the conflict. The existing room keeps working throughout.
- **Docs to update:** `docs/CONTRACTS.md` (error code); `docs/features/realtime/FEATURE.md` → Invariants; `docs/DOMAIN.md` already states the rule — check the wording still matches.
- **Depends on:** Step 1

### [ ] Step 4 — The ingest endpoint
- **Tasks:**
  - `POST /pc/v1/machine/events` in `app/realtime/`, bootstrapped by the single `require_once` line in `functions.php`. Shared-secret authentication with a constant-time compare, rate-limited, every call audited.
  - Dispatch the payload to `Machine_Ingest_Service::ingest_bonus` / `ingest_relay_closed` / `ingest_coins_dropped`. `event_key` is required; a request without one is rejected, not accepted-and-logged.
  - A replayed `event_key` answers 200 with an "already recorded" body and credits nothing.
  - An unattributed event (nobody holds the turn) logs and answers 200 — the transport must not retry it forever.
  - The secret is a wp-config constant. Any new option gets its default in `Install_Schema`.
- **Tests:** Money zone, so tests ship in this step. Replay of one `event_key` credits once. A missing key is refused. A wrong secret is 401 and reveals nothing. A bonus credits exactly the mapped coin count at the FIFO-head unit price. An unattributed event moves no wallet and writes no ledger row.
- **Verification (manual):** With a player holding the turn in a room, POST a bonus event with `curl`. The player's coin balance rises by the number the bonus map gives, and their **History** screen shows no new transaction. POST the identical body again — nothing changes anywhere.
- **Docs to update:** `docs/CONTRACTS.md` (endpoint + error codes, moved into "current"); `docs/DATA-MODEL.md` (new options and constants); `docs/ARCHITECTURE.md` (the ingest data flow, which currently says nothing pushes events in).
- **Depends on:** Step 1

### [ ] Step 5 — The transport
- **Tasks:**
  - Build the transport the Step 2 `DECISIONS.md` entry names. The task list differs per option, so `/plan-step` produces the real plan after Step 2 closes — writing it now would be guessing:
    - *HA automation → webhook:* the automation committed as documentation, the shared secret in place on both sides, `event_key` built Home-Assistant-side.
    - *Cron poller:* a `wp pc` command plus a real cron entry, a cursor transient, and the `last_changed`-derived key.
    - *WebSocket worker:* a small always-on service, its deploy target, and a reconnect/backoff policy.
  - Common to every option: it survives its own restart without replaying or losing events, and it fires at most one ingest call per real machine event.
- **Tests:** A restart mid-stream credits nothing twice. A dropped connection or a missed poll is either recovered or recorded as an explicit gap — never silently lost.
- **Verification (manual):** Someone plays a real turn. A real bonus on the machine credits that player within the latency the Step 2 entry promised, visible on the **Room** screen, with nobody running a command.
- **Docs to update:** `docs/ARCHITECTURE.md` (Integrations and the data flow); `docs/TECH-STACK.md` if a dependency or a deploy target is added; `docs/DECISIONS.md` if the build contradicted the spike.
- **Depends on:** Step 2, Step 4

## Definition of Done
- [ ] Every step closed via /close-step (report + verification guide + worklog)
- [ ] The check command (`docs/TECH-STACK.md` → Check command) exits 0 on the sprint branch, in both `backend/` and `frontend/`
- [ ] Docs match reality (DATA-MODEL, ARCHITECTURE, CONTRACTS, DECISIONS current)
- [ ] Sprint boundary: the sprint's work merged into `main` per the git model, and `backend` `main` pushed — that push *is* the production FTP release, and it is the run that verifies the transport in production. The ingest secret must exist in production `wp-config.php` **before** the merge, or the first real event is lost.
- [ ] Tymofii watched a real bonus on the machine credit the player holding the turn, with no command run by hand
- [ ] The same event replayed credited nothing a second time, observed in that player's balance
- [ ] Two rooms can no longer claim one machine — observed by trying it in the admin **Room list**
- [ ] Step 2 closed with a `DECISIONS.md` entry that names one transport and one `event_key` formula

## Out of scope
- The browser channel, the relay lock in the SPA, and removing the three 3-second polls — Sprint 2.
- Ops alerts — Sprint 3.
- The operations dashboard that aggregates the admin sections — a future `ops` feature, not this one.
- Everything in `BACKEND-REVIEW.md` except §12 (which is Step 3, because attribution is this feature's job). The rest keeps going through `/adhoc`.

## Risks / notes
- **Whether we have Home Assistant admin access is unknown until Step 2**, and it is the difference between the cheapest option and the most expensive one. Chase it on day one.
- Step 2 and Step 5 both need the physical machine powered and someone able to watch a real bonus. Neither can be finished from a desk.
- If Step 2 finds that a repeated bonus number is invisible in the sensor state, polling is off the table for money and the choice narrows to the webhook or the worker. Budget for that outcome.
- `ddev start` strips the JWT and Google constants from `wp-config-ddev.php` (see `docs/LEARNINGS.md`) — expect to restore them before local verification.
- A push to `backend` `main` is a production deploy. Nothing in this sprint may be merged there with a secret missing or a half-built transport.
