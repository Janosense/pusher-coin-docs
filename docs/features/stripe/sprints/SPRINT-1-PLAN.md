# Sprint 1 — step plans (`stripe`)

<!-- Written by /plan-step, one section per step. A section's status moves
     awaiting approval → approved, in progress → implemented, awaiting verification →
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

---

## Plan — Sprint 1, Step 3: The Stripe client and the Checkout Session   (status: closed)

### Branch
`stripe/sprint-1-checkout-session` ← `stripe/sprint-1`, **`backend/` and the root
documentation repository**. `frontend/` and `admin/` are not touched — the SPA hand-off
is Step 5.

### Tasks (ordered)
Docs are committed with the code they describe (core rule 5), so there is no trailing
documentation task.

- [x] **1. The Stripe client** — `app/stripe/bootstrap.php` (requires the client, the
  single entry point) and `app/stripe/stripe-client.php`: `final class Stripe_Client` in
  namespace `PC`, mirroring `LiqPay_Client`'s shape.
  - `API_BASE = 'https://api.stripe.com/v1'`, `API_VERSION = '2026-06-24.dahlia'` — the
    version the spike pinned, as a class constant, not an option: an API version is a
    code contract, not an operator-tunable business value (core rule 3).
  - `secret_key()` / `webhook_secret()` private, reading `PC_STRIPE_SECRET_KEY` and
    `PC_STRIPE_WEBHOOK_SECRET`; `is_configured()` true only when **both** are present
    (step text, and `FEATURE.md` invariant 5).
  - `mode()` — `test` for an `sk_test_` prefix, `live` for `sk_live_`, `''` otherwise.
  - `to_kopiykas( string $amount ): string` — `bcmul( $amount, '100', 0 )`. A named,
    testable unit so the conversion has somewhere to be tested; never a float.
  - `create_checkout_session( array $params, string $idempotency_key )` —
    `wp_remote_post` with a Bearer key, `Stripe-Version`, `Idempotency-Key`, and
    **`adaptive_pricing[enabled] = false` injected here**, the way `LiqPay_Client`
    injects `version`/`public_key`, so a caller cannot forget it (the spike found
    Stripe enables it unasked). Non-2xx or transport failure → typed
    `WP_Error( 'stripe_call_failed', … )` carrying Stripe's own `error.message` **and
    never the key**; the key appears only in the request header.
  - `verify_signature( string $raw_body, string $header, string $secret, int $tolerance = 300 ): bool`
    — parse the header into `t` and **every** `v1` (the spike saw `t,v1,v0`: match by
    key name, never by position), recompute `HMAC-SHA256` over `{t}.{raw_body}`,
    compare with `hash_equals`, and reject a timestamp outside the tolerance.
  - One `require_once TEMPLATE_DIR . '/app/stripe/bootstrap.php';` line in
    `functions.php`. *Touches shared code — `functions.php` is `core`'s.*
  - `tests/stripe-client.php` (WP-CLI eval, same guard and `$check` harness as
    `tests/wallet-rollback.php`): `to_kopiykas` for `0.01` → `1`, `10.00` → `1000`,
    `499.99` → `49999`, `123456.78` → `12345678`; `mode()` for both prefixes and for an
    unset key; `is_configured()` with each constant missing in turn; and
    **`verify_signature`** — a self-signed valid fixture accepted, a one-byte-altered
    body rejected, a wrong secret rejected, a stale timestamp rejected, a header
    carrying `v0` before `v1` still accepted. The signature cases are here rather than
    waiting for Step 4 because the profile makes signature verification a
    test-critical zone, and code shipped there without tests is an unfinished task.
  - `docs/PROJECT-TREE.md`: `app/stripe/` and `tests/stripe-client.php`.
  → backend commit `feat(stripe): Stripe client — session creation and signature verification`

