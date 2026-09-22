# Sprint 3 — step plans (`realtime`)

<!-- Written by /plan-step, one section per step. A section's status moves
     awaiting approval → approved, in progress → implemented, awaiting close →
     closed. Never edit another step's section. -->

## Plan — Sprint 3, Step 1: The machine is unreachable during a broadcast window   (status: closed)

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

---

## Plan — Sprint 3, Step 2: A toss that moved nothing   (status: closed)

### Branch
`realtime/sprint-3-toss` ← `realtime/sprint-3` ← `main`
(git model in root `CLAUDE.md`. The sprint branch exists — Step 1 is merged into it. The
same branch name in `backend/` and the docs repository; `frontend/` and `admin/` are
**not touched**: no screen, no REST response and no error code changes.)

### What is settled before this step starts
- **`sensor.coin` counts coins paid out since the last toss, and a toss resets it to 0** — `DECISIONS.md` 2026-09-18, the spike. It is not cumulative. 13 of the 14 resets in ten days of history came **0.09–1.98 s** after a `toss_a_coin` press; the first coin of a payout came **1–12 s** after a press, and a payout kept counting for up to **~20 s**.
- **A repeated value leaves no trace** — same entry, measured: in all 2,768 live samples `last_reported` equalled `last_updated`, and the two real toss presses at 13:16:41Z and 13:17:04Z (which re-wrote the counter's `0`) produced **no row at all**. Home Assistant's integration writes only *changed* values. This is the fact the whole step turns on; see "The detection rule" below.
- **`Machine_Service` is the only code that talks to Home Assistant** — root `CLAUDE.md` invariant 8. The reading is `Machine_Service::get_state_history()` (`machine-service.php:112`), the same call the transport makes, with its own 10-second budget.
- **Machine events live in `wp_pc_machine_events`; operator-facing events live in `wp_pc_auth_audit_log`** — the sprint's Fixed decisions. No third store, no new table.
- **`event_key` is the idempotency guard** — `FEATURE.md` → Invariants #1 and `DATA-MODEL.md`. "Exactly one record" rests on the unique index, not on a counter.
- **Thresholds and windows are configuration**, defaults in `Install_Schema` — root `CLAUDE.md` core rule 3.
- **Alerting never blocks a money path** — `FEATURE.md` → Invariants #2. Nothing here runs inside the toss request at all (see below), so it cannot delay, fail or roll one back.

### What already exists, and what it gives this step
- **The toss is already recorded.** `RoomQueueController::play()` writes a `toss` row through `Machine_Ingest_Service::log_event()` (`RoomQueueController.php:223-231`) carrying `machine_id`, `user_id` and a payload of `{room_id, session_id}` — **the player and the session the step asks the record to name are already on file.** `created_at` is `DATETIME(6)`, UTC, microsecond precision. The row has no `event_key` (NULL is permitted).
- **The poll pass** (`Machine_Poller::run()`) runs every `pc_realtime_poll_interval_seconds` (60) and already carries two watches called before the coin work and **outside** its machine-id/secret guards, so neither half can stop the other. This step adds the third in that position, for that reason. **No new cron, nothing new to deploy.**
- **`Machine_Service::get_state_history( $entity, $since_iso, $until_iso )`** returns every state an entity passed through, oldest first, `last_updated` verbatim, with `significant_changes_only=0` so resets are included. Home Assistant **returns the state at the window's start as the first row**, which is what makes "what did the counter read at the moment of the toss?" answerable in the same call.
- **`Machine_Poller::sensor_entity()`** resolves `pc_machine_coin_sensor_entity` (default `sensor.coin`). It is `private` today (`machine-poller.php:440`); this step makes it `public` so the watch reads **the same entity the transport credits from** rather than resolving its own — a disclosed visibility change inside `app/realtime/`, not shared code.
- **`wp_pc_machine_events.correlation_id`** is documented as "reserved for `wp_pc_bet_sessions.id`; always NULL today — no caller supplies it" (`DATA-MODEL.md`). This step is the first caller to supply it, for exactly that purpose.
- **`wp pc machine-ingest log --type=toss --player=<id>`** writes a toss row by hand, but passes **no payload**, so such a row names no room and no session. The watch must handle that row shape; the verification guide uses a `ddev wp eval` line to make a toss row *with* a session.

### The detection rule
**The evidence is the counter's reset, not the coins.** A coin pusher pays nothing on most
tosses, so "no coins fell" is an ordinary afternoon and can never be the signal. What the
machine's own controller does on *every* accepted toss is reset `sensor.coin` to 0 within
about two seconds. That reset is the machine saying "I acted".

**And the reset is only visible when the counter was not already 0.** The integration
writes only changed values, so a toss at `0` re-writes `0` and leaves no row — the spike
measured exactly this, twice, with real presses. Three outcomes follow, and the watch
names them:

| The counter at the toss | Anything in the window? | What it means | What is written |
|---|---|---|---|
| any | yes — a reset or a rise | the machine acted | nothing |
| **> 0** | **no** | **the machine answered 200 and did not act** — the counter should have reset within ~2 s and did not | **one `toss_no_movement` row** |
| 0 (or unreadable) | no | not observable — a reset to 0 leaves no trace | nothing; counted as `unconfirmed` |

The third row is Question 1. Recording it as a fault would put a record on most ordinary
tosses and the record would stop being evidence — the same defect as an alarm that goes
off every evening, which the sprint's goal forbids in as many words.

**It runs on the poll pass, not in the toss request.** The window is ~30 s of wall clock;
a player is not kept waiting for it, and the money path is untouched. Each pass judges the
tosses whose window has closed, oldest first, from `pc_realtime_toss_cursor`.

**A read that fails holds the cursor** — invariant 7's rule, for the same reason: Home
Assistant keeps ten days of history, so a toss judged late is judged correctly, while a
toss judged on a failed read would be judged wrong. A toss older than
`pc_realtime_toss_max_age_seconds` is past what history can answer and is retired
**audited**, never silently.

### Tasks (ordered)
- [x] **1 — What `core` needs to hold the record.** `app/utils/machine-events.php`: a `TYPE_TOSS_NO_MOVEMENT = 'toss_no_movement'` constant (added to `type_values()`), and `since( string $created_after, string $created_until, ?string $event_type, int $limit ): array` — rows in a `created_at` half-open range, **oldest first**, mapped exactly as `recent()` maps them. `app/utils/machine-ingest-service.php`: `log_event()` passes `correlation_id` through from `$context`, as `settle()` already does. `app/utils/install-schema.php`: `pc_realtime_toss_window_seconds` (30) and `pc_realtime_toss_max_age_seconds` (604800), `DB_VERSION` `1.14.0` → `1.15.0`. **Touches shared code (`core`): all three files, additive only — a new constant, a new read method, one pass-through key, two `add_option` lines and the version bump; consuming features: `core`, `realtime`, `stripe`.** `DATA-MODEL.md` (the new event type, the two options, the runtime cursor, the two audit types, the version row, and the `correlation_id` line that says nobody supplies it) in the same commit. → `feat(core): the machine event log can hold a toss that moved nothing`
- [x] **2 — The watch.** `app/realtime/toss-watch.php` — `Realtime_Toss_Watch::run( bool $dry_run = false ): array` returning `{judged:int, moved:int, unconfirmed:int, recorded:int, expired:int, stopped:?string}`. Unconfigured Home Assistant ⇒ `stopped: 'unconfigured'`, silent. Otherwise: up to **5** toss rows per pass (a constant — plumbing, not a business value), `created_at` after the cursor and at least `pc_realtime_toss_window_seconds` old, oldest first; each judged from one `get_state_history()` call over `[toss − 2 s, toss + window]`; the cursor advances only past a toss actually judged. A read failure audits `machine_toss_read_failed` and stops the pass; a toss past `pc_realtime_toss_max_age_seconds` audits `machine_toss_expired` and is retired unjudged. `--dry-run` judges, reports and writes nothing. `bootstrap.php`: one `require_once`. `PROJECT-TREE.md` and `ARCHITECTURE.md` (the new file in both trees, and the watch in the alerting flow) in the same commit. → `feat(realtime): a toss the machine did not act on is written down`
- [x] **3 — The record itself, and the pass that runs the watch.** In task 2's class: the `toss_no_movement` row, written through `Machine_Ingest_Service::log_event()` with `event_key = 'toss:{id}:no-movement'` (so a rewound cursor can never double-record), `user_id` = the player, `machine_id` from the toss row, `correlation_id` = the session id, and a payload carrying `{room_id, session_id, toss_event_id, toss_at, window_seconds, window:{from,to}, coin_sensor, before, after, changes}` — the player, the session, the toss event and **the sensor readings on both sides**, which is what the step asks the record to settle a dispute with. `machine-poller.php`: `Realtime_Toss_Watch::run( $dry_run )` immediately after the outage watch, inside the try and outside the guards; `toss` added to `run()`'s shape and to `pc_realtime_poll_last_run`; `sensor_entity()` made public. `machine-poll-command.php`: a `tosses:` line in the pass output and in `--status`, mirroring `relay:` and `machine:` — without it a host with no shell cannot see that the watch is running. *(Executed note: `sensor_entity()` was made public in task 2 instead — the watch's own file does not run without it.)* `DOMAIN.md` (the operator's term and rule) in the same commit. → `feat(realtime): the poll pass judges the tosses whose window has closed`
- [x] **4 — The checks.** `tests/realtime-toss.php`, listed below. → `test(realtime): a toss that moved nothing, and the three outcomes`
- [x] **5 — The feature's own record.** `docs/features/realtime/FEATURE.md` — the watch in Interfaces, the options and the cursor under Data, and the shared-code delta for the three `core` files task 1 touches. → `docs(realtime): the toss watch in the feature record`

### Files to create/change
**`backend/` — created**
- `wp-content/themes/pc/app/realtime/toss-watch.php`
- `wp-content/themes/pc/tests/realtime-toss.php`

**`backend/` — changed**
- `wp-content/themes/pc/app/realtime/bootstrap.php` — one `require_once`.
- `wp-content/themes/pc/app/realtime/machine-poller.php` — one call, one key in the result and in `pc_realtime_poll_last_run`, `sensor_entity()` private → public.
- `wp-content/themes/pc/app/realtime/machine-poll-command.php` — one reporting line in two places.
- `wp-content/themes/pc/app/utils/machine-events.php` — **shared (`core`)**, additive: one constant, one read method.
- `wp-content/themes/pc/app/utils/machine-ingest-service.php` — **shared (`core`)**, additive: `correlation_id` passed through in `log_event()`.
- `wp-content/themes/pc/app/utils/install-schema.php` — **shared (`core`)**, additive: two option defaults, `DB_VERSION` `1.14.0` → `1.15.0`. No table change; the bump is what makes `install_default_options()` run again, as 1.10.0–1.12.0 and 1.14.0 did.

**Not changed, and worth saying so:** `RoomQueueController.php` — the toss path is not touched
at all, which is the point: nothing this step adds can cost a player a coin or a
millisecond. Also `machine-service.php`, `queue-service.php`, `wallet-service.php`, every
other controller, and anything under `frontend/src/` or `admin/src/`.

### Tests to write
`backend/wp-content/themes/pc/tests/realtime-toss.php`, a WP-CLI `eval-file` script with
the DDEV guard every other script carries, an `$audit_floor` and a machine-events floor
captured at start-up, before/after row counts printed for every table it can touch
(`LEARNINGS.md` 2026-09-21 ×3 — a script that cleans up its fixtures and not the rows its
code writes), and Home Assistant answered by `pre_http_request` so the counter's history
is a variable each check sets.

1. **A toss followed by a rise is silent.** Counter 0 → 4 inside the window: `moved`, no record, cursor advanced.
2. **A toss followed by a reset is silent.** Counter 5 → 0 one second after the toss — the machine's own confirmation: `moved`, no record.
3. **A toss that moved nothing raises exactly one record.** Counter at 5 throughout the window: one `toss_no_movement` row, `user_id` = the player, `correlation_id` = the session, payload naming the room, the session, the toss event id, the window and `before: 5 / after: 5`.
4. **Exactly one, even with the cursor rewound.** Re-running the pass judges nothing; rewinding `pc_realtime_toss_cursor` and re-running produces **no second row** — the `event_key` refuses it.
5. **The unobservable case is not a fault.** Counter at 0 with nothing in the window: no record, counted `unconfirmed` (Question 1's behaviour, asserted rather than assumed).
6. **A toss inside its window is not judged yet.** A toss 5 s old with a 30 s window: `judged: 0`, cursor unmoved; it is judged on a later pass.
7. **The window comes from configuration.** `pc_realtime_toss_window_seconds` = 5 makes that same toss judgeable, and the history call's own window moves with it. The check reads the behaviour, not the constant.
8. **A failed read holds the cursor.** Home Assistant answering an error: nothing recorded, cursor unmoved, one `machine_toss_read_failed`; the next pass with a working read judges the same toss.
9. **A toss past the max age is retired, not judged.** Older than `pc_realtime_toss_max_age_seconds`: no record, one `machine_toss_expired`, cursor past it.
10. **A toss with no payload is handled.** The shape `wp pc machine-ingest log --type=toss` writes: no room, no session — the record carries nulls and `correlation_id` NULL rather than failing.
11. **`--dry-run` writes nothing.** Judged and reported, no row, no audit, cursor untouched.
12. **Unconfigured Home Assistant is silent.** `stopped: 'unconfigured'`, nothing written. *(The `PC_MACHINE_TOKEN`-absent branch is the one `LEARNINGS.md` 2026-09-21 says a test script cannot reach — it must define the constant to run. The check drives the same guard through `Machine_Service::is_configured()` and the docblock names the limit, as `tests/realtime-outage.php` does.)*
13. **It never touches money, and it never blocks the pass.** Wallet, coin-lot and transaction row counts identical before and after; and `Machine_Poller::run()` credits a stubbed payout byte-for-byte the same with the watch recording and with it silent.
14. **Cleanup.** Tosses, records, rooms, users, options and the audit rows above the floor all removed, with before/after counts printed.

### Docs to update
- `docs/DATA-MODEL.md` — the `toss_no_movement` event type in the `wp_pc_machine_events` block, `pc_realtime_toss_window_seconds`, `pc_realtime_toss_max_age_seconds`, the runtime `pc_realtime_toss_cursor`, the two new audit types in the realtime row, `1.15.0` in the version table, and the `correlation_id` bullet that currently says no caller supplies it. Task 1.
- `docs/DOMAIN.md` — the operator's term and the rule: a toss the machine did not act on is written down against that player and turn, and is not the same thing as a toss that paid nothing. Task 3.
- `docs/ARCHITECTURE.md` and `docs/PROJECT-TREE.md` — the new file in both trees and the watch in the alerting flow. Task 2. *(On the list deliberately: `LEARNINGS.md` 2026-09-21 records that the tree maps have no owner in the close checklist.)*
- `docs/features/realtime/FEATURE.md` — Interfaces, Data, and the shared-code delta. Task 5.
- **Not touched:** `CONTRACTS.md` (no endpoint, payload or error code changes — the ingest endpoint's `type` enum is **not** widened, deliberately: an outside caller must not be able to file a toss or a no-movement finding), `DESIGN.md` and `FEATURE.md` → UI (no screen), `TECH-STACK.md` (no dependency), `DECISIONS.md` (nothing is re-decided — the channel was Step 1's entry and the sensor model is the spike's; if Question 1 is answered against the recommendation, the answer becomes an entry), `ROADMAP.md` (**Phase 7 §4 → `[done]` belongs to Step 3**, the step that finishes the item).

### Checks
- **ANTI-PATTERNS:** none violated. Home Assistant is reached only through `Machine_Service`; no `ENUM` — the new type is a class constant on a `VARCHAR(32)` column; no money column is touched anywhere (the watch never credits, debits or refunds); `$wpdb` is never expected to throw — the only writes are `update_option`, `Audit_Log::record` and `Machine_Event_Log` through its existing `record_result()`, which already reports a failed insert honestly; no float; no machine payout in the ledger; no REST route, so no `permission_callback` question; nothing is hard-deleted; **no new cron** — the watch rides the pass that already exists; the window and the max age are operator-tunable options with defaults in `Install_Schema`, never constants.
- **Docs vs reality:** mismatch, three, all folded into tasks or resolved here.
  1. **The step's premise is narrower than it reads.** "After a successful toss, the coin sensor is expected to move within a bounded window, as Sprint 1 Step 2 measured" — what Sprint 1 Step 2 actually measured is that a toss at a counter of `0` moves **nothing** and leaves no trace, twice, with real presses. `DECISIONS.md` outranks a sprint step on a fact, so the detection rule above is written from the spike; what is left over is a product choice, and it is Question 1 rather than a silent narrowing.
  2. **The step's manual verification cannot be run as written.** It says "replay a toss event … through the Sprint 1 ingest endpoint", but `POST /machine/events` accepts `coins_dropped | bonus | relay_closed` and has never accepted `toss` — a toss row is written by the play endpoint, and by `wp pc machine-ingest log --type=toss`, which supplies no room or session. The guide will carry a paste-able `ddev wp eval` block that writes a toss with a session, and a stubbed history for the window. Same class as `LEARNINGS.md` 2026-09-18 ("a sprint's manual verification named local data that nothing creates"), caught before any code again.
  3. **The step asks for a record and never for a notification**, while the sprint goal is about an operator being told. Taken literally, as the plan rule requires: this step writes the record and sends nothing. `Realtime_Alerts` exists from Step 1 and one call site is all a later step or an `/adhoc` would need — worth knowing, not worth adding uninvited.
  **Observations that change no task:** the carried items are unchanged — the late-payout handover (which this step touches nothing of, though its records will show it), the two unassigned `BACKEND-REVIEW.md` §10 bullets, `settle()`'s unchecked `mark()`, the stale `TECH-STACK.md` eval-script list, the two test scripts that leave a row behind per run, and the fifth copy of the schedule-rules query. All `/adhoc`.
- **Design:** n/a — no screen, component, token or string. The sprint's Fixed decisions forbid a new operator screen.
- **Check command:** `backend/bin/check` (`docs/TECH-STACK.md` → Check command). DDEV is running, so stage 2 executes all thirteen `tests/*.php` including the new one. `frontend/bin/check` is not run: `frontend/` is not touched.
- **Not locally verifiable:** **whether a real toss on the real machine is judged correctly.** Everything above is checked against a stubbed Home Assistant, and the stub is built from the spike's recorded rows — but no toss has ever been timed against this code. The one real run that verifies it is **a venue-hours toss with coins in the machine**, which the step's own manual verification names: it must produce no record. Until then the window (30 s) is a desk guess sized from the spike's 1–12 s to first coin and ~20 s of counting, and it is an option precisely so it can be widened without a deploy.

### Questions / ambiguities

**Question 1 — A toss whose counter was already `0` cannot be judged. Is that a record or a silence?** The step says "a toss followed by no move raises exactly one record". The spike says a toss at `0` re-writes `0`, which Home Assistant does not record at all — so "no move" covers two different things: *the machine did not act* (the counter was non-zero and did not reset) and *we cannot tell* (the counter was already zero). In a coin pusher most tosses pay nothing, so after one fruitless toss the counter sits at `0` and stays there: the second case is the common one.
**Resolved: approved as recommended — (a) a `toss_no_movement` row is written only when the counter was non-zero at the toss and did not reset; the unobservable case is counted, not recorded.**
- **(a) — recommended. Record only the case that is evidence; count the rest.** A `toss_no_movement` row is written only when the counter was non-zero at the toss and did not reset. The unobservable case is counted as `unconfirmed` in the pass summary and on `wp pc machine-poll --status`, so it is visible and never silent, but it puts no row on the file. This keeps the record meaning one thing — "the machine answered 200 and did not act" — which is the only version that settles a dispute. *The tasks above are written to this.* Honest cost: the check is blind on a run of fruitless tosses, and that blindness is a property of the machine, not of the code.
- **(b) Record every toss with no movement, as the step reads literally.** One rule, no sub-cases, and nothing is ever missed. But at this venue's rhythm most tosses would carry a record, the type would become the most common row in `wp_pc_machine_events`, and a support answer built on it would be wrong more often than right — the alarm-every-evening failure, moved into a table.
- **(c) Record both, distinguished by an `outcome` field in the payload.** Everything is on file and a dispute can be answered either way. It costs the same row volume as (b) and asks whoever reads the table to know which outcome means what; the ops dashboard that would make that legible is explicitly a future feature, not this sprint.

---

## Plan — Sprint 3, Step 3: Withdrawals piling up   (status: approved, in progress)

### Branch
`realtime/sprint-3-withdrawals` ← `realtime/sprint-3` ← `main`
(git model in root `CLAUDE.md`. Read live at plan time: the sprint branch is docs `c466ca4`
and backend `44128410`, Steps 1 and 2 both merged into it. The same branch name in
`backend/` and the docs repository; `frontend/` (`main` `2d525d4`) and `admin/` are **not
touched** — no screen, no REST response, no error code. **This is the last step of the
sprint**, so its `/close-step` merges `realtime/sprint-3` into `main` in both repositories;
the merge is local, and the backend deploy happens when the user pushes `main`.)

### What is settled before this step starts
- **Thresholds are configuration**, with defaults in `Install_Schema` — the sprint's Fixed decisions and root `CLAUDE.md` core rule 3.
- **Alerting never blocks a money path** — `FEATURE.md` → Invariants #2. The watch runs on the poll pass, next to the crediting path, so every failure is audited and swallowed.
- **The channel is email through `Realtime_Alerts::send()`** — `DECISIONS.md` 2026-09-21, decided in Step 1 before any alert existed, precisely so the third alarm would be a call site rather than a decision. This step adds the second call site; it does not re-open the channel.
- **Operator-facing events live in `wp_pc_auth_audit_log`** — the sprint's Fixed decisions. No third store, no new table.
- **No new operator screen** — the sprint's Fixed decisions. None is needed: the admin SPA's **Withdrawals** screen (`admin/src/views/WithdrawalsView.vue`) already lists the pending queue and approves or rejects it, which is exactly what clears the backlog this step alerts on.
- **A player may have one withdrawal request open at a time** — `DOMAIN.md`. So a count of pending withdrawals is also a count of players waiting to be paid, which is what makes a count worth alerting on at all.
- **Every money mutation goes through `Wallet_Service`** — root `CLAUDE.md` invariant 3 and the `TECH-STACK.md` anti-pattern. **This step writes nothing to a money table**; it reads one aggregate.

### What already exists, and what it gives this step
- **A pending withdrawal is a row in `wp_pc_transactions`** with `type = 'withdraw'` and `status = 'pending'` (`Wallet_Service::TYPE_WITHDRAW` / `STATUS_PENDING`). It is created by `Wallet_Service::request_withdrawal()` (`wallet-service.php:368`), which FIFO-debits the coins inside a transaction, and it leaves `pending` only through `AdminWithdrawalController::approve` → `completed` or `reject` → `refunded` (`AdminWithdrawalController.php:99,113`). So "pending withdrawals" is a single, exact query and needs nothing invented.
- **The table is indexed for it.** `wp_pc_transactions` carries `KEY status (status)` (`install-schema.php`), so the count and the oldest row are cheap even once the ledger is long.
- **`created_at` is written with `current_time( 'mysql' )`** — the site's *local* time, not UTC (`wallet-service.php:396`). An age computed against `time()` would be wrong by the site's offset. The idiom already in this codebase for exactly this is `Queue_Service::prune()` (`queue-service.php:735`): `gmdate( 'Y-m-d H:i:s', strtotime( current_time( 'mysql' ) ) - $seconds )` — both sides of the comparison pass through the same clock. This step reuses it rather than inventing a second way. **The local install is on UTC** (`gmt_offset` `0`), so a mixup would be invisible in the checks; section 8 of the test script therefore switches the site timezone to `Europe/Kyiv` for one section and asserts a withdrawal created *now* still reports an age of seconds, not hours.
- **The poll pass already carries three watches.** `Machine_Poller::run()` calls `Realtime_Relay_Watch`, `Realtime_Outage_Watch` and `Realtime_Toss_Watch` in that order, **inside the `try` and before the machine-id / ingest-secret guards** (`machine-poller.php:90-113`), so a transport that cannot run stops none of them. The event is scheduled unconditionally on `init` (`ensure_scheduled()`, `machine-poller.php:396`) — it does not depend on a machine id — so a fourth watch there ticks on any install. **No second cron**, which is also what the `TECH-STACK.md` anti-pattern on cron jobs points at, and nothing new to deploy.
- **`Realtime_Alerts::send()`** (`alerts.php`) — plain-text email to `pc_realtime_alert_email` → `pc_support_email` → `admin_email`, returning a bool on every path including the ones that caught a `Throwable`. DDEV runs Mailpit at `https://pusher-coin.ddev.site:8026`, so the whole channel is checkable locally.
- **`wp pc machine-poll` and `--status`** already print one line per watch (`machine-poll-command.php:71-73,101-103`) and `pc_realtime_poll_last_run` already carries a per-watch summary, which is how a host with no shell sees any of this. The fourth watch gets the same line, in the same two places.

### The alert rule
**Two thresholds, either one fires.** More than `pc_realtime_withdrawal_alert_count` pending
withdrawals, **or** an oldest pending one older than
`pc_realtime_withdrawal_alert_age_seconds`. Both are "exceeds", strictly greater, as the
step words it. Setting either to `0` switches that threshold off; both `0` switches the
watch off entirely and it reports `stopped: 'disabled'` — the same shape as every other
"the empty configuration *is* the off switch" in this codebase.

**Two gates, and an alert needs both.** The step says two things in one breath — "alert at
most once per period" and "re-alert only after dropping below and crossing again" — and the
honest reading is that both hold at once:
- **the latch** — one alert per crossing. Once alerted, the watch stays quiet however long
  the backlog lasts, until the count and the age both fall back under their thresholds. That
  clearing is recorded (`withdrawal_backlog_cleared`) and is what arms the next alert;
- **the period ceiling** — at most one withdrawal alert per
  `pc_realtime_withdrawal_alert_period_seconds`, counted from the last one **sent**, and
  therefore remembered across a clearing. Without this, a backlog hovering on the threshold
  could clear and re-cross all afternoon and send an email each time, which is the stream of
  notifications the step exists to prevent.

The alternative reading — the period as a *reminder*, re-alerting while the backlog persists
— is ruled out by the step's own second sentence ("re-alert **only** after dropping below and
crossing again"), so nothing reminds and nothing escalates. Where the two rules disagree the
watch stays quiet, which is the same direction every judgement in Step 2 was taken in.

**The latch records the decision to alert, not the delivery.** As in the outage watch, it is
set whether or not `wp_mail` accepted the message; `Realtime_Alerts` audits a failed send as
`operator_alert_failed`. A mailer that is refusing everything must not turn into one attempt
a minute for as long as the backlog lasts.

**What the email has to be worth reading on a phone**, which is the standard Step 1 set: how
many are waiting, how long the oldest has waited, how much money that is in total, which
threshold was crossed, and that approving or rejecting them in the admin **Withdrawals**
screen is what clears it. No link is invented — the admin SPA has no deploy target and no
URL to name (`ROADMAP.md` Phase 8).

### Tasks (ordered)
- [x] **1. What "piling up" is, and the three settings that decide it.** Add
  `Wallet_Service::pending_withdrawal_summary(): array{count:int, oldest_at:?string, oldest_age_seconds:?int, total_money:string}` — one aggregate `SELECT COUNT(*), MIN(created_at), COALESCE(SUM(amount_money),'0.00')` over `type = withdraw AND status = pending`, with the age taken through the `current_time( 'mysql' )` clock as `Queue_Service::prune()` does, and the money kept as a decimal string. Seed
  `pc_realtime_withdrawal_alert_count` (10), `pc_realtime_withdrawal_alert_age_seconds` (86400) and `pc_realtime_withdrawal_alert_period_seconds` (86400) in `install_default_options()`, and bump `DB_VERSION` `1.15.0` → `1.16.0` — no table change; the bump is what makes the defaults appear on an existing install. Create `tests/realtime-withdrawals.php` with its first two sections (the aggregate over a mixed ledger; the clock). Update `docs/DATA-MODEL.md` in the same commit: the `1.16.0` row, the three option rows and the runtime state option.
  **Touches shared code (`core`) — may affect other features:** `app/utils/wallet-service.php` and `app/utils/install-schema.php`, both purely additive (a new read-only method; three `add_option` lines). Consumers of `Wallet_Service`, checked by grep rather than by the audit (`LEARNINGS.md` 2026-09-21): `WalletController`, `TransactionsController`, `AdminWithdrawalController`, `AdminTopupController`, `RoomQueueController`, `Machine_Ingest_Service`, `Stripe_Webhook_Controller` and `wp pc machine-ingest` — **no existing signature changes**, so none of them is affected. `stripe` reads the same table for top-ups and is untouched: the new method filters on `type = withdraw`.
  → commit `feat(core): a reading of the pending-withdrawal backlog, and what counts as too much`
- [x] **2. The fourth watch, and the operator's email.** New `app/realtime/withdrawal-watch.php` — `final class Realtime_Withdrawal_Watch` with `run( bool $dry_run = false ): array{pending:int, oldest_age:?int, breached:bool, alerted:bool, cleared:bool, stopped:?string}`, the latch-and-period rule above, state in `pc_realtime_withdrawal_alert_state` (written only when something changes, `autoload` false), audit rows `withdrawal_backlog_alerted` and `withdrawal_backlog_cleared`, and the notification through `Realtime_Alerts::send()`. Wire it: one `require_once` in `app/realtime/bootstrap.php`; the call in `Machine_Poller::run()` after the toss watch, inside the `try` and before the guards, with `'withdrawals'` in the result shape and in the `pc_realtime_poll_last_run` summary; a `withdrawals:` line in both `wp pc machine-poll` and `--status`. Sections 3–10 of `tests/realtime-withdrawals.php` (below). Update `docs/ARCHITECTURE.md` (the alerting flow paragraph and two tree lines), `docs/PROJECT-TREE.md` (the two new files) and `docs/features/realtime/FEATURE.md` (Data, Interfaces, a ninth invariant, and the shared-code delta notes) in the same commit.
  **Touches shared code (`core`):** none in this task — `app/realtime/` only.
  → commit `feat(realtime): tell the operator when withdrawals pile up, once per pile`
- [x] **3. The operator's own words, and the status board.** `docs/DOMAIN.md` — the term **Withdrawal backlog** in the glossary and one rule saying what the operator is told and when. `docs/DECISIONS.md` — one entry for the two-gate rule (a crossing *and* the period, never a reminder) and for putting the watch on the poll pass rather than a cron of its own. `docs/ROADMAP.md` — Phase 7 §4 **Ops alerts** `[todo]` → `[done]` with the three alarms named, the tracking matrix row for item 17, and Phase 5 re-read against reality as the step asks: §7 **Machine-event channel** is `[todo]` but was built by `realtime` Sprint 2, so it is re-tagged with the honest caveat that no Ably account has ever existed, and the phase's exit criteria ("Met except for 'real-time' in the browser") is rewritten to match.
  → commit `docs(realtime): withdrawals piling up, and the phases this sprint finishes`

### Files to create/change
**Create (backend):**
- `backend/wp-content/themes/pc/app/realtime/withdrawal-watch.php` — `Realtime_Withdrawal_Watch`
- `backend/wp-content/themes/pc/tests/realtime-withdrawals.php` — the checks

**Change (backend):**
- `app/utils/wallet-service.php` — **shared `core`, additive**: `pending_withdrawal_summary()`
- `app/utils/install-schema.php` — **shared `core`, additive**: three options, `DB_VERSION` → `1.16.0`
- `app/realtime/bootstrap.php` — one `require_once`
- `app/realtime/machine-poller.php` — the fourth watch in `run()`, the result shape, the `last_run` summary
- `app/realtime/machine-poll-command.php` — the `withdrawals:` line in the pass output and in `--status`

**Change (docs):** `DATA-MODEL.md`, `ARCHITECTURE.md`, `PROJECT-TREE.md`, `DOMAIN.md`, `DECISIONS.md`, `ROADMAP.md`, `features/realtime/FEATURE.md`, this plan file.

**Not changed:** `frontend/`, `admin/`, `CONTRACTS.md` (no endpoint, payload or error code), `DESIGN.md` and `FEATURE.md` → UI (no screen), `TECH-STACK.md` (no dependency).

### Tests to write
`backend/wp-content/themes/pc/tests/realtime-withdrawals.php`, a WP-CLI `eval-file` script
in the shape of `tests/realtime-outage.php` (DDEV guard, `$check`, before/after row tally,
everything restored in a `finally`). Mail is stubbed with `pre_wp_mail`, so nothing leaves
the box and the message body is a variable the checks read. Fixtures are **real money
paths**: a throwaway user per pending withdrawal (`DOMAIN.md` allows one open request per
player), each credited a lot through `Wallet_Service::credit_lot()` and requesting through
`Wallet_Service::request_withdrawal()`. Only a fixture's `created_at` is set directly, to
age it — no service method can make a row older — and the script says so where it does it.
**Cleanup names the tables the code under test and its fixtures write** (`LEARNINGS.md`
2026-09-21), following `tests/wallet-rollback.php`, which is the one existing script that
deletes money rows: `pc_transactions`, `pc_coin_lots`, `pc_wallets` for the throwaway users,
the users themselves, `pc_auth_audit_log` above the floor, and every option saved at the top.

1. **The reading.** A ledger holding pending withdrawals, an approved one, a rejected one and a pending *top-up* → the summary counts only pending withdrawals, `oldest_at` is the oldest of those, `total_money` is their sum as a decimal string, and an empty backlog reports `0` / `null` / `'0.00'`.
2. **The clock.** With the site timezone switched to `Europe/Kyiv`, a withdrawal created now reports an age of seconds, not of the offset — the check the UTC-only local install cannot make by itself.
3. **Quiet below both thresholds.** Pending withdrawals under the count and under the age: no mail, no audit row, no state written.
4. **Crossing the count alerts exactly once.** One more pending withdrawal than the threshold → one mail, one `withdrawal_backlog_alerted`, and the body names the count, the age of the oldest and the total.
5. **Staying above is silent.** Three further passes with the backlog unchanged, and with one *more* withdrawal added: still one mail in total.
6. **Crossing the age alone alerts**, with the count left under its threshold — one backdated pending withdrawal, and the body says it is the age that was crossed.
7. **Clearing is recorded.** Approving the backlog through `Wallet_Service::approve_withdrawal()` drops it under both thresholds → `withdrawal_backlog_cleared`, no mail.
8. **The period ceiling holds.** A second crossing inside the period sends nothing, though the latch has been cleared.
9. **And releases.** With `last_alert_at` rewound past the period, the same second crossing sends exactly one.
10. **Configuration decides.** Changing `pc_realtime_withdrawal_alert_count` alone flips the verdict on an unchanged backlog; both thresholds at `0` report `stopped: 'disabled'` and send nothing however many are pending.
11. **An alert can never cost anything.** A `pre_wp_mail` that returns false still latches and is audited `operator_alert_failed`; one that *throws* leaves `run()` returning normally, and a following `Machine_Poller::run()` still completes its own pass.
12. **`--dry-run` reads and writes nothing** — no mail, no audit row, no state option.
13. **Cleanup:** every counted table back to its starting count.

**Test-critical zones** (project profile): money — this step's code performs **no** wallet
mutation, and section 1's fixtures go through `Wallet_Service` precisely so nothing in the
check bypasses it; the checks assert that a pass leaves `pc_wallets`, `pc_coin_lots` and
`pc_transactions` row counts and balances unchanged.

### Docs to update
- **`docs/DATA-MODEL.md`** — the `1.16.0` row in the version table; three option rows in *Realtime — operator alerts* (`pc_realtime_withdrawal_alert_count`, `…_age_seconds`, `…_period_seconds`) plus `pc_realtime_withdrawal_alert_state` as a runtime-written option; and `withdrawal_backlog_alerted` / `withdrawal_backlog_cleared` in the audit event-type table, as a new `realtime — the withdrawal backlog` row. *(Named explicitly because this table enumerates and has no owner in the close checklist — `LEARNINGS.md` 2026-09-21.)*
- **`docs/ARCHITECTURE.md`** — a paragraph after the toss-watch one for the fourth watch, and the two new files in the tree.
- **`docs/PROJECT-TREE.md`** — the two new files. *(Same reason: an enumerating document with no owner.)*
- **`docs/DOMAIN.md`** — **Withdrawal backlog** in the glossary, and one rule in the operator's terms.
- **`docs/DECISIONS.md`** — the two-gate alert rule and the choice of the existing poll pass over a cron of its own.
- **`docs/ROADMAP.md`** — Phase 7 §4 → `[done]`; the tracking matrix row for item 17; Phase 5 §7 and the phase's exit criteria re-read against what Sprint 2 actually shipped, as the step asks.
- **`docs/features/realtime/FEATURE.md`** — Data (the four options), Interfaces (a `Realtime_Withdrawal_Watch` bullet), a ninth invariant (one alert per crossing and never more than one per period), and the shared-code notes for `wallet-service.php` and `install-schema.php`.
- **`SPRINT-3-PLAN.md`** — checkboxes and status.

### Checks
- **ANTI-PATTERNS:** none violated. No `ENUM` — the new audit types are strings in an existing `VARCHAR(64)` column and the thresholds are options; **no money column is touched** — the watch reads one aggregate and writes nothing to `pc_transactions`, `pc_wallets` or `pc_coin_lots`; `$wpdb` is not expected to throw — the only writes are `update_option` and `Audit_Log::record`, and the aggregate read is checked for `null`; **no float** — the total stays a decimal string from MySQL to the email body; no machine payout in the ledger; no REST route, so no `permission_callback` question; nothing is hard-deleted; **no new cron job** — the watch rides the pass that already exists; no secret in an option; **no hardcoded operator-tunable value** — all three thresholds are options with defaults in `Install_Schema`; no new plugin, no new dependency.
- **Docs vs reality:** mismatch, three, all resolved in the tasks above.
  1. **`wp_pc_transactions.created_at` is local time, and the local install is UTC.** Nothing in `DATA-MODEL.md` → Conventions says which clock a `DATETIME` is in, and `current_time( 'mysql' )` is the site's, not UTC. A check written on this machine cannot tell the two apart (`gmt_offset` is `0`), so the age arithmetic reuses the existing `Queue_Service::prune()` idiom and section 2 flips the site timezone to make the trap reachable. Same class as `LEARNINGS.md` 2026-09-21 ("a plan listed a check the test harness itself makes impossible to write"), caught at plan time and made reachable rather than dropped.
  2. **The step's manual verification says "Approve them in the admin Withdrawals screen".** That screen exists and works, but it is behind the admin SPA's two-step sign-in, whose code arrives by email — and this agent does not sign into it or place a token in a browser (`LEARNINGS.md` 2026-09-18). So the guide will carry both: the `ddev wp eval` route the agent can run and verify, and the admin-screen route for the user, marked as the user's own check. Nothing in the tasks depends on the answer.
  3. **The step's Docs-to-update asks for "Phase 5's exit criteria re-read against reality", and reality has moved twice.** Phase 5 §7 (the machine-event channel) still reads `[todo]` although Sprint 2 built it, and the exit criteria still says the SPA polls. Both are corrected in task 3 with the caveat that no Ably account has ever existed, so nothing has been observed end to end. This is the step's own instruction, not scope growth.
  **Observations that change no task:** the carried `/adhoc` items are unchanged — the late-payout handover, the two unassigned `BACKEND-REVIEW.md` §10 bullets, `settle()`'s unchecked `mark()`, the stale `TECH-STACK.md` eval-script list, the two test scripts that leave a row behind per run, and the fifth copy of the schedule-rules query. One more joins them: `AdminWithdrawalController::list_withdrawals` queries `wp_pc_transactions` directly rather than through `Wallet_Service`, which is the drift this step's new method deliberately does not extend.
- **Design:** n/a — no screen, component, token or string. The sprint's Fixed decisions forbid a new operator screen, and the one that clears a backlog already exists.
- **Check command:** `backend/bin/check` (`docs/TECH-STACK.md` → Check command). DDEV is running, so stage 2 executes all fourteen `tests/*.php` including the new one. `frontend/bin/check` is not run: `frontend/` is not touched. After the step's merge, the check must also exit 0 on `realtime/sprint-3` and then on `main`, because this close merges the sprint.
- **Not locally verifiable:** **whether `wp_mail` actually delivers from the production host.** Locally Mailpit catches everything, and the shared host's own mail path has never been confirmed — the open question Step 1 raised and Step 2 carried. The one real run that verifies it is **the next deploy from `main` followed by a real backlog crossing the threshold** (or, cheaper, the test support ticket Step 1's guide already names). Nothing in this step can close it.

### Questions / ambiguities
none.
