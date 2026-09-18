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
- **Entry point:** one `require_once TEMPLATE_DIR . '/app/realtime/bootstrap.php'` line in `functions.php`, the same pattern `app/stripe/bootstrap.php` uses; in each SPA, one import from the store that consumes the channel
- **Shared code it depends on:** `Machine_Ingest_Service` and `Machine_Event_Log` (the ingest seam), `Queue_Service` (attribution and the open bet session), `Machine_Service` (relay and sensor reads), `Room_Schedule_Calculator` (broadcast windows, for alerts), `Rate_Limiter`, `Audit_Log`, `Install_Schema`. On the SPA side `stores/queue.js`, `stores/chat.js` and `components/PlaceBet.vue` — all owned by `core`, so touching them is a plan task marked **"touches shared code"**. Sprint 1 Step 1 refines this list.

## Data
Owns no table. It writes `wp_pc_machine_events` **only through
`Machine_Ingest_Service`** — the row shape and its `event_key` unique index belong
to `core` (`docs/DATA-MODEL.md`). Owns these keys, and no other feature writes them:
- WP options `pc_realtime_*` — channel name, alert thresholds and windows, any
  transport setting the spike's choice needs. Defaults seeded by `Install_Schema`.
- Transients `pc_realtime_cursor_*` — last-seen sensor state, only if the spike
  picks polling. Rebuildable; never a source of truth for money.
- wp-config constants `PC_ABLY_KEY` and the ingest shared secret. Never options,
  never logged.

## Invariants
1. **Every accepted inbound event carries an `event_key`.** A transport that
   cannot produce a stable one is not accepted.
2. **Publishing and alerting are fire-and-forget.** A failure is logged and
   swallowed; it never fails, rolls back or delays the money path that triggered it.
3. **The SPA never sees the Ably key** — it asks for a scoped token.
4. **The ingest endpoint is not public.** Shared secret, rate-limited, audited; a
   bad secret is a 401 that says nothing about why.
5. **A machine id resolves to at most one room with a live queue.**
6. **Nothing here debits a wallet, and nothing here switches the machine.** It
   credits only, only through `Machine_Ingest_Service`; power is a hand at the venue.

## Interfaces
- `POST /pc/v1/machine/events` — the ingest endpoint, whatever transport calls it.
- `GET /pc/v1/realtime/token` — scoped channel token for a signed-in SPA.
- The channel naming convention — one per room, one for the machine — which the
  admin SPA reads too; changing it is "touches shared surface".
- It consumes, and does not change, `core`'s `pc_machine_event_player` filter and
  `pc_machine_event_credited` action.

## UI
- **Screens:** none of its own. It changes two screens `core` owns, indexed in
  `docs/DESIGN.md` → Screens: **Room** (player SPA) and **Machine** (admin SPA).
- **Reuses:** `PlaceBet`, `UserControls`, `RoomQueue`, `RoomChat`.
- **Introduces:** — (no design; `design/` stays empty — `DECISIONS.md` 2026-09-15).

## Roadmap
- Sprint 1 — a bonus on the machine credits the player holding the turn, with nobody running a command; the sensor model matches the machine (`sprints/SPRINT-1.md`)
- Sprint 2 — the room reflects coins, bonuses and the relay lock without polling (`sprints/SPRINT-2.md`)
- Sprint 3 — the operator learns about a fault from the system, not from a support ticket; a hand-switched power-off is not a fault (`sprints/SPRINT-3.md`)
- Left for later, outside these sprints: the operations dashboard (a future `ops` feature); error tracking and request logging (ROADMAP Phase 8); out-of-hours escalation (a business decision nobody has made); the admin **Machine** screen's own 3-second poll, which may ride the machine channel when Sprint 2 makes it free but is not a goal.