- [x] **2. `WalletController::topup` on Stripe** — validation, bounds and the `pending`
  row stay byte-for-byte as they are (`WalletController.php:83-119`); only what happens
  after `record_transaction` changes.
  - New `Wallet_Service::set_external_ref( int $txn_id, string $ref ): bool` — replaces
    the raw `$wpdb->update` at `WalletController.php:127`, checked like every other
    write (`false !== $result`, the `update_transaction_status` pattern).
  - The controller: unconfigured → `stripe_not_configured` 500 **before** the row is
    written, so no row is created (the step's negative verification). Otherwise build
    one line item — `quantity=1`, `unit_amount = to_kopiykas( amount )`,
    `currency=uah`, `product_data[name]` naming the coin count and unit price;
    `client_reference_id = txn_id`; `success_url` / `cancel_url` from
    `pc_spa_base_url` (`install-schema.php:242`). **`quantity=1` with the total as
    `unit_amount`** is what the spike validated, and it makes Step 4's `amount_total`
    check an exact comparison against `amount_money` with no arithmetic on Stripe's
    side. `Idempotency-Key` = `pc-topup-{txn_id}`.
  - Store the returned `cs_…` id through `set_external_ref`; respond
    `{ transaction_id, external_ref, amount, checkout_url }`.
  - A failed session creation → `update_transaction_status( …, STATUS_FAILED, <error code> )`
    and a 502, mapping `stripe_call_failed` → 502 with the lookup-array pattern of
    `RoomQueueController.php:228-230`.
  - Append the `set_external_ref` cases to `tests/stripe-client.php`: a missing row
    returns `false`; a real row round-trips through
    `find_transaction_by_external_ref`.
  - `docs/CONTRACTS.md`: the `POST /wallet/topup` response (`order_id` and the `liqpay`
    envelope out, `external_ref` in), `stripe_not_configured` 500 and
    `stripe_call_failed` 502 in the registry, and `liqpay_not_configured` (row 1528)
    narrowed from "wallet/topup, payments/liqpay/callback" to the callback alone.
  - *Touches shared code — `Wallet_Service` and `WalletController`, both `core`'s;
    consuming feature: `realtime`, which credits lots through `Wallet_Service` but
    touches neither `WalletController` nor the top-up path (their Step 1 audit).*
  → backend commit `feat(stripe): /wallet/topup creates a Checkout Session`

- [x] **3. Retire the LiqPay option** — `Install_Schema`: `DB_VERSION` `1.8.0` → `1.9.0`,
  drop `add_option( 'pc_liqpay_public_key', '' )` (`:249`), and add a
  `remove_retired_options()` called from `maybe_install()` that runs
  `delete_option( 'pc_liqpay_public_key' )`. **There is no upgrade path today** —
  `maybe_install()` only installs schema and adds defaults — so this task creates the
  smallest thing that removes an option once per version bump, not a general migration
  framework (core rule 2).
  - `docs/DATA-MODEL.md`: the two wp-config constants, the removed option, the new
    `DB_VERSION`, and the `external_ref` convention (`cs_…` for Stripe-era rows;
    LiqPay-era rows keep `pc-topup-N`).
  - *Touches shared code — `Install_Schema` is `core`'s.*
  → backend commit `feat(stripe): retire pc_liqpay_public_key, bump DB_VERSION to 1.9.0`

### Files to create/change
| File | Change |
|---|---|
| `app/stripe/bootstrap.php` | **new** — the single entry point |
| `app/stripe/stripe-client.php` | **new** — `Stripe_Client` |
| `functions.php` | one `require_once` line (shared) |
| `tests/stripe-client.php` | **new** — grows across tasks 1 and 2 |
| `app/utils/wallet-service.php` | `set_external_ref()` added (shared) |
| `app/rest-api/WalletController.php` | `topup()` rewritten in place (shared) |
| `app/utils/install-schema.php` | `DB_VERSION`, option removal (shared) |
| `docs/CONTRACTS.md`, `docs/DATA-MODEL.md`, `docs/PROJECT-TREE.md` | as above |

Unchanged on purpose: `PaymentController.php`, `liqpay-client.php` and their loader
lines — removing them is Step 5. `frontend/`, `admin/` — untouched.

### Tests to write
`backend/wp-content/themes/pc/tests/stripe-client.php`, picked up automatically by
`backend/bin/check` stage 2 (a new file in `tests/` needs no change to the script).
Money zone and the test-critical signature zone, so it ships with its code, not after:

- `to_kopiykas`: `0.01` → `1`, `10.00` → `1000`, `499.99` → `49999`, `123456.78` → `12345678`
- `mode()`: `sk_test_` → `test`, `sk_live_` → `live`, unset → `''`
- `is_configured()`: false with either constant missing, true with both
- `verify_signature()`: valid self-signed fixture accepted; body altered by one byte
  rejected; wrong secret rejected; timestamp beyond tolerance rejected; `v0` present
  alongside `v1` still accepted
- `set_external_ref()`: missing row → `false`; real row round-trips

Not tested here, and why: `settle_topup` is untouched by this step and already covered
by `wallet-rollback.php`; the webhook's routing and idempotency are Step 4's
`tests/stripe-webhook.php`; a live Stripe call is not faked — it is exercised by the
manual verification below.

### Docs to update
`docs/CONTRACTS.md` (task 2), `docs/DATA-MODEL.md` (task 3), `docs/PROJECT-TREE.md`
(task 1).

### Checks
- **ANTI-PATTERNS:** none violated, and three are directly engaged —
  *money never in a float*: `bcmul` on the decimal string throughout, kopiykas as an
  integer string; *no money column written outside `Wallet_Service`*: this step
  **removes** the last such write, the `$wpdb->update` at `WalletController.php:127`;
  *every route declares an explicit `permission_callback`*: the topup route keeps
  `Permissions::require_play_ready` untouched. No `ENUM`, no new option, no meta-key
  literal, no new dependency (`wp_remote_post`, no SDK — `DECISIONS.md` 2026-09-17),
  and no transaction reaches `completed` here.
- **Docs vs reality:**
  1. **The player SPA's top-up breaks between this step and Step 5, by design.**
     `services/walletService.js:22-31` still maps `order_id` and the `liqpay` envelope,
     and `ReplenishmentBalance.vue:76` throws `Missing LiqPay envelope fields` when
     they are absent. The sprint sequences the hand-off into Step 5, and nothing
     deploys before the sprint boundary, so this is expected — but it means **the
     manual verification of this step must call the endpoint directly** (curl with a
     bearer token for a play-ready user), not press the SPA's button. The step text's
     "from the SPA's dev server" is read as "against the running local stack".
  2. `Install_Schema` has **no upgrade path** — the step says `delete_option` "in the
     upgrade path", and there is none to put it in. Task 3 creates the minimal one.
  3. `liqpay_not_configured` cannot be fully retired here: `PaymentController` still
     returns it until Step 5. CONTRACTS row 1528 is narrowed, not deleted.
  4. `external_ref` is `VARCHAR(128)`; the session id observed in the spike is 66
     characters. Verified — no column change needed.
  5. `Wallet_Service::record_transaction` does not check its own insert (it returns
     `insert_id` unconditionally, unlike every other write in that class). Pre-existing,
     outside this step — a `BACKEND-REVIEW`/`/adhoc` item, not touched here.
- **Design:** n/a — no screen. `stripe` has no UI design (`DECISIONS.md` 2026-09-17).
- **Check command:** `backend/bin/check` (and `frontend/bin/check`, unaffected) — run
  before every commit, with DDEV up so the money checks execute rather than showing the
  `SKIPPED` box.
- **Not locally verifiable:** n/a for the deploy sense — nothing deploys. Two caveats:
  the manual verification needs `PC_STRIPE_SECRET_KEY` and `PC_STRIPE_WEBHOOK_SECRET`
  added by hand to the local `wp-config.php` (gitignored, but `#ddev-generated`, so
  `ddev start` can wipe them — `LEARNINGS.md` 2026-09-15); and the sandbox `sk_test_`
  key **expires 2026-10-13**, after which this verification needs a fresh key.

### Questions / ambiguities
none

### Execution notes (for `/close-step`)
- **Commits.** backend `433b19a8` `d6aa851f` `4603f09e`, docs `5ab61a1` `6afde91`
  `e683895`. `frontend/` and `admin/` never left `stripe/sprint-1`.
- **Verified against real Stripe from the local stack**, not only by unit checks:
  `POST /wallet/topup` answered 200 with `external_ref` `cs_test_a1rqvK7C…`,
  `amount` `120.00` and a `checkout.stripe.com` URL; the row was `pending` with that
  ref and `settled_at` NULL; reading the session back showed
  `adaptive_pricing {enabled: false}`, `currency uah`, `amount_total 12000`,
  `client_reference_id 59`. The negative case: with `PC_STRIPE_WEBHOOK_SECRET`
  commented out, `stripe_not_configured` 500 and **the transaction count did not move**.
- **The upgrade path was replayed, not assumed:** seeded `pc_db_version` `1.8.0` with
  `pc_liqpay_public_key` present → after `maybe_install()` the option is absent and the
  version is `1.9.0`; a second run changes nothing.
- **Checks:** `tests/stripe-client.php` 39, `tests/wallet-rollback.php` 53, both under
  `backend/bin/check`, which needed no change to pick the new file up.
- **Two deviations from the plan**, both narrowing rather than widening:
  1. `stripe_call_failed` → 502 is set directly on the `WP_Error` instead of through a
     lookup array. `create_checkout_session()` returns exactly one error code, so the
     table would have had a single row — machinery over nothing (core rule 2). Same
     behaviour.
  2. A `set_external_ref` failure answers `wallet_write_failed` 500 and parks the row
     `failed`. The plan said the write is "checked like every other write" but did not
     say what the check does on failure; a row whose session id was never stored can
     never be found by the webhook, so failing loudly beats returning a URL that can
     never settle. Reuses an existing registered code rather than inventing one.
- **Considered and left out as out of scope:** `Audit_Log` entries on the two failure
  paths. `Audit_Log` is listed in `FEATURE.md` as shared code this feature uses, but
  Step 4 is the step that names it; the failures are already recorded in the row's
  `notes` and returned to the caller.
- **Local-only state left behind on purpose:** wp-config constants (gitignored,
  `#ddev-generated`, so `ddev start` will drop them), the `stripe-step3-check` player
  and one `pending` top-up row. None of it is tracked or deployed.
- **`docs/DATA-MODEL.md` invariant 6 deliberately still says "the LiqPay callback"** —
  it still is, until Step 4 ships the webhook and Step 5 removes the route. That
  rewording is in Step 4's own docs list.

---

## Plan — Sprint 1, Step 4: The webhook and settlement   (status: closed)

### Branch
`stripe/sprint-1-webhook` ← `stripe/sprint-1`, **`backend/` and the root documentation
repository**. `frontend/` and `admin/` are not touched.

**No shared code is modified.** Unusually for this sprint, every code file is the
feature's own: `Wallet_Service` and `Audit_Log` are *called*, not changed, and
`functions.php` already requires the feature's bootstrap. The only `core`-owned files
touched are documentation (root `CLAUDE.md`, `docs/ARCHITECTURE.md`).

### Tasks (ordered)

- [x] **1. The webhook** — `app/stripe/StripeWebhookController.php`, registered from
  `app/stripe/bootstrap.php` on `rest_api_init` (the feature's single entry point, so
  `core`'s `app/rest-api.php` stays untouched).

  **The credential.** `permission_callback => '__return_true'`, with the signature as
  the credential — the shape ANTI-PATTERNS allows for a public route. The raw body
  comes from `$request->get_body()` and is never re-encoded: re-serialising the JSON
  changes bytes and the signature would not match.

  **The only non-2xx the handler returns** (the rest is settled by the Questions
  section below):
  - no `Stripe-Signature` header, or an empty body → **400 `missing_required_fields`**
    (already registered; the LiqPay callback uses it for the same thing)
  - signature fails `Stripe_Client::verify_signature` → **401 `stripe_signature_invalid`**
    (new code). A missing `PC_STRIPE_WEBHOOK_SECRET` lands here too, because the
    verifier returns false on an empty secret — and 401 is the *right* outcome: Stripe
    retries for three days, so events are not lost while the operator restores the
    secret. The audit entry distinguishes the two.

  **After a valid signature, every branch answers 200** with a `note`, exactly as the
  LiqPay callback does, because a 4xx/5xx would only make Stripe retry something that
  cannot improve (`DECISIONS.md` 2026-09-17):
  | Condition | Answer |
  |---|---|
  | Body is not valid JSON, or carries no `type` | `note: ignored` |
  | `checkout.session.completed` / `…async_payment_succeeded`, `payment_status` ≠ `paid` | `note: ignored` |
  | …`paid`, no row for that `external_ref` | `note: unknown_session` |
  | …row is not `pending` | `note: already_settled` |
  | …`amount_total` ≠ the row, or `currency` ≠ `uah` | `note: amount_mismatch`, no settlement |
  | …otherwise | `Wallet_Service::settle_topup`, no note |
  | `checkout.session.async_payment_failed` / `…expired`, row `pending` | row → `failed`, note = the event type |
  | …same events, row not `pending` | `note: already_settled`, nothing changes |
  | any other event type | `note: ignored` |

  The amount comparison is `Stripe_Client::to_kopiykas( $row['amount_money'] )` against
  the event's `amount_total` — both integer strings, compared with `bccomp`, never a
  float. `settle_topup` is called with the **row's** `amount_coins` and `unit_price`,
  never the event's: Stripe confirms the money, the ledger defines the coins.

  Every branch writes `Audit_Log::record` with the event id, event type and session id
  in `metadata` (event-type names stay under the column's 64 characters).

  `tests/stripe-webhook.php` (WP-CLI eval, same guard and `$check` harness as the two
  existing scripts) signs its own fixtures with a known secret and drives the
  controller through a `WP_REST_Request` with `set_body()` / `set_header()` — no HTTP,
  no Stripe. Cases listed under **Tests** below.

  `docs/CONTRACTS.md`: a new `POST /pc/v1/payments/stripe/webhook` section beside the
  LiqPay callback, its `note` vocabulary, and `stripe_signature_invalid` 401 plus
  `missing_required_fields` extended to the new route in the registry.
  `docs/PROJECT-TREE.md`: the controller and the test script.
  → backend commit `feat(stripe): the settlement webhook`
  → docs commit `docs: CONTRACTS — the Stripe settlement webhook`

- [x] **2. The system-level record** — the docs that only become true once settlement
  exists:
  - root `CLAUDE.md` **invariant 5** — the Stripe webhook is the only place a
    transaction flips `pending → completed`, idempotent on the session id and the
    row's status (not `(order_id, status)`, which was LiqPay's scheme). The LiqPay
    route still exists until Step 5, so the invariant says so in a parenthetical
    rather than pretending it is already gone.
  - root `CLAUDE.md` **test-critical zones** line — "the LiqPay callback — signature
    verification and `(order_id, status)` idempotency" becomes the Stripe webhook and
    its actual idempotency rule.
  - `docs/ARCHITECTURE.md` → **Top-up data flow** rewritten to the Stripe path, and a
    **Stripe row added** to Integrations. The LiqPay row is *not* removed here — it is
    still wired until Step 5, which owns its removal.
  - `docs/TECH-STACK.md` → the ANTI-PATTERNS line "Do not flip a transaction to
    `completed` anywhere but the LiqPay callback" is Step 5's to rewrite; this step
    leaves it, because until Step 5 that line is still literally true of the code.
  → docs commit `docs: settlement moves to the Stripe webhook — invariants and data flow`

### Files to create/change
| File | Change |
|---|---|
| `app/stripe/StripeWebhookController.php` | **new** |
| `app/stripe/bootstrap.php` | requires and registers the controller |
| `tests/stripe-webhook.php` | **new** |
| `docs/CONTRACTS.md` | the webhook section + two registry rows |
| `docs/PROJECT-TREE.md` | both new files |
| `CLAUDE.md` | invariant 5, test-critical zones line |
| `docs/ARCHITECTURE.md` | Top-up data flow, Integrations (Stripe row added) |

Unchanged on purpose: `Wallet_Service`, `Audit_Log`, `WalletController`,
`functions.php`, `app/rest-api.php`, everything LiqPay, `frontend/`, `admin/`.

### Tests to write
`tests/stripe-webhook.php`, picked up automatically by `backend/bin/check`. This is the
sprint's test-critical step — the webhook is named in the project profile — so the
script is the gate, and it creates one throwaway user and removes every row it makes.

1. Valid signature, `completed`, `paid` → row `completed`, **one** coin lot at the
   row's `unit_price`, `balance_coins` up by `amount_coins`
2. The identical event again → `already_settled`; balance, lots and row unchanged
3. `async_payment_succeeded` with `paid` → settles the same way (the second settling type)
4. `completed` with `payment_status: unpaid` → `ignored`, nothing written
5. Invalid signature → 401, nothing written
6. Timestamp beyond the tolerance → 401, nothing written
7. Missing `Stripe-Signature` header → 400, nothing written
8. `amount_total` one kopiyka off → `amount_mismatch`, row still `pending`, no lot
9. `currency: "usd"` with the right amount → `amount_mismatch`, no settlement
10. Unknown `external_ref` → `unknown_session`, nothing written
11. `expired` on a `pending` row → row `failed`, no lot, balance unchanged
12. `expired` on a `completed` row → **unchanged**, still `completed`, balance unchanged
13. `async_payment_failed` on a `pending` row → row `failed`
14. `charge.dispute.created` → 200 `ignored`, nothing changes anywhere
15. Every branch above wrote an `Audit_Log` row carrying the event id

Not tested here: `settle_topup`'s own rollback behaviour, already covered by
`wallet-rollback.php`'s 53 checks; a live Stripe delivery, which is the manual
verification.

### Docs to update
`docs/CONTRACTS.md`, `docs/PROJECT-TREE.md` (task 1); root `CLAUDE.md`,
`docs/ARCHITECTURE.md` (task 2).

### Checks
- **ANTI-PATTERNS:** none violated. *Public route carries its own credential* — the
  provider signature, the shape the list names. *No money column written outside
  `Wallet_Service`* — the controller calls `settle_topup` and
  `update_transaction_status`, and writes nothing itself. *Money never a float* — the
  amount check is `bccomp` on integer strings of kopiykas. *No `ENUM`, no meta-key
  literal, no new option, no dependency.* *Nothing is hard-deleted.* The one line that
  will conflict — ANTI-PATTERNS' "do not flip a transaction to `completed` anywhere but
  the LiqPay callback" — stays as written this step, because until Step 5 removes that
  route it is still true of the shipped code; Step 5 owns the rewrite.
- **Docs vs reality:**
  1. **`wp_pc_transactions` has no `currency` column.** The step says to compare the
     event's `currency` "to the row"; there is nothing on the row to compare to. The
     check is therefore against the product's only currency, `uah` — the sprint's fixed
     decision and root `CLAUDE.md` invariant 7. Recorded rather than silently dropped.
  2. **There is nothing to "move into current" in CONTRACTS.** The step says the webhook
     moves from planned to current, but `## Endpoints — planned` never listed a Stripe
     webhook (only a machine one). Task 1 adds a new current section instead.
  3. **The manual verification cannot use the SPA.** The step says "pay a session
     created through the SPA"; the SPA's top-up button stays broken until Step 5
     (Step 3's close). The session is created by calling `/wallet/topup` directly, as
     in Step 3's guide, and then paid on Stripe's hosted page — which exercises exactly
     the same server path.
  4. **Both Step 4 and Step 5 name ARCHITECTURE's Integrations row.** Resolved by
     splitting it the way the code splits: this step **adds** the Stripe row (Stripe is
     live from now on), Step 5 **removes** the LiqPay row (it is still wired until then).
  5. `settle_topup` deliberately does not check the row's status — its docblock makes
     idempotency the caller's job. The controller's `pending` check is that guard, and
     test 2 is what proves it.
- **Design:** n/a — no screen.
- **Check command:** `backend/bin/check` (and `frontend/bin/check`, unaffected), run
  before every commit with DDEV up so all three scripts execute.
- **Not locally verifiable:** n/a — `stripe listen` delivers real events to DDEV, as
  the Step 2 spike proved, so the whole step is exercisable locally. Two things are
  **not this step's** and belong to the sprint's Definition of Done: registering the
  production webhook URL in the Stripe Dashboard, and the production `whsec_`, which
  differs from the CLI's.

### Questions / ambiguities

1. **What should the webhook answer when settlement itself fails?**
   `Wallet_Service::settle_topup` returns `false` when a database write fails: the
   transaction rolls back and the row stays `pending` (proved by `wallet-rollback.php`).
   The step and the 2026-09-17 decision both say a non-2xx is reserved for signature
   failures — but neither considered this case, and the tasks differ by one branch.
   - **(a) Answer 500 and let Stripe retry.** Stripe re-delivers for up to three days,
     so a transient database fault heals itself and the player gets their coins with no
     human involved. The cost: it is a third non-2xx, which the step's wording did not
     anticipate.
   - **(b) Answer 200 with `note: settle_failed` and an audit entry.** Keeps the
     decision's wording exactly. The cost: the player has paid, the row sits `pending`
     forever, and nobody finds out until someone reads the audit log.

   **Resolved: approved as recommended — (a).** A failed settlement answers
   **500 `wallet_write_failed`** (already in the registry) so Stripe retries; the row
   stays `pending` and the audit entry records it. This is a third non-2xx beyond the
   step's "signature only" wording, taken deliberately and recorded here. Test 16
   covers it: a forced settlement failure answers 500 and leaves the row `pending`.

   **Recommendation: (a).** The decision's own reasoning for answering 200 was that
   retrying an *unknown or already-settled* session is pointless — both are permanent
   conditions. A failed write is the opposite: transient, and precisely what a retry
   fixes. Answering 200 here would convert a database hiccup into a silently unpaid
   top-up, which is the one outcome the money zone exists to prevent. Under (a) the
   error code is `wallet_write_failed` 500, already in the registry; the extra test is
   "a forced settlement failure answers 500 and leaves the row `pending`".

### Execution notes (for `/close-step`)
- **Commits.** backend `0e051977`, docs `3a27739` `9cba287`. `frontend/` and `admin/`
  never left `stripe/sprint-1`. **No shared code was modified**, as planned.
- **Checks:** `stripe-webhook.php` **54**, `stripe-client.php` 39,
  `wallet-rollback.php` 53 — 146 in all under `backend/bin/check`. The new script
  passed on its first run, including the forced-failure case.
- **Verified live against real Stripe**, not only by script, with `stripe listen`
  forwarding to DDEV and `PC_STRIPE_WEBHOOK_SECRET` set to the CLI's `whsec_`:
  - a real card payment took the player from **0 → 2 coins**, row `completed`,
    `settled_at` written;
  - **resending the same event from Stripe left the balance at 2** and wrote
    `stripe_webhook_already_settled` — the sprint's Definition-of-Done behaviour,
    observed rather than assumed;
  - `stripe trigger charge.dispute.created` → 200, balance and row unchanged;
  - the audit log shows every branch exercised: `settled`, `already_settled`,
    `ignored`, `amount_mismatch`, `failed_event`, `signature_invalid`, `unsigned`,
    `settle_failed`, `unknown_session`.
- **Question 1 resolved as (a)** and implemented: a rolled-back settlement answers
  **500 `wallet_write_failed`** so Stripe retries. Test 16 proves the retry then
  settles normally.
- **Deviations from the plan:** none.
- **Deliberately not touched, each owned by Step 5:** `docs/TECH-STACK.md`'s
  ANTI-PATTERNS line about the LiqPay callback (still literally true), the LiqPay
  Integrations row (marked "being removed", not deleted), and `docs/DOMAIN.md`.
- **Local-only state:** the `stripe-step3-check` player now holds 2 coins and a
  `completed` top-up from the live test; `wp-config.php` carries a placeholder webhook
  secret again (the CLI's `whsec_` is per-session).

---

## Plan — Sprint 1, Step 5: The SPA hand-off, and LiqPay leaves   (status: closed)

### Branch
`stripe/sprint-1-handoff` ← `stripe/sprint-1`, in **all four repositories** —
`frontend/`, `backend/`, `admin/` and the root documentation repository. This is the
sprint's only step that touches every repository; they merge together into
`stripe/sprint-1` at the close, and into `main` only at the sprint boundary
(`/close-sprint`).

**Touches shared code in all three apps — may affect other features.** Every code
file here is owned by `core`; the feature's own `app/stripe/` is edited only where a
comment points at a class this step deletes. Overlap with `realtime`:
`frontend/src/stores/wallet.js` (their Sprint 2 — this step changes one docblock, no
behaviour) and nothing else. `realtime`'s own shared list names `stores/queue.js`,
`stores/chat.js` and `PlaceBet.vue`, none of which is touched.

### Tasks (ordered)

- [x] **1. The hand-off** — the player's top-up button works again. This task alone
  closes the gap that Steps 3 and 4 opened deliberately.

  `frontend/src/components/ReplenishmentBalance.vue`: the hidden-form POST becomes
  `window.location.assign(result.checkoutUrl)`. The existing "browser navigates away"
  handling stays exactly as it is — `submitting` is **not** reset on success, so the
  button stays disabled if the player comes back before Stripe's page paints. The
  `try/catch` around the old helper is replaced by the guard it existed for: a reply
  without a `checkout_url` shows the error and re-enables the button.

  `frontend/src/services/liqpayCheckout.js`: **deleted**, with its one import.

  `frontend/src/services/walletService.js`: `mapTopupResponse` becomes
  `{ transactionId, externalRef, amount, checkoutUrl }` — `orderId` and the `liqpay`
  envelope are dropped. `order_id` has not been in the response since Step 3, so
  `orderId` has been mapping `undefined`; nothing in the SPA reads either field
  (verified by grep across `frontend/src/`).

  `frontend/src/stores/wallet.js` (`startTopup` docblock) and
  `frontend/src/views/AccountView.vue:56` (the "return-from-LiqPay banner" comment):
  comment edits only, no behaviour. The `success` / `cancel` banners and
  `handleTopupReturn` stay untouched — the return page still settles nothing.

  Docs in the same task: `docs/ARCHITECTURE.md` → the `services/` row (drop
  `liqpayCheckout.js`); `docs/PROJECT-TREE.md` → the `liqpayCheckout.js` line;
  `docs/DESIGN.md` → Flows → Top-up; `docs/INVENTORY.md` → the Phase 4
  `ReplenishmentBalance.vue` highlight gets a superseding clause (see Checks).

  Gate: `frontend/bin/check` (lint, then build — the build is what proves no import
  of the deleted module survives).
  → frontend commit `feat(stripe): the player SPA hands off to Stripe's hosted page`
  → docs commit `docs: the top-up hand-off is Stripe's hosted page`

- [x] **2. LiqPay leaves the backend** — the route, the client, and every name that
  points at them.

  **Deleted whole:** `app/rest-api/PaymentController.php` (108 lines, one route,
  nothing else lives there) and `app/utils/liqpay-client.php`.

  **Loaders:** `app/rest-api.php` loses `use PC\PaymentController;` (:13), its
  `require_once` (:38) and the two registration lines (:91-92); `app/utils.php` loses
  the `liqpay-client.php` require (:15).

  **Names that would dangle:** `WalletController::topup`'s docblock (:76-85) still
  describes handing the SPA "a signed LiqPay envelope" and settling via
  `POST /payments/liqpay/callback` — rewritten to the Stripe session and the Stripe
  webhook. `app/stripe/stripe-client.php:97` cites `LiqPay_Client` as the precedent
  for injecting a parameter — reworded, because after this task the class it names
  does not exist. `app/utils/captcha-verifier.php:20` and
  `backend/wp-content/themes/pc/CAPTCHA_SETUP.md:46` both cite
  `PC_LIQPAY_PRIVATE_KEY` as the example wp-config secret → `PC_STRIPE_SECRET_KEY`.

  **Kept on purpose:** `app/utils/install-schema.php` keeps every
  `pc_liqpay_public_key` mention. `remove_retired_options()` is the upgrade path that
  *deletes* that option; removing the name would strand the option forever on any
  install still carrying it. Its comments already explain why.

  **wp-config:** `wp-config-sample.php` and `wp-config-ddev.php` each gain two
  **commented** placeholder lines for `PC_STRIPE_SECRET_KEY` and
  `PC_STRIPE_WEBHOOK_SECRET`. Commented, not defined, for two reasons: a real key must
  never be committed (core rule 4), and an empty define would read exactly like the
  absent constant to `is_configured()` while looking like configuration. This is an
  **addition**, not the replacement `SPRINT-1.md` Step 5 describes — neither file
  defines any `PC_*` constant today (`FEATURE.md` → Fit into the host, the Step 1
  delta-audit correction). The `wp-config-ddev.php` block carries a one-line note that
  `ddev start` strips it and `git checkout --` restores it (`LEARNINGS.md`
  2026-09-15), which is how `JWT_AUTH_SECRET_KEY` already lives in that file.

  Docs in the same task: `docs/CONTRACTS.md` — the whole
  `POST /pc/v1/payments/liqpay/callback` section is deleted, together with its three
  registry rows (`liqpay_payload_invalid` 400, `liqpay_signature_invalid` 401,
  `liqpay_not_configured` 500 — the last already carries the note "removed in Sprint 1
  Step 5"), and the `GET /transactions` note calling `external_ref` a "LiqPay
  order_id" is corrected to the Checkout Session id. `docs/ARCHITECTURE.md` — the ASCII
  boundary diagram, the `PaymentController.php` row of the controller table, the "Must
  never do" line, the public-route list entry, the trust-boundary bullet naming the
  LiqPay private key, the backend tree's `liqpay-client.php` line, the
  **Integrations LiqPay row deleted outright**, and the italic "*Until Sprint 1 Step 5
  the player SPA still speaks the LiqPay envelope…*" paragraph deleted. The
  "18 controllers" count is left at 18 but made precise — 17 in `app/rest-api/` plus
  the `stripe` feature's webhook controller. `docs/PROJECT-TREE.md` — the
  `PaymentController.php` and `liqpay-client.php` lines.

  Gate: `backend/bin/check`, **with DDEV running** (see Tests).
  → backend commit `refactor(stripe): remove the LiqPay integration`
  → docs commit `docs: the LiqPay callback and client are gone`

- [x] **3. Settings names Stripe** — `admin/src/views/SettingsView.vue`: the "LiqPay
  credentials" section becomes "Stripe". Static text naming the two wp-config
  constants and the webhook URL the operator must register in the Stripe Dashboard,
  rendered from the SPA's existing `VITE_API_BASE_URL` as
  `{base}/payments/stripe/webhook`, so each environment shows its own URL rather than
  a hardcoded one. The `wp option update pc_liqpay_public_key …` CLI hint goes: there
  is no provider option any more (`pc_db_version` 1.9.0 deleted it). The live
  `configured` / `mode` badge is Sprint 2 and the section says so.

  Docs in the same task: `docs/DESIGN.md` → the Settings screen row;
  `docs/ARCHITECTURE.md` → the `/settings` row; `docs/PROJECT-TREE.md` → the
  `SettingsView.vue` comment.

  Gate: `npm run lint && npm run build` in `admin/` (it has no `bin/check`).
  → admin commit `feat(stripe): Settings names the Stripe constants and the webhook URL`
  → docs commit `docs: the admin Settings hint is Stripe`

- [x] **4. The system record** — documentation only; the statements that become false
  the moment tasks 1–3 land.

  - root `CLAUDE.md` **invariant 2** — `PC_LIQPAY_PRIVATE_KEY` is replaced by
    `PC_STRIPE_SECRET_KEY` and `PC_STRIPE_WEBHOOK_SECRET`, and "LiqPay public key"
    leaves the public-counterparts sentence: the top-up provider now has **no** WP
    option at all.
  - root `CLAUDE.md` **invariant 5** — the closing sentence "The LiqPay callback still
    exists until Sprint 1 Step 5 removes it…" is deleted. It has done its job.
  - `docs/DATA-MODEL.md` **invariant 6** — "A transaction reaches `completed` only
    through the LiqPay callback, idempotent on `(order_id, status)`" becomes the
    Stripe webhook, idempotent on the row's own status. Step 3's worklog assigned this
    rewording to Step 4 and Step 4 did not carry it out; it lands here (Checks).
  - `docs/DATA-MODEL.md` audit-log event table — the `payments` row's seven
    `liqpay_*` types are replaced by the ten the Stripe webhook actually writes:
    `stripe_webhook_unsigned`, `_unconfigured`, `_signature_invalid`, `_ignored`,
    `_unknown_session`, `_already_settled`, `_amount_mismatch`, `_settled`,
    `_settle_failed`, `_failed_event`. The `pc_liqpay_public_key` sentence under
    Options stays — it records the 1.9.0 migration.
  - `docs/TECH-STACK.md` — the ANTI-PATTERNS line "Do not flip a transaction to
    `completed` anywhere but the LiqPay callback, and keep that handler idempotent on
    `(order_id, status)`" becomes the Stripe webhook and the row's status; the
    secrets line loses "LiqPay public key" from its public-counterparts parenthesis.
  - `docs/DOMAIN.md` → **Top-up**: "through Stripe's hosted Checkout page".
  - `docs/ROADMAP.md` → Phase 4 item 3 re-tagged with the provider, and the resolved
    open question "Resolved Phase 4: **LiqPay Checkout** (UAH)" gains a superseding
    clause rather than being rewritten — ROADMAP is a history document.
  - `docs/INVENTORY.md` → the Phase 4 `ReplenishmentBalance.vue` highlight (task 1)
    and nothing else; its REST-surface table is the frozen Phase 0 baseline and never
    listed the LiqPay route (Checks).
  - `docs/features/core/FEATURE.md` → **one line** under Interfaces: the top-up
    provider path has moved to the feature `stripe`, `PaymentController` and its
    callback are gone, leaving 17 controllers in `app/rest-api/`. The frozen Purpose
    & scope paragraph — "the wallet with FIFO coin lots and LiqPay top-ups" — is
    **not** edited: it describes the product as it stood on 2026-09-15.
  - `docs/features/stripe/FEATURE.md` → Shipped so far (the hand-off is live; the
    button is no longer broken), Fit into the host (the Step 5 touchpoints now read as
    done), and the UI line for Settings.
  → docs commit `docs: LiqPay is gone — invariants, domain and the frozen records`

### Files to create/change

| File | Change |
|---|---|
| `frontend/src/components/ReplenishmentBalance.vue` | `window.location.assign(checkoutUrl)`; import removed |
| `frontend/src/services/liqpayCheckout.js` | **deleted** |
| `frontend/src/services/walletService.js` | `mapTopupResponse` drops `orderId` + `liqpay`, adds `externalRef` |
| `frontend/src/stores/wallet.js` | docblock only |
| `frontend/src/views/AccountView.vue` | comment only |
| `backend/.../app/rest-api/PaymentController.php` | **deleted** |
| `backend/.../app/utils/liqpay-client.php` | **deleted** |
| `backend/.../app/rest-api.php` | `use`, `require_once`, registration removed |
| `backend/.../app/utils.php` | `require_once` removed |
| `backend/.../app/rest-api/WalletController.php` | `topup()` docblock |
| `backend/.../app/stripe/stripe-client.php` | one comment (cites a deleted class) |
| `backend/.../app/utils/captcha-verifier.php` | one comment (example secret) |
| `backend/.../CAPTCHA_SETUP.md` | one line (example secret) |
| `backend/wp-config-sample.php`, `backend/wp-config-ddev.php` | two commented placeholders each |
| `admin/src/views/SettingsView.vue` | the LiqPay section becomes Stripe |
| `docs/CONTRACTS.md` | callback section + 3 registry rows deleted; `external_ref` note |
| `docs/ARCHITECTURE.md` | diagram, controller table, must-never-do, route list, trust boundary, tree, Integrations row, the Step 5 note, `/settings`, `services/` |
| `docs/DATA-MODEL.md` | invariant 6; the `payments` audit-event row |
| `docs/PROJECT-TREE.md` | 3 deleted files + the `SettingsView.vue` comment |
| `docs/DESIGN.md` | Flows → Top-up; the Settings screen row |
| `docs/DOMAIN.md` | Top-up |
| `docs/TECH-STACK.md` | ANTI-PATTERNS line; the secrets line |
| `docs/ROADMAP.md` | Phase 4 item 3; the resolved provider question |
| `docs/INVENTORY.md` | the Phase 4 `ReplenishmentBalance.vue` highlight |
| `CLAUDE.md` | invariants 2 and 5 |
| `docs/features/core/FEATURE.md` | one line under Interfaces |
| `docs/features/stripe/FEATURE.md` | Shipped so far, Fit into the host, UI |

Unchanged on purpose: `Wallet_Service`, `Audit_Log`, `Install_Schema` (its
`pc_liqpay_public_key` names are the migration), `StripeWebhookController`,
`functions.php`, `app/stripe/bootstrap.php`, `AccountView`'s banner logic,
`docs/BACKEND-REVIEW.md`, `docs/WORKLOG.md`, `docs/DECISIONS.md`.

### Tests to write
**None new.** This step deletes code and rewires one call site; it adds no behaviour
to assert. The step's own Tests line names the three existing gates, and the sprint's
money and webhook scripts (39 + 54 + 53 = 146 checks) must keep passing untouched.

What each gate actually proves here, because it is not obvious:

- `frontend/bin/check` — **the build is the real test.** Vite resolves every import at
  build time, so a surviving `import … from '@/services/liqpayCheckout.js'` fails the
  build. Lint alone would not catch it.
- `backend/bin/check` — **Stage 2 is the real test, and it only runs when DDEV is
  running.** `php -l` cannot see a dangling `new PaymentController()`: it is valid
  syntax. Stage 2 boots WordPress through `ddev wp eval-file`, which loads
  `functions.php` → `app/rest-api.php`, so a missed `use`, `require_once` or
  registration line is a fatal that fails the script. **With DDEV down, Stage 2 prints
  a SKIPPED banner and the script still exits 0** — so no commit of task 2 is made
  without DDEV up, and the check output must show the three "All N checks passed"
  lines, not the banner.
- `admin/` — `npm run lint && npm run build`, same reasoning as the player SPA.

### Docs to update
`docs/ARCHITECTURE.md`, `docs/PROJECT-TREE.md`, `docs/DESIGN.md`, `docs/INVENTORY.md`
(task 1); `docs/CONTRACTS.md`, `docs/ARCHITECTURE.md`, `docs/PROJECT-TREE.md`
(task 2); `docs/DESIGN.md`, `docs/ARCHITECTURE.md`, `docs/PROJECT-TREE.md` (task 3);
root `CLAUDE.md`, `docs/DATA-MODEL.md`, `docs/TECH-STACK.md`, `docs/DOMAIN.md`,
`docs/ROADMAP.md`, `docs/features/core/FEATURE.md`,
`docs/features/stripe/FEATURE.md` (task 4).

### Checks
- **ANTI-PATTERNS:** none violated. The step writes no money code at all — it deletes
  a settlement path and rewires one browser navigation. The one ANTI-PATTERNS line
  that names LiqPay is rewritten by task 4 *because* task 2 makes it false; that is
  the line changing to match the code, not the code deviating from the line. No new
  route, no new option, no hardcoded operator value (the admin webhook URL is composed
  from the existing `VITE_API_BASE_URL`), no secret in the repository.
- **Docs vs reality:** mismatches found, each resolved here rather than asked about —
  - `SPRINT-1.md` Step 5 says "replace the LiqPay constant" in `wp-config-sample` /
    DDEV config. Neither file defines any `PC_*` constant, so task 2 **adds**. Already
    recorded as a correction in `FEATURE.md` → Fit into the host by the Step 1
    delta-audit; `SPRINT-1.md`'s text is left as written, as it was then.
  - `SPRINT-1.md` Step 5's verification expects `grep -ri liqpay` to find only
    `DECISIONS.md`, `WORKLOG.md`, `ROADMAP.md`, `INVENTORY.md`, `BACKEND-REVIEW.md`
    and one line in `core/FEATURE.md`. That list is incomplete in two ways, and the
    verification guide will use the corrected one: **(a)**
    `backend/.../app/utils/install-schema.php` keeps the name deliberately — it is the
    upgrade path that deletes the option; **(b)** this sprint's own records —
    `SPRINT-1.md`, `SPRINT-1-PLAN.md`, `SPRINT-2.md` and the Step 1 / Step 3
    verification guides — are history in the same sense as `WORKLOG.md`.
  - `docs/DATA-MODEL.md` invariant 6 still names the LiqPay callback, and its
    audit-log `payments` row still lists only `liqpay_*` events. Step 3's worklog
    assigned the rewording to Step 4; Step 4's docs list did not carry it and it was
    missed. Step 5 is where it stops being merely stale and becomes false, and the
    sprint's Definition of Done names DATA-MODEL, so task 4 fixes both. DATA-MODEL is
    not in Step 5's own "Docs to update" list — it is here under core rule 5, as
    documentation describing code this step changes.
  - Same reasoning for root `CLAUDE.md` (invariants 2 and 5 name a constant and a
    route this step deletes) and `docs/CONTRACTS.md` (a shipped route leaving the
    surface takes its contract and its error codes with it — root `CLAUDE.md`'s
    documentation table makes that same-commit work).
  - `docs/INVENTORY.md` → REST surface is the **frozen Phase 0 baseline** and never
    listed `POST /payments/liqpay/callback`; its own closing sentence says so. Step 5's
    "INVENTORY REST surface" instruction therefore has nothing to change there. The
    single LiqPay mention is a Phase 4 highlight, which task 1 gives a superseding
    clause.
  - `docs/BACKEND-REVIEW.md` items 2 ("a duplicate LiqPay notification can credit
    coins twice") and 11 ("LiqPay `sandbox` payments count as real money") describe
    defects in code this step deletes. `SPRINT-1.md` routes BACKEND-REVIEW items to
    `/adhoc` and its grep list keeps the file as history, so they are **not** touched
    here — flagged for `/close-sprint` or an `/adhoc` to mark resolved-by-removal.
  - `docs/features/stripe/FEATURE.md` remains over its ≤80-line guideline
    (`LEARNINGS.md` 2026-09-17). Task 4 edits it without growing it.
- **Design:** matches. No new screen and no new token. **Replenishment balance**
  changes only where it navigates; **Account** is untouched; **Settings** (admin)
  swaps one static section — the three screens `docs/DESIGN.md` and
  `FEATURE.md` → UI already name for this sprint. `design/` stays empty
  (`DECISIONS.md` 2026-09-17 "`stripe` has no UI design").
- **Check command:** `backend/bin/check` and `frontend/bin/check`
  (`docs/TECH-STACK.md` → Check command); `admin/` has none — its gate is
  `npm run lint && npm run build`.
- **Not locally verifiable:** n/a. Every claim of this step is verifiable on the DDEV
  site and the two dev servers. The deployed-stack payment and the registration of the
  production webhook URL in the Stripe Dashboard are the **sprint's** Definition of
  Done, not this step's.

### Questions / ambiguities
none.
