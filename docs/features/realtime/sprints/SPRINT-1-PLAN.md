# Sprint 1 — step plans (`realtime`)

<!-- Written by /plan-step, one section per step. A section's status moves
     awaiting approval → approved, in progress → implemented, awaiting close →
     closed. Never edit another step's section. -->

## Plan — Sprint 1, Step 1: Delta-audit   (status: closed)

### Branch
`realtime/sprint-1-delta-audit` ← `realtime/sprint-1` ← `main`, merged back `--no-ff` by `/close-step`.

**Only the root docs repository.** This step changes one documentation file and no
code, so, per the git model ("the same branch name in every repository a step
touches"), `realtime/sprint-1` is cut in the root repository only. `backend/`,
`frontend/` and `admin/` get theirs from the first step that touches them (Step 3 for
`backend/`), as `stripe` Sprint 2 did with `admin/`. Nothing is pushed.

**Precondition for `/do-step` — the abandoned branches are gone.** `DECISIONS.md`
2026-09-18 "`realtime/sprint-1` is dropped" says the user deletes them; at plan time
all five still exist and are unmerged:

| Repository | Branch | Tip |
|---|---|---|
| root (docs) | `realtime/sprint-1` | `d6e5c05` |
| root (docs) | `realtime/sprint-1-spike-transport` | `ab5844d` |
| `backend/` | `realtime/sprint-1` | `8418a859` |
| `frontend/` | `realtime/sprint-1` | `a0d9cc8` |
| `admin/` | `realtime/sprint-1` | `28f0181` |

The user deletes them (from the project root, e.g. with the `!` prefix):

```bash
git branch -D realtime/sprint-1 realtime/sprint-1-spike-transport
git -C backend  branch -D realtime/sprint-1
git -C frontend branch -D realtime/sprint-1
git -C admin    branch -D realtime/sprint-1
```

`/do-step` checks `git branch --list 'realtime/*'` in all four repositories first and
**stops** if any of the five remains. It never deletes, renames, resets or reuses
them: the decision gives that to the user, and a stale `realtime/sprint-1` in an app
repository is exactly what a later step would otherwise branch from by mistake.

### Tasks (ordered)

- [x] **1. Cut the branches** — in the root repository only: `realtime/sprint-1` from
  `main` (`cfc3d75` at plan time), then `realtime/sprint-1-delta-audit` from it.
  → no commit

- [x] **2. The shared-code audit** — read `docs/features/core/FEATURE.md` and
  `docs/features/stripe/FEATURE.md` (Interfaces, Invariants) and the shared code below,
  then rewrite `FEATURE.md` → Fit into the host → **Shared code it depends on** as a
  per-file list. Each entry: the path (with `:line` where a line pins the behaviour),
  a one-line **why it matters here**, and the step(s) of Sprints 1–3 that touch it.
  All of it is owned by `core`; the list says so once, and that editing any entry is a
  plan task marked "touches shared code".
  - **Scope of the list** = the files the step names, plus every entry the current
    `FEATURE.md` line already names in general terms (each must become a concrete file
    — that is what the verification reads for), plus any other `core` file a step of
    Sprints 1–3 names. Concretely:
    - Backend (`backend/wp-content/themes/pc/`): `app/utils/machine-ingest-service.php`,
      `app/utils/machine-events.php`, `app/utils/queue-service.php`,
      `app/utils/machine-service.php`, `app/utils/cli/machine-ingest.php` (today's only
      producer), `app/rest-api/RoomQueueController.php` (the toss, the 423, the refund),
      `app/rest-api/AdminRoomController.php` (Step 3), `app/utils/install-schema.php`,
      `app/utils/rate-limiter.php`, `app/utils/audit-log.php`,
      `app/utils/room-schedule-calculator.php`, `app/utils/support-service.php`
      (Sprint 3 Step 1's email option), `functions.php` (the entry line).
    - Player SPA (`frontend/src/`): `stores/queue.js`, `stores/chat.js`,
      `components/PlaceBet.vue`, `components/UserControls.vue` (Sprint 2 Step 2 names
      it), `stores/wallet.js` (reached through `stores/queue.js:5,113`), and
      `RoomQueue.vue` / `RoomChat.vue` only if they hold behaviour a step changes
      (FEATURE.md → UI lists them as reused).
  - **Conflicts with the siblings' invariants** — a short list under the same heading,
    each naming the invariant, the file and the step that must answer it; "none" is
    written out for a sibling with none.
  - **Search each repository on its own** (`git -C backend grep …`,
    `git -C frontend grep …`), never a `grep -r` from the project root — the root
    ignores all three apps and returns a plausible, empty answer
    (`docs/LEARNINGS.md` 2026-09-17).
  - What inspection at plan time already shows, for `/do-step` to re-verify against
    the code and write down (not to take on trust):
    - `Machine_Event_Log::record()` returns 0 both for a duplicate `event_key` and for
      a failed insert (`machine-events.php:82-85`), and `settle()` reports any 0 as
      `duplicate: true` (`machine-ingest-service.php:137-145`) — so a database error
      looks like "already recorded". Step 4's replay answer sits on this.
    - `settle()` writes the event row, then credits, with no transaction around the
      two and `mark()`'s result unchecked (`machine-ingest-service.php:129-173`): a
      crash between them leaves a `recorded` row that every replay then treats as a
      duplicate. Step 5's "a restart credits nothing twice, loses nothing" sits on this.
    - Attribution resolves a machine to a room with `posts_per_page => 1` over
      `publish` **and** `draft` rooms, ignoring `available`
      (`queue-service.php:435-447`) — Step 3's "one machine, one active room" has to
      account for it, not just for the admin toggle.
    - `get_coin_count()` is documented as cumulative (`machine-service.php:60`) and
      `get_relay_closed()` reads `sensor.relay_on` through `normalise_truthy()`
      (`:75-82`, `:248-250`), both contradicted or unsettled by `DECISIONS.md`
      2026-09-18 points 6, 8, 9 — Steps 2 and 4.
    - Root `CLAUDE.md` invariant 8 / `core` invariant 2 ("`Machine_Service` is the only
      caller of Home Assistant") conflicts with one of Step 5's three options, the
      WebSocket worker, which would be a second client of Home Assistant outside
      WordPress; the HA-automation option makes Home Assistant call WordPress and does
      not. Step 2's `DECISIONS.md` entry has to address it if it picks the worker.
    - `core` invariant 3 (one open bet session per room) is asserted, not enforced
      (`queue-service.php:253-277`) — `BACKEND-REVIEW.md` §15, Sprint 2 Step 5.
    - `stripe`: no invariant conflict found so far — `realtime` never writes the ledger
      and never touches a transaction's status; it only calls
      `Wallet_Service::credit_lot()`. `stripe`'s own FEATURE.md claims an overlap on
      `stores/wallet.js` "(their Sprint 2)"; the current `realtime` sprint files never
      name that file, and the audit records the real path to it (via `stores/queue.js`).
  → docs commit `docs(realtime): delta-audit — the shared code realtime touches, and why`

- [x] **3. Entry point, check coverage, review items** — three more entries under
  `FEATURE.md` → Fit into the host:
  - **Entry point** — confirm against `functions.php:14-17` (the `Features` block
    holding `stripe`'s single `require_once`) and `app/stripe/bootstrap.php` (routes
    registered on `rest_api_init` inside the bootstrap, not through `app/rest-api.php`).
    Record where `realtime`'s line goes: under the same block, after `app/utils.php`
    has loaded, so `core`'s services and the `pc_machine_event_player` /
    `pc_machine_event_credited` hooks (`queue-service.php:496-497`) already exist.
    `docs/ARCHITECTURE.md:257-260` already states this rule and stays as is. SPA side:
    no entry until Sprint 2, and only in `frontend/`.
  - **Check coverage** — `backend/bin/check` lints every PHP file under the theme, so
    `app/realtime/` is covered the moment it exists, and it picks up any new
    `tests/*.php` without an edit; `frontend/bin/check` lints and builds all of `src/`.
    Record the gaps as they are: the money checks run only with DDEV up (otherwise a
    boxed `SKIPPED` and exit 0) and CI never runs them; a Step 5 artefact outside the
    theme — a worker, a cron line, an HA automation kept as documentation — is covered
    by no check command, so Step 5's plan must name its own.
  - **`admin/`** — confirm no step of Sprints 1–3 edits it. Record what inspection
    found: Step 3's check happens on the **Room form** (`RoomFormView.vue`), which
    already prints the server's `message` (`admin/src/stores/rooms.js`), so it needs
    no admin edit; but `MachineView.vue:156,160,168` hard-codes "Phase 5 Step 4 swaps
    this for real-time push", `sensor.coin` and "Relay closed (`sensor.relay_on`)" —
    labels Step 2 may prove wrong, with no step to change them (an `/adhoc` candidate,
    not added to any step). `admin/src/services/realtime.js` is named as a code path
    (root `CLAUDE.md` Features table, `ARCHITECTURE.md:53`, this FEATURE.md) but no
    step of Sprints 1–3 creates it — it is the Roadmap's "left for later" item.
  - **`BACKEND-REVIEW.md` §10, §12, §15** — for each item, and for each of §10's three
    bullets separately, the step that settles it. A step "settles" an item only when
    its text names that behaviour; a step whose work merely depends on it is written
    as "depends on it, does not name it"; an item no step names is written as
    **unassigned**, with "re-planning or `/adhoc`" — never quietly given to a step.
    Plan-time reading: §12 → Sprint 1 Step 3 (named); §15 → Sprint 2 Step 5 (named);
    §10 "a database error on a machine event is reported as a duplicate"
    (`machine-events.php:85`) → Step 4 depends on it, does not name it; §10 "a failed
    refund loses the coin" (`RoomQueueController.php:187-190`) and §10 "the 2-second
    timeout makes a slow toss free" (`machine-service.php:30`) → no step names them.
    If that holds, the audit lists it as a conflict with `FEATURE.md` → Purpose &
    scope, which puts all of §10 inside this feature.
  → docs commit `docs(realtime): delta-audit — entry point, check coverage, review items`

- [x] **4. The gate and the evidence** — run `backend/bin/check` (with DDEV up, so
  stage 2 executes rather than printing `SKIPPED`) and `frontend/bin/check` at the
  `main` HEAD of `backend/` and `frontend/` — the exact commits `realtime/sprint-1`
  will be cut from there. Record both exit codes and the stage-2 line; count
  `FEATURE.md`'s lines. → no commit

### Files to create/change
- `docs/features/realtime/FEATURE.md` — the **Fit into the host** section only (root docs repository).
- `docs/features/realtime/sprints/SPRINT-1-PLAN.md` — this file; status lines only, by `/do-step`.
- Nothing in `backend/`, `frontend/` or `admin/`.

### Tests to write
None. The step's **Tests** is "—", and it changes no code in a test-critical zone —
no wallet, webhook, token, permission or machine-event code. The evidence is task 4's
two check runs and the reading of the updated `FEATURE.md`.

### Docs to update
`docs/features/realtime/FEATURE.md` → Fit into the host (tasks 2–3). Nothing else:
`docs/ARCHITECTURE.md:257-260` already describes the bootstrap pattern the audit
confirms; no decision is taken, so `docs/DECISIONS.md` is unchanged; no file is added,
so `docs/PROJECT-TREE.md` is unchanged.

### Checks
- **ANTI-PATTERNS:** none violated — one documentation file changes, no code, no
  dependency, no money path, no route, no meta key.
- **Docs vs reality:** mismatches, each resolved without changing the step's work:
  1. The five abandoned branches still exist, although `DECISIONS.md` 2026-09-18 says
     the user deletes them → precondition of `/do-step` (Branch above).
  2. The step's verification says the checks exit 0 "on the untouched sprint branch";
     Step 1 creates no sprint branch in `backend/` or `frontend/` → the checks run at
     their `main` HEAD, which is the commit that branch will be cut from, so no code
     differs.
  3. Step 3's verification names the admin **Room list**; status and machine id are set
     on the **Room form** (`docs/DESIGN.md` → Screens) → recorded by task 3; `SPRINT-1.md`
     is not edited.
  4. `BACKEND-REVIEW.md` §12 cites `machine-service.php:38`, which is now `power_on()`;
     the attribution site the item is about is `queue-service.php:435-447`. §10's and
     §15's citations still match the code → task 3 records current lines.
  5. `stripe`'s FEATURE.md names a `realtime` overlap on `stores/wallet.js` that no
     current `realtime` sprint file names → recorded on `realtime`'s side (task 2);
     `stripe`'s document is not edited.
  6. `FEATURE.md` is 72 lines against its "≤80" guideline, and a per-file audit will
     pass it, as `stripe`'s did (96) → the detail is kept and the overshoot recorded
     for the close report, per `docs/LEARNINGS.md` 2026-09-17 (pending a playbook
     decision); no other section is trimmed to make room.
- **Design:** n/a — no screen; `design/` stays empty (`DECISIONS.md` 2026-09-15).
- **Check command:** `backend/bin/check` and `frontend/bin/check` (`docs/TECH-STACK.md`
  → Check command) — present on `main`, and both **exit 0 at plan time**: backend
  49 files linted and stage 2 executed (`stripe-client.php`, `stripe-webhook.php`,
  `wallet-rollback.php` all passed, not skipped); frontend lint and build OK. `admin/`
  has none, and this step does not touch it.
- **Not locally verifiable:** n/a — nothing is deployed or pushed.

### Questions / ambiguities
none

### Execution notes (for `/close-step`)
- **Commits** (root docs repository, branch `realtime/sprint-1-delta-audit`): `46fa55f`
  (the shared-code list and the conflicts), `99b6120` (entry point, check coverage,
  `admin/`, review items), plus this plan file. Nothing in `backend/`, `frontend/` or
  `admin/`; nothing pushed, nothing merged.
- **Precondition met.** The first `/do-step` run stopped: all five abandoned branches
  still existed. The user deleted them; the second run found `git branch --list
  'realtime/*'` empty in all four repositories.
- **`main` was `1cfa370`, not the `cfc3d75` the plan names** ("docs(realtime): update
  Sprint 2 plan and goal…"). *Corrected at close:* the reflog shows `main` moved at
  12:39:44, 21 minutes before the plan file was written (13:01:02), so the plan took
  `cfc3d75` from the session-start git snapshot rather than a live `git rev-parse main`.
  (The first version of this note said the sprint text "sat uncommitted on disk during
  planning" — a guess the reflog does not support; `docs/LEARNINGS.md` 2026-09-18.) Step 1's
  text on `1cfa370` is the text the plan was derived from, so the plan held.
  `realtime/sprint-1` was cut from `1cfa370`.
- **Gate evidence (task 4).** At `backend` `main` `5ebe9610` and `frontend` `main`
  `7210c59`, both clean: `backend/bin/check` exit 0 — 49 files linted, stage 2
  **executed** (`stripe-client.php`, `stripe-webhook.php`, `wallet-rollback.php` passed,
  not `SKIPPED`); `frontend/bin/check` exit 0 — lint and build OK. Same result before
  each of the two commits.
- **FEATURE.md is 123 lines**, against its ≤80-line guideline (Docs vs reality 6). The
  audit detail was kept and no other section was trimmed.
- **Found by the audit, not foreseen by the plan — for the close report:**
  1. **A late payout goes to the next player.** A player's last declared coin closes their
     turn and session on the spot (`queue-service.php:237-240`); the payout lookup
     re-syncs the turn before answering (`:401-405`), so coins that fall after that toss
     credit the next player in the queue, or nobody. This is against the Sprint 1 goal
     ("the player who holds the turn"), and **no step of Sprints 1–3 names it**. Step 2's
     measured latency says how often it would happen. Recorded in FEATURE.md → Conflicts;
     nothing fixed. It needs the user's call: re-planning (a step) or `/adhoc`.
  2. **The Room screen's balance does not move on a machine credit** — only the winnings
     counter does, through the poll. The balance changes after the next toss or a reload
     (`views/RoomView.vue:54`, `stores/queue.js:113`). Sprint 1 Step 5's manual
     verification reads "balance … visible on the Room screen"; its plan should say
     which of the two it observes.
  3. **`Install_Schema` seeds no `pc_machine_*` option**, although TECH-STACK →
     ANTI-PATTERNS says the bonus map and entity ids have defaults there. The defaults
     live in code. TECH-STACK is not edited here — an `/adhoc` candidate, or Step 4's
     Docs vs reality.
  4. `Rate_Limiter::client_ip()` trusts `X-Forwarded-For` (review item 5, open), which
     matters for Step 4's rate limit; `Support_Service::notify_support()` is private,
     which matters for Sprint 3 Step 1.
  5. Two §10 bullets are **unassigned**: a failed refund, and the 2-s timeout.
- **Carried from Checks → Docs vs reality:** `admin/src/views/MachineView.vue:156,160,168`
  carries sensor labels that Step 2 may prove wrong, with no step to change them (an
  `/adhoc` candidate).

---

## Plan — Sprint 1, Step 2: Spike — how machine events leave Home Assistant   (status: implemented, awaiting close)

### Branch
`realtime/sprint-1-spike-transport` ← `realtime/sprint-1` (`b596b2d`), **root documentation
repository only**. The step commits no application code, so `backend/`, `frontend/` and
`admin/` stay on `main` untouched (the same as Step 1). Nothing is pushed.

**Spike code is throwaway and never reaches a repository.** The logger, the helper
scripts and every log live in the session scratchpad, outside all four repositories, as
the `stripe` spike's did (`stripe` Sprint 1 Step 2). Only the three documents under
Docs to update are committed.

**Preconditions for `/do-step`:**
1. **The machine token in a private file.** `PC_MACHINE_TOKEN` is **not** defined in the
   local DDEV config (checked at plan time: `wp-config-ddev.php` and `wp-config.php` carry
   no such constant). The user saves the long-lived Home Assistant token to
   `~/.pusher-coin-ha-token`, so it never appears in the chat. Copy the token to the
   clipboard, then run:
   `! (umask 077; pbpaste | tr -d '\r\n' > ~/.pusher-coin-ha-token) && pbcopy </dev/null`
   The token is the one production uses; a dedicated token created for the spike in Home
   Assistant's profile page (and revoked afterwards) is better, if the user can sign in
   there. The scripts read the file at run time, and never print, log or pass it on a
   command line.
2. **One venue day of availability.** The logger runs for 24 hours inside this Claude Code
   session. The session stays open, and the laptop stays on mains power, lid open and
   online. `caffeinate` stops it sleeping while idle, but not when the lid is closed.
   Nobody needs to be at the venue.

**What the spike may do to the machine.** Reads only, with one exception:
`input_button/press` on `input_button.toss_a_coin`, and only after the user's explicit
go-ahead in this session. Each press is announced with its time before it is sent, one
press per go-ahead. **Never** `switch/turn_on` / `turn_off` (`DECISIONS.md` 2026-09-18,
power is manual). **Never** `input_button.relay_on` / `relay_off`; this step does not name
them. No automation, helper or configuration is created in the venue's Home Assistant.

Timebox: one desk session (tasks 1–3) plus one venue day (task 4). Analysis and writing
(tasks 5–7) follow the same day.

### Tasks (ordered)

- [x] **1. Branch and a smoke test** — cut `realtime/sprint-1-spike-transport` from
  `realtime/sprint-1`. Confirm the token file works with one
  `GET https://developer-it.com/api/` → `{"message":"API running."}`, printing only the
  status and body. This is a smoke test so a bad token does not waste the day; it is not
  a re-verification of Session A (`DECISIONS.md` 2026-09-18, fact 1). *No commit.*

- [x] **2. The logger** — in the scratchpad, with no new dependency: Node 26's built-in
  `WebSocket` and `fetch`, plus the system `jq` and `caffeinate`, are all present
  (checked at plan time).
  - **Socket.** `wss://developer-it.com/api/websocket`, `subscribe_events` for
    `state_changed`. Filtered on the client to the logged entities, writing each event's
    full `new_state` / `old_state` (`state`, `last_changed`, `last_updated`,
    `last_reported`, `context.id` / `parent_id` / `user_id`), HA's `time_fired`, and the
    local receipt time in ms. On a dropped connection it writes a `gap` record, then
    reconnects with backoff and writes when it resumed.
  - **REST sampler.** Every 10 s, `GET /api/states/<entity>` for each logged entity,
    written in the same shape with request/response times. Ten seconds keeps the load on
    someone else's Home Assistant at about 0.6 requests a second. It also resolves a
    bonus level that persists and a `last_reported` that ticks more slowly than that.
  - **Logged entities:** the four the step names — `sensor.coin`, `sensor.lc01_12`,
    `sensor.relay_on`, `sensor.sw_b_t_relay` — plus two that are read only:
    `input_button.toss_a_coin` (its state is the time of its last press, which pins toss
    times for latency) and `switch.s60tpf` (on/off marks the edges of the opening day).
  - JSON lines, appended and flushed per record, in files named by UTC hour. It stops
    by itself at the end time it is given. The token is read from the file into memory
    only. *No commit.*

- [x] **3. The desk session** — needs the machine on and idle (a quiet moment in venue
  hours; if it is off when `/do-step` starts, this waits for the first live, quiet
  minute):
  - **The step's own check.** Sample the four sensors twice, 60 s apart, while nothing
    happens. For each, compare `last_reported` with `last_changed` / `last_updated`.
  - **`state_reported`.** One `subscribe_events` attempt for the event Home Assistant
    fires on an unchanged re-write (HA 2024.3+). HA documents that listening to it for
    all entities is not allowed, so a refusal is the expected answer. Record which
    answer it gives.
  - **Can this Home Assistant call out?** An automation can send an HTTP request only
    through a service configured in HA itself (for example `rest_command`), which API
    admin rights may not be able to add. Read `GET /api/services` and `GET /api/config`
    (both read only) and record which outbound-call services and components exist. The
    entry needs this before it can say the webhook option is buildable.
  - **Clock.** The laptop's offset from `time.apple.com` (`sntp`, read only) and the
    socket's ping round trip, at the start and the end. Latency compares Home
    Assistant's timestamps with local receipt times, so the entry states this method.
  *No commit.*

- [x] **4. The venue day** — *Changed by the user during `/do-step`, 2026-09-18 14:3x
  Kyiv: "I can't wait that long", then the chosen option "History + presses". Instead of
  a 24 h wait, the evidence is Home Assistant's own stored history for the last 10 days
  (`/api/history/period` with an explicit `end_time`), plus the live recording already
  running since 11:24:30Z. The user also authorised up to 5 `toss_a_coin` presses today,
  each announced in the chat before it is sent. The recorder is stopped once the live part
  is analysed. The original text follows.* Start the logger as a background task under
  `caffeinate -i`, with an end time 24 h out. That covers one full opening day whatever
  the hours are, and `switch.s60tpf` / the sensors going live mark where it starts and
  ends.
  - **Toss presses:** only as described above (go-ahead, announcement, one press each,
    each press logged with its send and response times).
  - **Gaps:** a gap in the socket or the laptop is filled for *state changes* from
    `/api/history/period` with an explicit `end_time`. Session A (fact 6) found it
    silently returns one day without one. History shows no re-reports, so a gap stays a
    stated gap for the `last_reported` question.
  - **Stop rule:** if the 24 h hold **no** `sensor.coin` movement at all, `/do-step`
    stops and reports instead of writing anything inferred. Extending past the timebox
    is the user's decision.
  *No commit.*

- [x] **5. Analysis** — from the logs only. Each answer names the log lines it rests
  on.
  - **Bonus.** What a bonus looks like on `sensor.lc01_12`, and how long after the first
    `sensor.coin` movement of that payout it appears. If none happened: an observed
    absence.
  - **Coin counter.** What `sensor.coin` counts across one payout, when it resets, and
    whether it jumps or steps.
  - **Relay.** Which of `sensor.relay_on` / `sensor.sw_b_t_relay` changes around a
    payout, and which value means "closed". That includes what the idle `1` on
    `sensor.relay_on` means, which today makes `normalise_truthy()` read "closed"
    (Session A fact 9).
  - **Repeated values.** Does a repeated value leave a trace? Does `last_reported` (or
    `context.id`) of an unchanged sensor move at real machine events, move on a timer,
    or not at all?
  - **Latency.** The machine-side delays come from HA's own timestamps: toss press →
    first coin, coin → bonus, coin → relay. The transport-side delays come from
    receipt times: socket (`time_fired` → receipt) and REST (change → the first sample
    that shows it).
  - **`event_key`.** Candidate formulas per transport, each computable from the fields
    the logs actually hold.
  *No commit.*

- [x] **6. The decision** — one `docs/DECISIONS.md` entry, which:
  - names **exactly one** transport;
  - gives the exact `event_key` formula from logged fields;
  - gives **one** latency number, with how it was measured;
  - gives the corrected sensor model: what the coin counter counts and when it resets,
    the relay entity, and the relay polarity;
  - answers the three questions from observation: does a repeated bonus number leave a
    trace, what does `sensor.coin` count, which relay entity and value mean "closed".
  It says which facts were observed and which come from documents, and contains no
  "probably".
  - **If no bonus occurred,** it says so as a recorded unknown and names what the
    ingest may and may not price from, as the step's task allows.
  - **If it names the WebSocket worker,** it answers the conflict Step 1 recorded (a
    second Home Assistant client outside WordPress, against root invariant 8).
  - **If it names the webhook,** it says whether this HA can call out without the
    machine owner's help (task 3). It also states that the HA → WordPress HTTP leg was
    not observed here and is first measured by Step 5's verification.
  - It records the toss → payout delay: `FEATURE.md` → Conflicts ties the last-coin
    handover to it.
  → docs commit `docs(realtime): spike — how machine events leave Home Assistant`

- [x] **7. The machine as observed, and the spike removed**
  - **`docs/PUSHER-COIN-COMMANDS.txt`:** a dated `NOTE` block under the 2026-09-16 one.
    It covers the observed contact entity and polarity, `sensor.sw_b_t_relay`,
    `sensor.coin`'s semantics and whether values are re-reported. The old lines are not
    rewritten as if they had always said so. It says the entity-default table still
    shows the code's defaults, which later steps change.
  - **`docs/ROADMAP.md`:** Phase 5 §6 becomes "replaced by the spike" with a link to the
    entry. It is tagged `[done]` only if every item it lists is answered by observation
    or already documented; otherwise `[partial]`, naming what remains. The token's
    rotation policy is already in `PUSHER-COIN-COMMANDS.txt:14-16`. The same change
    updates the tracking-matrix row 6 and the "Walk through `PUSHER-COIN-COMMANDS.txt`
    with Dima" open question (root `CLAUDE.md`: the tag and the matrix change together).
  - **Removing the spike:** confirm no logger process is left (`pgrep`), and that
    `git status` is clean in all four repositories apart from this step's documents.
    Remind the user to delete `~/.pusher-coin-ha-token`, and to revoke the token if it
    was a dedicated one.
  → docs commit `docs: the machine as observed — PUSHER-COIN-COMMANDS.txt and ROADMAP Phase 5 §6`

### Files to create/change
- `docs/DECISIONS.md`: one new entry (task 6).
- `docs/PUSHER-COIN-COMMANDS.txt`: a dated note (task 7).
- `docs/ROADMAP.md`: Phase 5 §6, tracking-matrix row 6, one open question (task 7).
- `docs/features/realtime/sprints/SPRINT-1-PLAN.md`: this section's status and ticks.
- Not committed anywhere: the scratchpad logger, helper scripts and logs;
  `~/.pusher-coin-ha-token` (the user's).
- Nothing in `backend/`, `frontend/` or `admin/`.

### Tests to write
None. The step's **Tests** is "—", and spike code is throwaway by the step's own rule.
The evidence is the logs and the log lines the entry cites. No test-critical code is
touched.

### Docs to update
`docs/DECISIONS.md` (the transport entry); `docs/PUSHER-COIN-COMMANDS.txt` where the machine
contradicts it; `docs/ROADMAP.md` Phase 5 §6 (with its matrix row and open question). These
are the three the step names. `FEATURE.md` is not edited: its Data section already covers
whichever transport is chosen (options, cursor transients, the ingest secret), and Step 5
is where the build lands.

### Checks
- **ANTI-PATTERNS:** none violated.
  - The spike calls Home Assistant directly, but from throwaway tooling outside every
    repository. "Only `Machine_Service` calls Home Assistant" governs product code, as
    "only `Stripe_Client` calls Stripe" did while the `stripe` spike used raw `curl`.
  - No power switching; no product code; no WP option written; no secret logged or
    committed.
  - No dependency: Node 26 built-ins, `jq`, `sntp` and `caffeinate` are already
    installed.
- **Docs vs reality:** mismatches, each resolved without adding work:
  1. **The desk rule on repeats does not follow.** The step reads "`last_reported`
     moves while idle" as "a repeated bonus is observable", and "does not move" as
     "invisible to every transport". Neither follows. A timer-driven re-report makes a
     repeat look like a heartbeat, and no idle movement does not exclude a device
     re-writing an identical value at a real event. So the desk check describes idle
     behaviour, and the venue-day log answers the question (task 5).
  2. **Admin rights may not be enough for a webhook.** "Admin rights are confirmed" is
     about the API; an automation can send HTTP only through a service configured in
     HA. So task 3 reads what exists before the entry calls the webhook buildable.
  3. **Latency.** The logs observe the machine-side delays and the socket and REST legs.
     The webhook's HA → WordPress leg cannot be observed without building it, and the
     step does not ask for that. So if the entry names the webhook, it says so (task 6).
  4. **Two more entities.** The step names four sensors; the logger also records
     `input_button.toss_a_coin` and `switch.s60tpf`, read only, for press times and the
     edges of the opening day. No call changes.
  5. **A day with no bonus.** The verification wants all three answers "from
     observation"; the task allows a day with no bonus to end as a recorded unknown.
     Resolution: an observed absence, stated as such, answers the bonus part. Every
     other answer still needs log lines.
  6. **The relay's meaning.** `PUSHER-COIN-COMMANDS.txt:83-100` describes
     `sensor.relay_on` as the state of a relay the system itself closes and opens
     (`input_button.relay_on` / `relay_off`). The code reads "closed" as "payout
     settling" (`RoomQueueController.php:129-135`). Observation settles which (Session A
     facts 8–9); nothing in the tasks changes.
  7. **The token.** `PC_MACHINE_TOKEN` is absent locally, so the spike reads the token
     from a file the user creates (precondition 1).
- **Design:** n/a — no screen.
- **Check command:** `backend/bin/check` and `frontend/bin/check`
  (`docs/TECH-STACK.md` → Check command). Both exited 0 after Step 1's merge, and
  `backend` `main` `5ebe9610` and `frontend` `main` `7210c59` are unchanged. They gate
  the two docs commits.
- **Not locally verifiable:** the spike itself — a 24-hour run against the live venue
  Home Assistant. That run *is* the verification, and nothing else re-creates it.
  Nothing is deployed.

### Questions / ambiguities
none

### Execution notes (for `/close-step`)
- **Commits** (root docs repository, branch `realtime/sprint-1-spike-transport`):
  `e6ebd27` (the `DECISIONS.md` entry) and `72c3b3c` (`PUSHER-COIN-COMMANDS.txt` note,
  ROADMAP Phase 5 §6 + matrix row 6 + the open question), plus this plan file. Nothing in
  `backend/`, `frontend/` or `admin/`; nothing pushed, nothing merged.
- **Preconditions.** The first `/do-step` run stopped: `~/.pusher-coin-ha-token` did not
  exist. The user created it (183 bytes, mode 600) and re-ran. The smoke test answered
  `{"message":"API running."}`, HTTP 200.
- **Deviation 1 — no full venue day (the user's decision, recorded in task 4).** The
  user interrupted ("I can't wait that long") and chose "History + presses" over the
  24-h wait. The evidence is HA's stored history 2026-09-08 → 2026-09-18, plus a live
  recording 11:24:30–13:19:46Z (4,514 records, no socket gap), plus one press.
  - Only **1** of the 5 authorised presses was used: history already held ~200.
  - The step's "a full opening day" is therefore met by ten stored days, not one
    recorded day. The entry states its evidence base.
- **Deviation 2 — the user chose the transport.** The plan had the entry name it. The
  choice had real trade-offs (latency against a new deploy target against depending on
  the machine owner), so it was put to the user with a recommendation. They chose
  "WordPress polls (Recommended)".
- **Deviation 3 — order.** The logger was started before the desk pair, because the
  machine had just been switched on and its sensors were not live yet.
  `state_reported`, outbound services, ping and `sntp` ran during that wait. The idle
  pair ran at 11:25:12Z / 11:26:13Z.
- **Found — for the close report:**
  1. **Every toss is refused today while the machine is on.** `sensor.relay_on`'s
     normal `1` reads as "closed", so `POST /rooms/{id}/play` answers 423. This is a
     `core` defect for `/adhoc`, not fixed here.
  2. **Sprint 2 Step 3's relay lock has no payout signal to follow.**
  3. **Step 4's bonus crediting has no observed bonus behind it,** and a bonus could pay
     twice through `sensor.coin`. Step 4 plans against the entry.
  4. **A second `toss_a_coin` press** (13:17:04Z) came from the same HA user as the
     token, but not from the spike. The user was asked whether it was them and has not
     answered.
  5. **The machine was switched off and on by hand** at 13:04:25Z / 13:14:25Z.
- **Spike removed.** No spike process remains (`pgrep`); the two `caffeinate -t 300`
  processes seen afterwards are not the spike's. The logs and scripts stay in the
  session scratchpad, uncommitted. No token-like string is in any committed document.
  **The user should delete `~/.pusher-coin-ha-token`** (and revoke the token if it was a
  dedicated one). It will be needed again for Step 4/5's verification, so deleting it is
  the user's call.
- **Gate.** `backend/bin/check` exit 0 (stage 2 executed: `stripe-client.php`,
  `stripe-webhook.php`, `wallet-rollback.php`) and `frontend/bin/check` exit 0, before
  each commit.
