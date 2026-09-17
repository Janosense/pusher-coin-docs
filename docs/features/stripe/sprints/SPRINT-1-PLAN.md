# Sprint 1 — step plans (`stripe`)

<!-- Written by /plan-step, one section per step. A section's status moves
     awaiting approval → approved, in progress → implemented, awaiting close →
     closed. Never edit another step's section. -->

## Plan — Sprint 1, Step 1: Delta-audit and the check command   (status: implemented, awaiting close)

### Branch
`stripe/sprint-1-check-command` ← `stripe/sprint-1` ← `main`, merged back `--no-ff`.

`stripe/sprint-1` does not exist yet in any repository — this step cuts it from
`main` in all four (root docs, `backend/`, `frontend/`, `admin/`) before cutting the
task branch. Nothing is pushed: a push of `backend` `main` is a production release,
and `main` moves only at the sprint boundary.

The four `realtime/sprint-1` commits are **read from** that branch and **copied**,
never cherry-picked — `DECISIONS.md` 2026-09-17 "The check command reaches
`stripe/sprint-1` as copied files, not as a cherry-pick onto `main`".

### Tasks (ordered)

- [x] **1. `backend/bin/check`** — write `git show realtime/sprint-1:bin/check` into
  `backend/bin/check` byte-for-byte, `chmod 755`. Verify with
  `git diff realtime/sprint-1 -- bin/check` (must print nothing).
  → backend commit `chore: add bin/check — theme syntax check plus DDEV eval checks`
  *touches shared code — this script gates every later commit of this sprint and is
  the same file `realtime` already gates its own commits with.*

- [x] **2. `frontend/bin/check` + the lint split** — write
  `git show realtime/sprint-1:bin/check` into `frontend/bin/check` byte-for-byte,
  `chmod 755`; in `frontend/package.json` drop `--fix` from `lint` and add
  `lint:fix` with it (the two lines exactly as on `realtime/sprint-1`). Verify both
  with `git diff realtime/sprint-1 -- bin/check package.json` (must print nothing).
  → frontend commit `chore: make lint report-only, add lint:fix and bin/check`
  *touches shared code — `frontend/.github/workflows/ci.yml` runs `npm run lint`, so
  from here CI reports fixable errors instead of silently repairing them. Consuming
  feature: `realtime`, which made the identical change on its own branch.*

- [x] **3. `admin/package.json` lint split** — the same two script lines.
  Verify with `git diff realtime/sprint-1 -- package.json` (must print nothing).
  → admin commit `chore: make lint report-only, add lint:fix`
  *touches shared code — `admin/` has no CI, so this only changes what a local
  `npm run lint` does. Consuming feature: `realtime`.*

- [x] **4. The check-command docs** — replace `docs/TECH-STACK.md` → `## Check
  command` (the single hunk, current lines 33–65) with the text from
  `git show realtime/sprint-1:docs/TECH-STACK.md`; replace the `## Commands` code
  block of the root `CLAUDE.md` with the realtime version, **keeping** the `stripe`
  row of the Features table, which exists on `main` and not on that branch; add the
  two `bin/` entries to `docs/PROJECT-TREE.md` (`backend/bin/check` after
  `backend/.gitignore`, `frontend/bin/check` after `frontend/README.md`) — for
  `PROJECT-TREE.md` the whole file may be taken from the branch, its only difference
  from `main` being those two entries.
  → docs commit `docs: the check command — TECH-STACK, CLAUDE.md commands, project tree`

- [x] **5. The delta-audit** — read, with no code change: `docs/features/core/FEATURE.md`
  (Interfaces, Invariants); `docs/features/realtime/FEATURE.md` **from the realtime
  branch** (`git show realtime/sprint-1:docs/features/realtime/FEATURE.md` — `main`'s
  copy predates their own Step 1 audit); and the shared code listed under Files
  below. Confirm or correct every touchpoint in
  `docs/features/stripe/FEATURE.md` → Fit into the host, naming for each the file and
  the line, and the step of this sprint that will change it.
  → docs commit `docs(stripe): delta-audit of shared code in FEATURE.md`

**Not a task — already satisfied.** The sprint step's third task ("add the `stripe`
row to the root `CLAUDE.md` Features table and to `docs/ARCHITECTURE.md` → Feature
map if discovery's row is missing after the merge with `main`") is a no-op: both rows
are present on `main` today (`CLAUDE.md:95`, `docs/ARCHITECTURE.md:54`). Task 4 only
has to avoid deleting the `CLAUDE.md` one while copying around it.

### Files to create/change

| File | Change |
|---|---|
| `backend/bin/check` | **new**, mode 755 — byte-identical copy (89 lines) |
| `frontend/bin/check` | **new**, mode 755 — byte-identical copy (39 lines) |
| `frontend/package.json` | `lint` loses `--fix`; new `lint:fix` |
| `admin/package.json` | same two lines |
| `docs/TECH-STACK.md` | `## Check command` section replaced |
| `CLAUDE.md` | `## Commands` block replaced; Features table untouched |
| `docs/PROJECT-TREE.md` | two `bin/check` entries added |
| `docs/features/stripe/FEATURE.md` | Fit into the host — audit result |

