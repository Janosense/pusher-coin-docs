# Sprint 1 — step plans (`stripe`)

<!-- Written by /plan-step, one section per step. A section's status moves
     awaiting approval → approved, in progress → implemented, awaiting close →
     closed. Never edit another step's section. -->

## Plan — Sprint 1, Step 1: Delta-audit and the check command   (status: closed)

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

---

## Plan — Sprint 1, Step 2: Spike — Stripe on the test keys   (status: closed)

### Branch
`stripe/sprint-1-spike-keys` ← `stripe/sprint-1`, **root documentation repository
only** — the step commits no application code, so `backend/`, `frontend/` and
`admin/` stay on `stripe/sprint-1` untouched.

Spike artefacts are throwaway and live outside every repository: probe scripts and
logs in the session scratchpad, and the one file that *must* live inside the
WordPress tree (the webhook sink) under `backend/wp-content/mu-plugins/`, which
`backend/.gitignore:35` already ignores — it cannot reach a commit, and task 6
deletes it and proves it gone.

Timebox: one working session. Tasks 1–2 answer the decisive question and can stop
the sprint on their own; 3–5 need `stripe listen` running alongside.

### Tasks (ordered)

- [x] **1. Account facts, from the API rather than the Dashboard** — `GET
  https://api.stripe.com/v1/account` with the test key in an environment variable.
  Record `id`, `country`, `default_currency`, `business_type`. Sending **no**
  `Stripe-Version` header makes Stripe answer with the account's default version in
  the `Stripe-Version` **response header** — that is the account default the step
  asks for, read without Dashboard access. Then fetch Stripe's API changelog for the
  latest dated version, and choose one, recording why. *No commit.*

- [x] **2. The decisive question: does this account accept `uah`?** — create a
  Checkout Session with a raw `curl` (not the CLI, not an SDK — the point is to pin
  the exact HTTP shape `create_checkout_session()` will send through `wp_remote_post`
  in Step 3): `mode=payment`, one line item with `currency=uah` and `unit_amount` in
  **kopiykas**, `client_reference_id`, `success_url` =
  `{pc_spa_base_url}account?topup=success`, `cancel_url` = `…?topup=cancel`
  (`pc_spa_base_url` defaults to `home_url()`, `install-schema.php:242`). Record the
  complete response — `id`, `url`, `expires_at`, `payment_status`, `amount_total`,
  `currency` — and the `Idempotency-Key` behaviour if a repeat is cheap to try.
  **If `uah` is refused: record the error verbatim and stop the spike here.** Per the
  sprint's fixed decisions the currency is not switched in code; see Questions for
  what "stop" means when the account is provisional. *No commit.*

- [x] **3. Webhook sink + `stripe listen`** — `backend/wp-content/mu-plugins/zz-stripe-spike.php`
  (gitignored, throwaway): registers `POST /pc/v1/payments/stripe/webhook` with
  `permission_callback => '__return_true'`, and appends every delivery — all headers,
  the **raw** unparsed body via `$request->get_body()`, and the arrival timestamp — to
  a log under `wp-content/uploads/`. It verifies nothing and settles nothing; it is a
  recorder. Then `stripe listen --forward-to
  https://pusher-coin.ddev.site/wp-json/pc/v1/payments/stripe/webhook`, pay the task-2
  session with `4242 4242 4242 4242`, and record: the `Stripe-Signature` header shape,
  the `checkout.session.completed` payload (`id`, `payment_status`, `amount_total`,
  `currency`, `client_reference_id`), and the measured delay between paying and
  delivery. *No commit.*

- [x] **4. Prove the manual signature scheme works here** — a scratchpad PHP script
  recomputing `HMAC-SHA256` over `{timestamp}.{raw body}` with the `whsec_` that
  `stripe listen` printed, compared against the `v1` value in the logged header. This
  is the exact scheme `verify_signature()` implements in Step 3, so it must be proved
  against a real delivery, not assumed. Record the result and the tolerance field
  (`t`). *No commit.*

- [x] **5. Expiry** — expire a session through Stripe's expire API
  (`POST /v1/checkout/sessions/{id}/expire`) rather than by waiting: `expires_at` has a
  30-minute minimum, which does not fit a timeboxed session. Record whether
  `checkout.session.expired` arrives, how long it took, and its payload. *No commit.*

