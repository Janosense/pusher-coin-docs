# Sprint 3 — step plans (`realtime`)

<!-- Written by /plan-step, one section per step. A section's status moves
     awaiting approval → approved, in progress → implemented, awaiting close →
     closed. Never edit another step's section. -->

## Plan — Sprint 3, Step 1: The machine is unreachable during a broadcast window   (status: implemented, awaiting close)

### Branch
`realtime/sprint-3-outage` ← `realtime/sprint-3` ← `main`
(git model in root `CLAUDE.md`. `realtime/sprint-3` does not exist yet and is cut from
`main` by `/do-step`. The same branch name in `backend/` and the docs repository;
`frontend/` and `admin/` are **not touched** — the sprint's own Fixed decisions say there
is no new operator screen, and nothing here changes a REST response.)

### What is settled before this step starts
- **Machine power is manual** — `DECISIONS.md` 2026-09-18. The machine goes off by hand every day. *Unreachable outside a broadcast window is the normal evening*, and an alert that fires every evening is a defect of this step.
- **`Machine_Service` is the only code that talks to Home Assistant** — root `CLAUDE.md` invariant 8. The probe is `Machine_Service::is_online()`, which `ROADMAP.md` Phase 5 §1 already earmarks for "Phase 7's ops view".
- **Alerting never blocks a money path** — `FEATURE.md` → Invariants #2, the same rule publishing follows. A failed notification is logged and swallowed.
- **Operator-facing events live in `wp_pc_auth_audit_log`** — the sprint's Fixed decisions. No third store, no new table.
- **Thresholds and windows are configuration**, defaults in `Install_Schema` — root `CLAUDE.md` core rule 3.
- **No new operator screen** — the sprint's Fixed decisions. The aggregated dashboard is a future `ops` feature.

### What already exists, and what it gives this step
- **`Machine_Service::is_online()`** (`machine-service.php:223`) — a 2-second probe of Home Assistant's `/api/` root. Returns a plain `false` for a connection error, a non-200 **and** an unconfigured install. That last case matters: an install with no token must not be reported as an outage, so the watch checks `is_configured()` separately rather than reading `false` as "the machine is down".
- **The poll pass** (`Machine_Poller::run()`) already runs every `pc_realtime_poll_interval_seconds` (60) and already carries one watch — `Realtime_Relay_Watch::run()` is called first, outside every guard, so that neither half can stop the other. This step adds the second watch in exactly that position, for exactly that reason. **No second cron**, nothing new to deploy.
- **`Room_Schedule_Calculator::compute( $rules )`** (`room-schedule-calculator.php:43`) answers `current_window` / `next_window` from `wp_pc_room_schedules` rows, in the site timezone. It is documented as *pure and free of DB access*, so the caller loads the rules. Four callers already do that with their own inline query (`RoomController.php:125`, `:421` and `AdminRoomController.php:201`, `:421` — `BACKEND-REVIEW.md` → Cleanup lists the duplication). This step adds a fifth read **inside `app/realtime/`** rather than a `rules_for_room()` on the calculator: putting a query on a class whose docblock promises no DB access is a worse trade than one contained copy, and `core` stays untouched. The copy names the duplication in a comment.
- **`Queue_Service::room_id_for_machine()`** — machine id → room, which is the only way to reach a schedule. No room carries the id ⇒ there is no window to be inside ⇒ **record, never notify**: with no schedule, an evening power-off and a fault are indistinguishable, and guessing would produce the nightly false alarm the sprint explicitly forbids.
- **`Support_Service::notify_support()`** is `private` (`support-service.php:271`) and shaped around a ticket. `FEATURE.md` already flags that reusing it would change shared code. This step writes its own ten-line sender instead and leaves `core` alone.
- **DDEV runs Mailpit** at `https://pusher-coin.ddev.site:8026`, so an email channel is fully checkable locally.

### The detection rule
**Unreachable is decided by `is_online()`, not inferred.** One extra 2-second-bounded call
per pass, on a schedule that already makes one. The typed `machine_offline` the relay read
may have just produced is recorded **alongside** as corroboration, never as the decision —
`machine_unavailable_state` and `machine_call_failed` mean Home Assistant answered and one
entity is unhappy, which is not an outage.

