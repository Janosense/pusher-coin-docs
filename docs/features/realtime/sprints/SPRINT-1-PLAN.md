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