- [x] **6. Write it down, then remove the spike** — a `docs/DECISIONS.md` entry
  recording, as observations rather than intentions: the account id and **country**,
  the verbatim answer to `uah` (accepted, with the session `url`; or the refusal), the
  pinned API version and why, one real `checkout.session.completed` payload, the
  signature-scheme result, the `expired` observation, and the measured timings. It
  states plainly which account the evidence came from and that the `uah` answer binds
  to that account's country only. `docs/LEARNINGS.md` gets an entry if the CLI, DDEV
  or the certificate needed a workaround. Then delete
  `backend/wp-content/mu-plugins/zz-stripe-spike.php` and the uploads log, and confirm
  with `ls` that both are gone.
  → docs commit `docs: spike — Stripe on the test keys`

### Files to create/change
- `docs/DECISIONS.md` — the spike entry (task 6; the step's actual deliverable)
- `docs/LEARNINGS.md` — only if a workaround was needed (task 6)
- `docs/features/stripe/sprints/SPRINT-1-PLAN.md` — status lines only
- **Never committed, deleted by task 6:** `backend/wp-content/mu-plugins/zz-stripe-spike.php`
  (gitignored at `.gitignore:35`) and its log under `wp-content/uploads/` (also gitignored)
- **Scratchpad only:** the `curl` probes, the signature checker, and the session logs

### Tests to write
None — the sprint step lists none, and the step writes no code that survives it. No
test-critical zone is touched: no money path, no wallet, no permission, no product
code changes anywhere. `backend/bin/check` is still run before the documentation
commit, and must show `Success: All 53 checks passed` rather than the `SKIPPED` box,
since DDEV is up for the spike anyway.

### Docs to update
`docs/DECISIONS.md` (the spike entry); `docs/LEARNINGS.md` if the CLI or DDEV needed a
workaround.

### Checks
- **ANTI-PATTERNS:** none violated — no product code changes at all. The throwaway
  route is deleted by the same task that records the findings, so "every REST route
  declares an explicit `permission_callback`" is satisfied literally (`__return_true`,
  the documented shape for a provider webhook) and temporarily. No money column is
  touched, no transaction reaches `completed`, no dependency is installed (the Stripe
  CLI is already present and is developer tooling, not a project dependency —
  `DECISIONS.md` 2026-09-17). Amounts are computed as integer kopiykas, never a float.
