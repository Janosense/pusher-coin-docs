# SPRINT 3 — The operator learns about faults first (after Sprint 2)

**Branch:** `realtime/sprint-3` ← `main`, task branches `realtime/sprint-3-{short-name}` merged `--no-ff`.
**Goal:** When the machine goes offline, when a toss produces no coin movement, or when withdrawal requests pile up unusually, an operator finds out from the system rather than from a player writing into support. Each alert is reproducible on demand, so the operator can trust that silence means "nothing wrong" rather than "nothing watching".

## Fixed decisions
- Machine events live in `wp_pc_machine_events`; operator-facing events live in `wp_pc_auth_audit_log` — `docs/DATA-MODEL.md`. Alerts read those two, they do not invent a third store.
- Alerting never blocks a money path. Same rule as publishing — `FEATURE.md` → Invariants #2.
- No new operator screen in this sprint: the aggregated operations dashboard is a future `ops` feature, and alerts must be useful before a dashboard exists to show them on.

## Steps

### [ ] Step 1 — The machine is offline
- **Tasks:**
  - Detect the transition to offline from what already exists: `Machine_Service::is_online()`, the typed `machine_offline` errors, and the absence of expected events. Record it as a `machine_offline` event rather than only as a failed request.
  - Notify the operator once per incident, not once per failed call — an offline machine produces a lot of failures.
  - Record the recovery too, so an incident has an end.
- **Tests:** A sequence of failures produces one notification, not many. Recovery is recorded. A single transient failure inside the retry window does not raise an incident.
- **Verification (manual):** Power the machine off from the admin **Machine** screen, or block the endpoint. One notification arrives. Turn it back on — the recovery is recorded and the next fault can raise a new incident.
- **Docs to update:** `docs/DATA-MODEL.md` (new audit event types); `docs/ARCHITECTURE.md` (the alerting flow).
- **Depends on:** —

### [ ] Step 2 — A toss that moved nothing
- **Tasks:**
  - After a successful toss, the coin sensor is expected to move within a bounded window. If it does not, record it — the machine answered 200 but nothing physical happened, which is the case a player will dispute.
  - Make the window and the threshold configuration, not constants.
  - Include enough context in the record to settle a dispute: the player, the session, the toss event, the sensor readings on both sides.
- **Tests:** A toss followed by a sensor move is silent. A toss followed by no move raises exactly one record. The window is read from configuration.
- **Verification (manual):** Perform a real toss with the coin sensor disconnected or the machine jammed, and confirm the record appears with the player and session named. A normal toss produces nothing.
- **Docs to update:** `docs/DATA-MODEL.md` (the options and the event type); `docs/DOMAIN.md` if this gives the operator a term they use.
- **Depends on:** Step 1

### [ ] Step 3 — Withdrawals piling up
- **Tasks:**
  - Alert when pending withdrawals exceed a configured count, or when the oldest pending one exceeds a configured age. Both thresholds are configuration.
  - Alert at most once per period, so a genuinely busy day does not become a stream of notifications.
- **Tests:** Crossing the threshold alerts once. Staying above it does not re-alert within the period. Dropping below and crossing again does.
- **Verification (manual):** Create pending withdrawals past the threshold in a local environment and confirm one notification. Approve them in the admin **Withdrawals** screen and confirm the alert clears.
- **Docs to update:** `docs/DATA-MODEL.md` (the options); `docs/ROADMAP.md` (Phase 7 §4 → `[done]`).
- **Depends on:** Step 1

## Definition of Done
- [ ] Every step closed via /close-step (report + verification guide + worklog)
- [ ] The check command exits 0 on the sprint branch
- [ ] Docs match reality (DATA-MODEL, ARCHITECTURE, ROADMAP current)
- [ ] Sprint boundary: merged into `main`; `backend` released by the FTP push from `main`
- [ ] Tymofii reproduced all three alerts deliberately and received all three
- [ ] A normal hour of operation produced no alert at all — silence means silence
- [ ] ROADMAP Phase 7 §4 is `[done]` and Phase 5's exit criteria re-read against reality

## Out of scope
- The operations dashboard aggregating machine, withdrawals, pricing and tickets into one screen — ROADMAP Phase 7 §3, a future `ops` feature.
- Error tracking and request logging (Sentry and friends) — ROADMAP Phase 8 observability.
- Anything that pages a human out of hours. These alerts reach an operator during the day; escalation policy is a business decision nobody has made.

## Risks / notes
- The delivery channel is undecided: email to `pc_support_email` is the cheapest and reuses what `Support_Service` already does, but an operator who does not read that inbox gains nothing. Settle it in Step 1's plan, with a `DECISIONS.md` entry.
- An alert nobody can reproduce is an alert nobody trusts. Every step here carries "reproduce it deliberately" in its manual verification for that reason.
- Thresholds set from a desk are guesses. Expect to tune them after the first week of real traffic, which is why they are configuration.