**"The absence of expected events" is not used,** though the step lists it as a candidate.
A machine that reports nothing is the normal state of a machine nobody is playing; it
cannot distinguish idle from unreachable, and building on it would produce exactly the
false alarm the sprint forbids. Recorded here rather than silently dropped.

**One incident, and the window decides only the notification.** A transient failure raises
nothing: the outage must persist past `pc_realtime_outage_grace_seconds` (300, five
passes) before it is an incident at all. The incident is recorded once, on the pass it
becomes one, with `in_window` as it stood at that moment. The **notification** fires the
first time an incident is live *and* the room is inside a broadcast window — so an outage
that begins at 03:00 and is still there when the evening window opens alerts when the
window opens, which is the moment it starts costing the business something. One
notification per incident, whichever way round it happened; recovery closes the incident
and the next fault can raise a new one.

### Tasks (ordered)
- [x] **1 — One place a notification leaves the system.** `app/realtime/alerts.php` — `Realtime_Alerts::send( string $subject, string $body, array $context ): bool`: resolves the destination (`pc_realtime_alert_email`, falling back to `pc_support_email`, falling back to `admin_email`), sends with `wp_mail`, audits `operator_alert_sent` / `operator_alert_failed`, and **swallows everything** — it is called from a money-adjacent path and may never throw, delay or roll one back. An address that resolves to nothing is a silent no-op, the same shape an unconfigured Ably key already has. Steps 2 and 3 of this sprint use this and nothing else. `DECISIONS.md` (the channel, per Question 1) in the same commit. → `feat(realtime): one door for operator alerts`
- [x] **2 — The outage watch.** `app/realtime/outage-watch.php` — `Realtime_Outage_Watch::run( bool $dry_run = false ): array` returning `{reachable:?bool, incident:bool, in_window:?bool, notified:bool, stopped:?string}`. Unconfigured machine id or unconfigured Home Assistant ⇒ `stopped`, silent. Reachable ⇒ close any open incident, audit `machine_outage_recovered` with its duration and whether it was ever notified, clear the state. Unreachable ⇒ hold `pc_realtime_outage_state` (`{down_since, failures, incident_at, notified}`), and past the grace record `machine_outage_started` once; notify through task 1 the first pass on which the incident is live and the room is inside a window, recording `machine_outage_notified`. Options seeded in `install-schema.php`: `pc_realtime_outage_grace_seconds` (300) and `pc_realtime_alert_email` (`''`), `DB_VERSION` `1.13.0` → `1.14.0`. **Touches shared code (`core`): `install-schema.php`, additive — two `add_option` lines and the version bump; consuming features: `core`, `realtime`, `stripe` (all read the same installer).** `DATA-MODEL.md` (the options and the audit event types) in the same commit. → `feat(realtime): an unreachable machine during a broadcast window is an incident`
- [x] **3 — Wire it into the pass that already runs.** `machine-poller.php`: call `Realtime_Outage_Watch::run( $dry_run )` immediately after the relay watch, inside the try and **outside** the machine-id/secret guard, for the reason the relay watch is there; add `outage` to `run()`'s returned shape and to `pc_realtime_poll_last_run`. `machine-poll-command.php`: an `outage:` line in the pass output and in `--status`, mirroring the `relay:` line Sprint 2 Step 3 added — without it a host with no shell cannot see whether the machine is currently down. `ARCHITECTURE.md` (the alerting flow) and `PROJECT-TREE.md` in the same commit. → `feat(realtime): the poll pass reports the machine's reachability`
- [x] **4 — The checks.** `tests/realtime-outage.php`, listed below. → `test(realtime): the outage incident and its window gate`
- [x] **5 — The feature's own record.** `docs/features/realtime/FEATURE.md` — the new module in Interfaces, the two options under Data, and an invariant for "an alert never blocks a money path, and an unreachable machine outside a window is not a fault". → `docs(realtime): the outage watch in the feature record`

### Files to create/change
**`backend/` — created**
- `wp-content/themes/pc/app/realtime/alerts.php`
- `wp-content/themes/pc/app/realtime/outage-watch.php`
- `wp-content/themes/pc/tests/realtime-outage.php`

**`backend/` — changed**
- `wp-content/themes/pc/app/realtime/bootstrap.php` — two `require_once` lines.
- `wp-content/themes/pc/app/realtime/machine-poller.php` — one call, one key in the result and in `pc_realtime_poll_last_run`.
- `wp-content/themes/pc/app/realtime/machine-poll-command.php` — one reporting line in two places.
- `wp-content/themes/pc/app/utils/install-schema.php` — **shared (`core`)**, additive: two option defaults, `DB_VERSION` `1.13.0` → `1.14.0`. No schema change; the bump is what makes `install_default_options()` run again, exactly as 1.10.0–1.12.0 did.

**Not changed, and worth saying so:** `machine-service.php` (`is_online()` is used as it stands), `support-service.php` (its private mailer is left alone), `queue-service.php`, `room-schedule-calculator.php` (it stays pure), every controller, and anything under `frontend/src/` or `admin/src/`. No REST route, response or error code changes.

### Tests to write
`backend/wp-content/themes/pc/tests/realtime-outage.php`, a WP-CLI `eval-file` script with
the DDEV guard every other script carries, an `$audit_floor` captured at start-up so every
audit assertion counts this run's rows only, before/after row counts printed for every
table it can touch, and **two stubs**: `pre_http_request` for Home Assistant (so
reachability is a variable the test sets) and `pre_wp_mail` for the notification (so
nothing is sent and the message can be read). The schedule fixtures use a whole-day window
for "in window" and a different weekday for "outside", which sidesteps the calculator's
`start_time < end_time` constraint and the midnight edge entirely.

1. **A single transient failure raises nothing.** One unreachable pass inside a live window: no notification, no `machine_outage_started`, and the state records the failure.
2. **Past the grace, inside a window: exactly one notification.** Repeated unreachable passes cross `pc_realtime_outage_grace_seconds`; exactly one `machine_outage_notified`, exactly one mail, exactly one `machine_outage_started`. Ten more passes add none of the three.
3. **The same sequence outside a window: none, and one record.** No mail at all, no `machine_outage_notified`, and exactly one `machine_outage_started` carrying `in_window: false`. **This is the check that the sprint's "an alert that fires every evening is a defect" is actually met.**
4. **An outage that runs into a window alerts when the window opens.** Down outside a window past the grace (silent), then the window becomes live: one notification, then none.
5. **Recovery ends the incident.** A reachable pass writes one `machine_outage_recovered` carrying the duration and whether it had been notified, and clears the state; a fresh outage afterwards can notify again. A second reachable pass records nothing.
6. **No room carries the machine id ⇒ recorded, never notified.** There is no window to be inside, and the evening power-off would otherwise alert every night.
7. **Unconfigured is not an outage.** No `pc_realtime_poll_machine_id`, and separately no Home Assistant token: `stopped`, no mail, no incident.
8. **The grace comes from configuration.** Setting `pc_realtime_outage_grace_seconds` to 0 makes the first failed pass an incident; the checks read the behaviour, not the constant.
9. **The alert cannot break the pass or the money.** `wp_mail` returning false and `wp_mail` throwing both leave `Realtime_Outage_Watch::run()` returning normally and `Machine_Poller::run()` crediting a stubbed payout exactly as it does with alerting silent — the money path is byte-for-byte unaffected (`FEATURE.md` → Invariants #2).
10. **The message is worth receiving.** Its body names the room, the window it is inside, how long the machine has been unreachable and the last error code — enough to act on without opening a database.
11. **The destination resolves as documented.** `pc_realtime_alert_email` wins; empty falls back to `pc_support_email`; both empty falls back to `admin_email`; an invalid address sends nothing and audits `operator_alert_failed` rather than throwing.
12. **Cleanup.** Rooms, schedule rules, users, options, transients and the audit rows above the floor are all removed, with before/after counts printed.

### Docs to update
- `docs/DECISIONS.md` — the delivery channel, per Question 1. Task 1, **before the sender is written**, which is the order the step text asks for.
- `docs/DATA-MODEL.md` — `pc_realtime_outage_grace_seconds`, `pc_realtime_alert_email` and `pc_realtime_outage_state` in the realtime options tables; the new audit event types. Task 2.
- `docs/ARCHITECTURE.md` — the alerting flow, and the new files in the tree. Task 3.
- `docs/PROJECT-TREE.md` — the three new files. Task 3. *(On the list deliberately: `LEARNINGS.md` 2026-09-21 records that the tree maps have no owner in the close checklist and went stale through two consecutive steps.)*
- `docs/features/realtime/FEATURE.md` — Interfaces, Data, Invariants. Task 5.
- **Not touched:** `CONTRACTS.md` (no endpoint, payload or error code changes), `DESIGN.md` and `FEATURE.md` → UI (no screen), `TECH-STACK.md` (no dependency — `wp_mail` is WordPress core), `DOMAIN.md` (Step 2 of this sprint is the one that may give the operator a new term), `ROADMAP.md` (**Phase 7 §4 → `[done]` belongs to Step 3 of this sprint**, which is the step that finishes the item; re-tagging it here would claim two thirds of it).

### Checks
- **ANTI-PATTERNS:** none violated. Home Assistant is reached only through `Machine_Service`; no `ENUM`; no money column touched anywhere; `$wpdb` is never expected to throw (the only writes are `update_option` / `add_option` and `Audit_Log::record`); no float in a money path; **no cron is added** — the watch rides the pass that already exists, which is also what keeps the "no cron for queue housekeeping" rule's spirit; no REST route, so no `permission_callback` question; nothing is hard-deleted; the grace, the address and the interval are all operator-tunable options with defaults in `Install_Schema`, never constants.
- **Docs vs reality:** mismatch, two, both folded into tasks. (1) `DATA-MODEL.md`'s audit event-type table is **stale** — it lists no `realtime` row at all, so `machine_poll_*` (S1.5), `machine_relay_read_failed` (S2.3) and `queue_session_orphan_closed` (S2.5) are undocumented; task 2 adds the row and lists them with the new ones rather than adding only its own. (2) The step's manual verification asks for "a room whose schedule window is *now*", which **nothing creates**: `wp pc seed-rooms` writes fixed weekday rules, and whether one covers the moment you run it is chance. The verification guide will carry a paste-able block that makes one — the same class as `LEARNINGS.md` 2026-09-18, "a sprint's manual verification named local data that nothing creates", caught before any code this time. **Observations that change no task:** `BACKEND-REVIEW.md` → Cleanup already lists the duplicated schedule-rule query; this step adds a fifth copy with its reason, and consolidating all five is `/adhoc`. The carried items are unchanged — the late-payout handover, the two unassigned §10 bullets, `settle()`'s unchecked `mark()`, the stale `TECH-STACK.md` eval-script list, and the two test scripts that leave a row behind per run.
- **Design:** n/a — no screen, component, token or string. The sprint's Fixed decisions forbid a new operator screen, and nothing here changes an existing one.
- **Check command:** `backend/bin/check` (`docs/TECH-STACK.md` → Check command). DDEV is running, so stage 2 executes all twelve `tests/*.php` including the new one. `frontend/bin/check` is not run: `frontend/` is not touched.
- **Not locally verifiable:** **whether `wp_mail` actually delivers from the production host.** DDEV catches mail in Mailpit, so everything above is checkable locally right up to the last hop; the FTP shared host's own mail path is not. It is the same mechanism support-ticket notifications already use, and it has never been confirmed in production either. The one real run that verifies it is **the first alert after the next deploy from `main`** — or, sooner and safer, the operator filing one test support ticket. Named here rather than assumed, and it is the reason Question 1's recommendation keeps the destination configurable.

### Questions / ambiguities

**Question 1 — Which channel does an alert go out on? The step requires this to be decided and written into `DECISIONS.md` before any code is written, and says in as many words that email to the support inbox "reuses what `Support_Service` already does, but an operator who does not read that inbox gains nothing".**
**Resolved: approved as recommended — (a) email, to its own configurable address `pc_realtime_alert_email`, falling back to `pc_support_email` and then `admin_email`.**
- **(a) — recommended. Email, to its own configurable address `pc_realtime_alert_email`, falling back to `pc_support_email` and then to `admin_email`.** It is (b)'s zero-cost mechanism with the step's own objection answered: the operator points alerts at whatever they actually watch — a phone-notifying address, a shared mailbox, a Telegram-bridging address — without redirecting support mail or waiting on this project. No new dependency (core rule 1 not engaged), no new secret, no account to provision, and fully checkable locally through DDEV's Mailpit. *The tasks above are written to this.* The honest cost: email is not a push notification, and delivery from the shared host is unproven — both stated in the `DECISIONS.md` entry rather than left to be discovered.
- **(b) Email to `pc_support_email`, with no separate option.** Simplest possible, one fewer option to document. It is the version the step already names and already doubts: alerts land in the same inbox as player complaints, which is precisely the inbox the sprint goal wants the operator to stop depending on.
- **(c) A Telegram bot.** Very likely what the venue operator would actually read, and a genuinely better channel. It costs a new external service, a bot token as a wp-config constant, and an account somebody has to create — and this project has just spent a whole sprint learning what it costs to plan around an account nobody provisioned (`LEARNINGS.md` 2026-09-21, and `TECH-STACK.md` still records the Ably peak as unmeasured for that reason). Under (a) the sender is one function with one call site, so adding Telegram later is a task, not a rewrite. Choose this if you already have a bot token in hand — then it belongs in this step and the tasks gain a second channel behind the same door.
- **(d) Record in the audit log only, deliver nothing.** Rejected on the sprint goal itself: "an operator finds out from the system rather than from a player writing into support" is not met by a row nobody reads, and the sprint's Fixed decisions rule out building the screen that would show it.

### Execution notes
- **Commits.** backend `6480a1b2` `05628135` `224a49be` `e2d6ec14`; docs `2783151` `3cebb3f` `4892c1a` `8beeccd`. `frontend/` and `admin/` untouched, as planned. `realtime/sprint-3` cut from `main`.
- **One check could not be written as planned, and says so instead of proving something else.** The plan's test 7 promised "no Home Assistant token: `stopped`, no mail, no incident". `Machine_Service::is_configured()` is false only when `PC_MACHINE_TOKEN` is absent, and every script in `tests/` must define that constant to run at all — a constant cannot be undefined mid-process. The guard is one line and mirrors the relay watch's, so it ships; the script's docblock names it as knowingly unchecked, and the check in its place covers the trap next to it: **blanking `pc_machine_endpoint` does not unconfigure an install**, because `Machine_Service::option()` treats an empty value as "use the built-in default". Anyone later reaching for that as a way to simulate a disconnected install now finds a check rather than a surprise. The machine-id half of the same guard is checked normally.
- **The differential is inside one run.** Section 2 and section 3 feed the watch the *same* sequence of failed passes and differ only in the room's schedule: one alerts, the other sends nothing and records exactly one outage. That is the sprint's "an alert that fires every evening is a defect of the step", proved rather than asserted — and it is why no code-reversion experiment was needed.
- **`DATA-MODEL.md`'s audit table gained five rows, not one.** The plan flagged it as stale; it was worse than "missing this step's types" — the transport (S1.5), the relay watch (S2.3) and the session cleanup (S2.5) had all been writing undocumented event types. All are listed now, with a note saying why they arrived together.
- **`pc_realtime_poll_last_run` gained an `outage` key** alongside `relay`, and the CLI a `machine:` line in both the pass output and `--status` — derived necessity, as the plan's task 3 stated: a host with no shell has nowhere else to read whether the machine is down.
- **Checks:** `tests/realtime-outage.php` 64 checks, cleanup identical across six tables. `backend/bin/check` exit 0, `php -l: 71 files OK`, all twelve `tests/*.php` passing. `frontend/bin/check` not run — `frontend/` is not touched.
- **Local install:** `pc_db_version` 1.14.0, both options seeded (`pc_realtime_alert_email` empty, grace 300), and the fallback chain resolves to the site admin address on this machine. `wp pc machine-poll --status` prints the new `machine:` line.