- **Docs vs reality:**
  1. **The Stripe CLI is already installed and logged in** — `stripe` 1.43.8, account
     `acct_1TtSrOElMyJqvLDl`, display name **"Ask Debt Pros Sandbox sandbox"**, a
     `sk_test_` key valid until **2026-10-13**. The sprint assumed keys would have to
     be provided; a working test account is already on this machine. Whether it is the
     *right* account is the open question below.
  2. No `PC_STRIPE_SECRET_KEY` or `PC_STRIPE_WEBHOOK_SECRET` exists anywhere — neither
     config file defines any `PC_*` constant (confirmed by Step 1's audit). The spike
     needs none: session creation is a raw `curl` from the host with the key in an
     environment variable, and the sink only logs. Where the constants finally live is
     Step 3's problem, not this step's.
  3. `backend/wp-config.php` is gitignored (safe for secrets) but carries the
     `#ddev-generated` header, so `ddev start` can silently wipe anything added by
     hand — `LEARNINGS.md` 2026-09-15. A reason not to put the keys there for a spike
     that does not need them.
  4. DDEV is running and serving `https://pusher-coin.ddev.site`, so the
     `stripe listen --forward-to` target in the step text is reachable as written.
- **Design:** n/a — no screen. `stripe` has no UI design (`DECISIONS.md` 2026-09-17).
- **Check command:** `backend/bin/check`, `frontend/bin/check` — present on this
  branch since Step 1, both expected green throughout (no application code changes);
  run before the documentation commit.
- **Not locally verifiable:** n/a in the deploy sense — nothing here is deployed. But
  the whole step is **evidence gathering against a third party**: its findings are only
  as good as the account they came from, and none of it re-runs identically later.

### Risks / notes
- `stripe listen` forwarding to DDEV's HTTPS host may reject its locally-trusted
  certificate. Fallback is the CLI's `--skip-verify`; if it is needed, that is a
  `LEARNINGS.md` line, not a silent workaround.
- The CLI's `whsec_` is **not** the Dashboard's — the local config carries the CLI's,
  production carries the Dashboard's (sprint Risks). The spike records which one it
  proved the scheme against.
- The CLI's test key expires **2026-10-13**. Not a problem for one session; worth
  knowing before Step 3 leans on it.
- The spike sends real (test-mode) API calls to Stripe. Nothing is charged, but the
  account's test-mode logs will show these sessions.

### Questions / ambiguities

1. **Which Stripe account should the spike run against?** This changes what the
   step's decisive answer is worth, so it is worth settling before the work starts.
   The Stripe CLI on this machine is logged into `acct_1TtSrOElMyJqvLDl`, display name
   **"Ask Debt Pros Sandbox sandbox"** — a name that does not look like Pusher Coin,
   and possibly a leftover from unrelated work.
   - **(a) Use it.** The spike runs today. The `DECISIONS.md` entry names the account
     and its country and states that the `uah` answer binds to *that country only*,
     to be re-verified on the client's real account before go-live.
   - **(b) You provide a Pusher Coin test account** (new sandbox, or existing keys).
     The spike runs against that instead; nothing else in the plan changes. Until the
     keys exist, the step is blocked.

   **Recommendation: (a), with the provisional caveat written into the entry.** The
   sprint itself says the client's account, country and owner are still unnamed, so
   there is no "right" account to wait for yet, and a recorded answer from a named
   country beats no answer. Choose (b) only if you already know the country the client
   will register in and can create the sandbox there — that is the one case where (a)'s
   answer could actively mislead.

   **Resolved: approved as recommended — (a).** The spike runs against
   `acct_1TtSrOElMyJqvLDl` ("Ask Debt Pros Sandbox sandbox"), and the `DECISIONS.md`
   entry names that account and its country and states that the `uah` answer binds to
   that country only, to be re-verified on the client's real account before go-live.
   Under (a) a `uah` refusal means **stop and report**, with the currency question
   reopened for the user — not the sprint abandoned on a provisional account's behalf.

   **A consequence worth agreeing now.** The step text says a `uah` refusal stops the
   sprint. Under (a) that is too strong: a refusal would be a fact about *this
   sandbox's* country, not about the client's future account. So under (a), a refusal
   means **stop and report**, with the currency decision reopened as a question for you
   — not the sprint abandoned on a provisional account's behalf. Under (b), with an
   account in the client's real country, the step text stands as written.

### Execution notes (for `/close-step`)
- **Account used:** `acct_1TtSrOElMyJqvLDl` — country **US**, `charges_enabled: false`,
  "Ask Debt Pros Sandbox sandbox", `sk_test_` expiring 2026-10-13. Provisional, per the
  resolved question; every finding is flagged as US-sandbox evidence.
- **The decisive answer: `uah` is accepted** (HTTP 200, `currency: uah`,
  `amount_total: 12000`, page rendered "UAH 120.00"). The sprint does **not** stop.
- **Findings the sprint did not anticipate**, all in the `DECISIONS.md` entry:
  Adaptive Pricing is on by default and must be disabled in Step 3; the signature
  header carries three parts (`t`, `v1`, `v0`); events do not arrive in creation order;
  an idempotent replay returns the cached original body, not live state.
- **No `LEARNINGS.md` entry:** the plan called for one only if the CLI, DDEV or the
  certificate needed a workaround. None did — `stripe listen` accepted DDEV's
  certificate without `--skip-verify`.
- **Spike removed and verified:** `wp-content/mu-plugins/zz-stripe-spike.php` and its
  uploads log deleted; `POST /wp-json/pc/v1/payments/stripe/webhook` now answers **404**;
  `backend/`, `frontend/` and `admin/` all report zero changes.
- **Payment method:** Stripe's published test number `4242…` in a `livemode: false`
  sandbox — a reserved number that belongs to nobody and moves no funds.
