# Feature — realtime

<!-- Lightweight ARCHITECTURE + DATA-MODEL for one feature. Lives at
     docs/features/realtime/FEATURE.md next to its sprints/. Root
     ARCHITECTURE.md holds only one row + a link here. Keep ≤80 lines. -->

## Purpose & scope
Makes the physical machine's events reach the right player by themselves. Today
nothing carries a coin drop, a bonus or a relay close from Home Assistant into
WordPress — the only producer is `wp pc machine-ingest` run by hand — so
settlement is wired and idle, and the room keeps three 3-second polls alive to
fake liveness. This feature builds the inbound transport, the outbound channel to
the browser, and the attribution guarantees that make a payout land on the player
who actually held the turn.

**Out of scope:** the operations dashboard that aggregates machine, withdrawals,
pricing and tickets into one screen (a UI surface — a future `ops` feature); all
Phase 8 launch readiness (i18n, a11y, performance, the `admin` deploy target,
the security review, compliance, observability, backups); the parked Google,
Apple and captcha integrations, which are blocked on credentials, not on code;
and the rest of `BACKEND-REVIEW.md`, which goes through `/adhoc`.

## Fit into the host
- **Code location:** `backend/wp-content/themes/pc/app/realtime/`, `frontend/src/services/realtime.js`, `admin/src/services/realtime.js`
- **Host area:** the repository root — the project's single code area, governed by the root `CLAUDE.md`
- **Entry point:** one `require_once TEMPLATE_DIR . '/app/realtime/bootstrap.php'` line in `functions.php`; in each SPA, one import from the store that consumes the channel
- **Shared code it depends on:** `Machine_Ingest_Service` and `Machine_Event_Log` (the ingest seam), `Queue_Service` (attribution and the open bet session), `Machine_Service` (relay and sensor reads), `Rate_Limiter`, `Audit_Log`, `Install_Schema` for any new option default. On the SPA side `stores/queue.js`, `stores/chat.js` and `components/PlaceBet.vue` — all owned by `core`, so touching them is a plan task marked **"touches shared code"**.

## Data
Owns no table. It writes `wp_pc_machine_events` **only through
`Machine_Ingest_Service`**, never directly — the row shape and its `event_key`
unique index belong to `core` (`docs/DATA-MODEL.md`).

Owns these keys:
- WP options `pc_realtime_*` — the channel name and any transport settings the
  spike's choice needs. Defaults seeded by `Install_Schema`.
- Transients `pc_realtime_cursor_*` — last-seen sensor state, only if the spike
  picks polling. Rebuildable; never a source of truth for money.
- wp-config constants `PC_ABLY_KEY` and the ingest shared secret. Never options,
  never logged — they are secrets like every other.

No other feature writes any of this.

## Invariants
1. **Every accepted inbound event carries an `event_key`.** A transport that
   cannot produce a stable one is not accepted — the unique index is the only
   thing between a retry and a double credit.
2. **Publishing to the browser channel is fire-and-forget.** A failed publish is
   logged and swallowed; it must never fail, roll back or delay the money path
   that triggered it.
3. **The SPA never sees the Ably key** — it asks for a scoped token.
4. **The ingest endpoint is not public.** Shared secret, rate-limited, audited;
   a bad secret is a 401 that says nothing about why.
5. **A machine id resolves to at most one room with a live queue.** Attribution
   is meaningless otherwise, and a win goes to the wrong player.
6. **Nothing here debits a wallet.** This feature only credits, and only through
   `Machine_Ingest_Service`.

## Interfaces
- `POST /pc/v1/machine/events` — the ingest endpoint, whatever transport calls it.
- `GET /pc/v1/realtime/token` — scoped channel token for a signed-in SPA.
- The channel naming convention (per room, plus one admin channel), which the
  admin SPA reads too — changing it is "touches shared surface".
- It consumes, and does not change, `core`'s `pc_machine_event_player` filter and
  `pc_machine_event_credited` action.

## UI
- **Screens:** none of its own. It changes the behaviour of two screens `core`
  owns, both indexed in `docs/DESIGN.md` → Screens: **Room** (player SPA) and
  **Machine** (admin SPA). Sprint steps name them exactly that way.
- **Reuses:** `PlaceBet`, `UserControls`, `RoomQueue`, `RoomChat` — existing
  components, existing tokens.
- **Introduces:** — (no new tokens, no new shared components; `design/` stays
  empty, per the 2026-09-15 decision that this feature has no UI design).

## Roadmap
- Sprint 1 — a bonus on the machine credits the player holding the turn, with nobody running a command (`sprints/SPRINT-1.md`)
- Sprint 2 — the room reflects coins, bonuses and the relay lock without polling (`sprints/SPRINT-2.md`)
- Sprint 3 — the operator learns about a fault from the system, not from a support ticket (`sprints/SPRINT-3.md`)