Read-only in task 5 (no change): `app/utils/wallet-service.php`,
`app/rest-api/WalletController.php`, `app/rest-api/PaymentController.php`,
`app/utils/liqpay-client.php`, `app/utils/install-schema.php`;
`frontend/src/components/ReplenishmentBalance.vue`, `src/stores/wallet.js`,
`src/views/AccountView.vue`, `src/services/liqpayCheckout.js`;
`admin/src/views/SettingsView.vue`, `src/router/index.js`,
`src/components/AdminLayout.vue`. All twelve exist on `main` — verified while planning.

### Tests to write
**None** — the step's own text says so, and nothing here touches money code. The step
instead makes the existing money check runnable by the gate: `backend/bin/check`
stage 2 must execute `wp-content/themes/pc/tests/wallet-rollback.php` and show its
53 checks, not skip them. That is what the manual verification looks for.

### Docs to update
`docs/TECH-STACK.md` → Check command; root `CLAUDE.md` → Commands;
`docs/PROJECT-TREE.md`; `docs/features/stripe/FEATURE.md` → Fit into the host.

### Checks

- **ANTI-PATTERNS:** none violated. The step writes no application code — two shell
  scripts, four `package.json` script lines and four documentation edits. No money
  path, no REST route, no meta key, no dependency (the Stripe CLI is Step 2's
  tooling, not this step's).

- **Docs vs reality:** mostly match, with four recorded items —
  1. `docs/TECH-STACK.md` → Check command on `main` still says "**There is none.**"
     That is exactly what task 4 replaces; on `main` it stays true until a sprint
     boundary lands the files there.
  2. The decision record "The check command calls the WP-CLI eval scripts, and skips
     them without DDEV" (`DECISIONS.md` 2026-09-15) lives **only** on
     `realtime/sprint-1`. It is **not** copied: the step and the 2026-09-17 copy
     decision both enumerate three doc targets, and `DECISIONS.md` is in neither
     list. No dangling reference results — the copied TECH-STACK text cites
     "`DECISIONS.md` 2026-09-15" for the WP-CLI + DDEV guard, and the entry
     "Interim money checks are WP-CLI eval scripts, not PHPUnit", which **is** on
     `main`, states that guard in its last line. Both entries reach `main` together
     at whichever sprint boundary comes first.
  3. `docs/PROJECT-TREE.md` lists `frontend/CLAUDE.md`; that file does not exist and
     is not tracked in the `frontend` repository. Outside this step's scope (the step
     adds `bin/` entries only) → `/adhoc`, or fold into a later PROJECT-TREE edit.
  4. `realtime/sprint-1` is one commit behind `backend` `main` on
     `app/utils/machine-service.php` (the 2026-09-16 power-switch ad-hoc). It does
     not touch `bin/check`, so the byte-identical copy is unaffected.

- **Design:** n/a — no screen. `docs/features/stripe/design/` is empty by decision
  (`DECISIONS.md` 2026-09-17 "`stripe` has no UI design").

- **Check command:** `backend/bin/check` and `frontend/bin/check` — **absent on
  `main`**, created by tasks 1 and 2 of this plan from `realtime/sprint-1`. From
  task 2 onward every commit of this sprint goes through them; tasks 1–2 are
  themselves gated by running the script being added, which is how the 2026-09-16
  ad-hoc handled the same bootstrap problem (`docs/LEARNINGS.md`).

- **Not locally verifiable:** n/a. Both scripts and both lint scripts run on this
  machine, and the report-only lint was exercised during planning: `eslint` with the
  copied (no `--fix`) arguments exits 0 in `frontend/` **and** `admin/` today, so
  task 2 does not turn `frontend` CI red. Host PHP is 8.5.4, matching the copied
  text's "8.5 on the development machine". `backend/bin/check` stage 2 needs DDEV
  running, which is a local run, not a deploy.

### Questions / ambiguities
none

### Execution notes (for `/close-step`)
- **Commits.** backend `40fbfc8c`, frontend `9167691`, admin `c935d4d`, docs `01c82fe`
  `636af4b`. Branch `stripe/sprint-1-check-command` in all four repositories; nothing
  pushed, nothing merged.
- **Byte-identity proved by blob hash, not by eye.** `backend/bin/check` and
  `frontend/bin/check` are blob `1df13c17` / the realtime blob at mode `100755`;
  `git diff realtime/sprint-1 -- bin/check package.json` is empty in `backend/`,
  `frontend/` and `admin/`.
- **Gate output.** `backend/bin/check` exits 0 with DDEV up: 45 files linted and
  `wallet-rollback.php` **executed**, "Success: All 53 checks passed" — not the
  SKIPPED box. `frontend/bin/check` exits 0 (lint + build). `admin/`
  `npm run lint && npm run build` exits 0. Report-only lint left `git status` showing
  only the intended files, in both SPAs.
- **Open for the close report:** `docs/features/stripe/FEATURE.md` is 96 lines, over
  its own ≤80-line guideline — the audit detail is what the step asked for, and
  trimming other sections was outside this step's scope.
- **Carried from Checks → Docs vs reality:** `docs/PROJECT-TREE.md` lists a
  `frontend/CLAUDE.md` that does not exist — an `/adhoc` candidate, untouched here.
