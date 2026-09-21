# SPRINT 3 — The operator learns about faults first (after Sprint 2)

<!-- playbook: v1.21. Written by discovery (Phase C) together with every
     other sprint of the plan — never by Claude Code. Rewritten by a
     re-planning chat only while no step is closed; afterwards steps may
     only be appended. This file
     describes the sprint and nothing else: goal, fixed decisions, steps.
     The only thing written here during the sprint is the tick in a step's
     heading, by /close-step. -->
**Branch:** `realtime/sprint-3` ← `main`, task branches `realtime/sprint-3-{short-name}` merged `--no-ff`.
**Goal:** When the machine is unreachable during a scheduled broadcast window, when a toss produces no coin movement, or when withdrawal requests pile up unusually, an operator finds out from the system rather than from a player writing into support. Each alert is reproducible on demand, fires once per incident rather than once per failed call, and stays silent through a normal day — including the daily hand-switched power-off outside broadcast hours, which is not a fault. The delivery channel is decided and written down before the first alert is built.

## Fixed decisions
- **Machine power is manual** — `DECISIONS.md` 2026-09-18. The machine goes off by hand every day; *offline during a scheduled window* is the incident, and `wp_pc_room_schedules` (`Room_Schedule_Calculator`) is what tells the two apart. An alert that fires every evening is a defect of the step.
- **Machine events live in `wp_pc_machine_events`; operator-facing events live in `wp_pc_auth_audit_log`** — `docs/DATA-MODEL.md`. Alerts read those two; they do not invent a third store.
- **Alerting never blocks a money path** — same rule as publishing, `FEATURE.md` → Invariants #2.
- **No new operator screen in this sprint.** The aggregated operations dashboard is a future `ops` feature (`FEATURE.md` → Roadmap); alerts must be useful before a dashboard exists to show them on.
- **Thresholds and windows are configuration**, with defaults in `Install_Schema` — `CLAUDE.md` core rule on business values. Set from a desk they are guesses; they will be tuned after the first week of real traffic.

## Steps

### [x] Step 1 — The machine is unreachable during a broadcast window
- **Tasks:**
  - Decide the delivery channel and record it as a `DECISIONS.md` entry before writing code: email to `pc_support_email` reuses what `Support_Service` already does, but an operator who does not read that inbox gains nothing. The step plan puts the question to the user.
  - Detect the transition to unreachable from what already exists — `Machine_Service::is_online()`, the typed `machine_offline` errors, the absence of expected events — **and gate it on the room's current schedule window**: unreachable outside a window is the daily power-off and is recorded, not alerted.
  - Notify once per incident, not once per failed call, and record the recovery so an incident has an end.
- **Tests:** A sequence of failures inside a window produces one notification; the same sequence outside a window produces none and one audit record. Recovery is recorded. A single transient failure inside the retry window raises nothing.
- **Verification (manual):** In a local environment with a room whose schedule window is *now*, point the machine endpoint at a dead address: one notification arrives on the chosen channel. Move the window to tomorrow and repeat: nothing arrives, the audit log shows the outage. Restore the endpoint: the recovery is recorded and the next fault can raise a new incident.
- **Docs to update:** `docs/DECISIONS.md` (the delivery channel); `docs/DATA-MODEL.md` (new audit event types, new options); `docs/ARCHITECTURE.md` (the alerting flow).
- **Depends on:** —

### [x] Step 2 — A toss that moved nothing
- **Tasks:**
  - After a successful toss, the coin sensor is expected to move within a bounded window, as Sprint 1 Step 2 measured. If it does not, record it — the machine answered 200 but nothing physical happened, which is the case a player will dispute.
  - The window and the threshold are configuration, not constants.
  - The record carries enough to settle a dispute: the player, the session, the toss event, the sensor readings on both sides.
- **Tests:** A toss followed by a sensor move is silent. A toss followed by no move raises exactly one record. The window is read from configuration.
- **Verification (manual):** During venue hours, a real toss with coins in the machine produces nothing. In a local environment, replay a toss event with no following coin event through the Sprint 1 ingest endpoint and confirm the record appears with the player and session named.
- **Docs to update:** `docs/DATA-MODEL.md` (the options and the event type); `docs/DOMAIN.md` if this gives the operator a term they use.
- **Depends on:** Step 1

### [ ] Step 3 — Withdrawals piling up
- **Tasks:**
  - Alert when pending withdrawals exceed a configured count, or when the oldest pending one exceeds a configured age. Both thresholds are configuration.
  - Alert at most once per period, so a busy day does not become a stream of notifications; re-alert only after dropping below and crossing again.
- **Tests:** Crossing the threshold alerts once. Staying above it does not re-alert within the period. Dropping below and crossing again does.
- **Verification (manual):** Create pending withdrawals past the threshold in a local environment and confirm one notification. Approve them in the admin **Withdrawals** screen and confirm the alert clears and a second crossing alerts again.
- **Docs to update:** `docs/DATA-MODEL.md` (the options); `docs/ROADMAP.md` (Phase 7 §4 → `[done]`, and Phase 5's exit criteria re-read against reality).
- **Depends on:** Step 1
