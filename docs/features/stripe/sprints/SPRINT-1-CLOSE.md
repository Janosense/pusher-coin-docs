# SPRINT 1 CLOSE — `stripe`: A player buys coins through Stripe (2026-09-17)

Handoff to the retro, to `/plan-step stripe 2 1`, and to whoever reads this after the
deploy. Written from the files, the code and git — not from a session's memory.

**Branch:** `stripe/sprint-1` ← `main`, five task branches merged `--no-ff` and deleted.
The same branch name existed in all four repositories; `backend/` carries four step
merges, the root docs repository five, `frontend/` and `admin/` two each.

---

## Definition of Done

- [x] **Every step closed via /close-step** — five steps, five verification guides
  (`verification/sprint-1-step-{1..5}.md`), five WORKLOG entries, five `--no-ff` merges.
  Docs `acc2491` `2c574a9` `0633cf6` `e29ddaa` `044a227`; backend `967d45c5` `2736a5b9`
  `3c45000a` `83f8fd9e`; frontend `5d753db` `1d12f24`; admin `4496f55` `5389287`.
- [x] **The check command exits 0** on `stripe/sprint-1` in `backend/` and `frontend/`;
  `admin/` passes `npm run lint && npm run build`. Run at this close:
  `php -l: 48 files OK`, then `All 39 / 54 / 53 checks passed` (**146**), exit 0 in all
  three. Stage 2 ran against a live DDEV — it had to, because a dangling reference to a
  deleted class is valid PHP and only a real WordPress start-up catches it.
- [x] **Docs match reality** — DATA-MODEL, ARCHITECTURE, CONTRACTS, DOMAIN and
  DECISIONS were updated in the step that changed the code they describe (the five
  closes' self-checks). LiqPay verified gone **per repository** with `git grep`, not
  with a repo-root search (see Contradictions 1): `frontend` none, `admin` none,
  `backend` exactly seven deliberate lines — the option-deleting migration in
  `install-schema.php` (5), the "LiqPay-era rows still render" note in
  `WalletController.php` (1), one decision citation in `StripeWebhookController.php` (1).
- [ ] **Sprint boundary: merged into `main`, `backend` and `frontend` `main` pushed;
  `PC_STRIPE_SECRET_KEY` and `PC_STRIPE_WEBHOOK_SECRET` in production `wp-config.php`
  and the production webhook URL registered in the Stripe Dashboard before the merge**
  — `open — sprint boundary`. The repository side is done (merge commits in the report
  below); **nothing is pushed**. The production wp-config and the Dashboard endpoint are
  server-side facts no command here can observe.
- [ ] **Tymofii paid with a test card on the deployed stack and saw the coins without
  reloading by hand** — `open — sprint boundary`. The equivalent was observed locally in
  Step 4 (0 → 2 coins, row `completed`); the deployed stack has never run this code.
- [ ] **The same event resent from the Stripe Dashboard credited nothing a second time,
  observed in that player's balance** — `open — sprint boundary`. The local equivalent
  is evidenced: Step 4 resent the identical event with `stripe events resend`, the
  balance stayed at 2 and the audit row read `stripe_webhook_already_settled`. What is
  missing is the Dashboard repeat against the deployed stack.
- [x] **Step 2 closed with a `DECISIONS.md` entry naming the account country, the `uah`
  answer and the pinned API version** — `DECISIONS.md` 2026-09-17 "Spike: Stripe accepts
  `uah`, on a provisional US sandbox; pin `2026-06-24.dahlia` and turn Adaptive Pricing
  off": account `acct_1TtSrOElMyJqvLDl` (US), HTTP 200 with `currency: uah` and
  `amount_total: 12000`, API version `2026-06-24.dahlia`.

**4 of 7 ticked.** The three open items are all the same fact: this code has never run
anywhere but a laptop.

---

## Built

What Sprint 2 and every later sprint inherit.

**The feature's own code — `backend/wp-content/themes/pc/app/stripe/`**
- `bootstrap.php` — the single entry point, one `require_once` in `functions.php`. It
  registers the feature's REST routes on `rest_api_init` itself, so `core`'s
  `app/rest-api.php` is untouched. **Sprint 2's `AdminTopupController` registers here.**
- `stripe-client.php` — `Stripe_Client`, the only code that talks to Stripe. No SDK;
  `wp_remote_post` against `https://api.stripe.com/v1`, `Stripe-Version:
  2026-06-24.dahlia`. `is_configured()`, `mode()` (`test` / `live` from the key prefix),
  `to_kopiykas()` (`bcmul` on the decimal string), `create_checkout_session()` (injects
  `adaptive_pricing[enabled]=false` so no caller can forget it; typed
  `stripe_call_failed` that never echoes the key), `verify_signature()` (HMAC-SHA256
  over `{timestamp}.{raw body}`, parses the header by key name, 300s tolerance).
  **`is_configured()` and `mode()` are what Sprint 2's status endpoint returns.**
- `StripeWebhookController.php` — `POST /pc/v1/payments/stripe/webhook`, the only place
  a top-up reaches `completed`.

**Shared code changed (owned by `core`)**
- `Wallet_Service::set_external_ref( int $txn_id, string $ref ): bool` — new; it
  replaced the raw `$wpdb->update` at `WalletController.php:127` and with it **the last
  money-column write outside `Wallet_Service` in the codebase**.
- `WalletController::topup` — rewritten. Answers
  `{ transaction_id, external_ref, amount, checkout_url }`; `stripe_not_configured` 500
  before anything is written; a failed session parks the row `failed` and answers 502.
- `Install_Schema` — `DB_VERSION` `1.8.0` → **`1.9.0`** and a new
  `remove_retired_options()` deleting `pc_liqpay_public_key`. **The installer had no
  upgrade path at all before this**; anything retiring an option now has one to extend.
- `frontend/src/components/ReplenishmentBalance.vue` → `window.location.assign(checkoutUrl)`;
  `services/walletService.js` → `{ transactionId, externalRef, amount, checkoutUrl }`.
- `admin/src/views/SettingsView.vue` — a static Stripe section naming the two constants
  and the webhook URL, composed from `VITE_API_BASE_URL`. **Sprint 2 Step 2 adds the
  live badge here.**

**The check command** — `backend/bin/check` (php -l, then every `tests/*.php` through
`ddev wp eval-file`) and `frontend/bin/check` (lint, then build), copied byte-identical
from `realtime/sprint-1`; `lint` / `lint:fix` split in both `package.json` files.
`admin/` still has none — its gate is `npm run lint && npm run build`.

**Tests** — `tests/stripe-client.php` (39 checks) and `tests/stripe-webhook.php` (54),
joining `wallet-rollback.php` (53). **146 total**, picked up automatically by
`backend/bin/check`. The webhook script signs its own fixtures and drives the controller
through `WP_REST_Request` — no HTTP, no Stripe.

**Data** — no table changed. `external_ref` on a top-up is now a Checkout Session id
(`cs_…`, 66 chars observed); LiqPay-era rows keep `pc-topup-N` and still render.
`wp_pc_transactions` has **no currency column**, so the webhook compares against the
constant `'uah'`. Ten `stripe_webhook_*` audit event types.

**Deleted** — `app/rest-api/PaymentController.php`, `app/utils/liqpay-client.php`,
`frontend/src/services/liqpayCheckout.js`, the WP option `pc_liqpay_public_key`, and the
`PC_LIQPAY_PRIVATE_KEY` constant from every document that named it.
`POST /pc/v1/payments/liqpay/callback` answers 404.

---

## Not locally verifiable

- **The deployed stack has never run any of this.** The FTP release (`backend` `main`
  pushed) and the Vercel build (`frontend` `main` pushed) are the first real run;
  Definition of Done items 4-6 are what that run verifies.
- **The production webhook endpoint and its signing secret.** Registered by hand in the
  Stripe Dashboard, per environment. The local `whsec_` comes from `stripe listen` and
  changes every restart; it is **not** the production one.
- **`uah` acceptance is provisional.** It was proved on `acct_1TtSrOElMyJqvLDl`, a **US
  sandbox** the Stripe CLI happened to be logged into — not the client's account, which
  is still unnamed. The presentment-currency list is per account country, so this must
  be re-verified on the real account before go-live. `DECISIONS.md` 2026-09-17 says so
  in its own words.
- **The player-facing hand-off was never exercised through a browser by the agent** —
  signing in as a player needs a password. What is verified: the shipped bundle contains
  `location.assign(checkoutUrl)` and zero LiqPay, and the endpoint it calls returns a
  `checkout_url`. The screen-level confirmation is §3 of the Step 5 guide.

---

## Deferred

From the sprint's own Out of scope, its plans and its worklog entries:

- **The admin Top-ups list, `GET /admin/topups`, `GET /admin/stripe/status` and the live
  Settings badge** → **Sprint 2** (`SPRINT-2.md` exists and plans exactly these).
- **Anything on a chargeback or a refund** → out of v1 (`DECISIONS.md` 2026-09-16). The
  webhook acknowledges those events and does nothing.
- **Withdrawals through Stripe** → out of v1; the 2026-05-13 decision (manual payouts,
  no KYC) stands.
- **The client's live Stripe account and live keys** → the client's, not a step.
  Production can only carry test-mode keys until it is named, and the `test` badge
  Sprint 2 adds is the intended visible warning.
- **`docs/BACKEND-REVIEW.md` items outside the top-up path** → `/adhoc`, as before.
  Items 2 and 11 are now *inside* it and describe deleted code (see Contradictions 3).
- **`docs/features/stripe/FEATURE.md` is 117 lines against an ≤80-line guideline** —
  `LEARNINGS.md` 2026-09-17. The delta-audit's file:line record is what overflows it.
- **`docs/PROJECT-TREE.md` lists a `frontend/CLAUDE.md`** (line 143) that does not exist
  in any of the three repositories — pre-existing drift, never in this sprint's scope.
  An `/adhoc` candidate.
- **The sandbox `sk_test_` key expires 2026-10-13.**

No Sprint 0 exists, so nothing was carried into this sprint.

---

## Contradictions

Listed, not resolved.

1. **`SPRINT-1.md` Step 5 → Verification vs. reality.** It expects
   `grep -ri liqpay` across the three repositories to find only `DECISIONS.md`,
   `WORKLOG.md`, `ROADMAP.md`, `INVENTORY.md`, `BACKEND-REVIEW.md` and one line in
   `core/FEATURE.md`. The true set also contains `backend`'s three files (seven
   deliberate lines) and this sprint's own records (`SPRINT-1.md`, `SPRINT-1-PLAN.md`,
   `SPRINT-2.md`, two verification guides). Worse, run from the project root that grep
   searches **none** of the three applications, because the root `.gitignore` lists
   them — it returns only `docs/*` and reads as a pass. `verification/sprint-1-step-5.md`
   carries the corrected, per-repository check; `SPRINT-1.md` is unchanged.
   Also `LEARNINGS.md` 2026-09-17.
2. **`SPRINT-1.md` Step 5 says "replace the LiqPay constant"** in `wp-config-sample.php`
   / `wp-config-ddev.php`; **`FEATURE.md` → Fit into the host** records (from the Step 1
   delta-audit) that neither file defines any `PC_*` constant, so it is an **addition**.
   Step 5 added two commented placeholders. `SPRINT-1.md`'s text was left as written.
3. **`docs/BACKEND-REVIEW.md` items 2 and 11** describe LiqPay defects ("a duplicate
   LiqPay notification can credit coins twice", "LiqPay `sandbox` payments count as real
   money") in code this sprint deleted. `SPRINT-1.md` → Out of scope routes
   BACKEND-REVIEW items to `/adhoc` and its grep list keeps the file as history, so
   neither the items nor the file were touched.
4. **`docs/INVENTORY.md` → CI status** describes the frontend CI as running
   `npm run lint`. Still true by name, but Step 1 changed what that script does — `lint`
   became report-only and fixing moved to `lint:fix`, so CI now reports fixable errors
   instead of repairing them. INVENTORY does not say so.
5. **`docs/DESIGN.md` → Screens has no Top-ups row** while **`FEATURE.md` → UI** lists
   **Top-ups** as one of the feature's screens. Sequenced rather than contradictory —
   `SPRINT-2.md` Step 2 adds the DESIGN row — but the two files disagree today.

---

## LEARNINGS — entries of this sprint still `pending`

Each needs "Transferred to playbook" filled at the retro (a version / `local` / `n/a`).

1. **2026-09-17 — A repo-root `grep` skips all three app repositories, so "it is gone"
   was almost verified against nothing.** The generalisable rule: a repo-root recursive
   search is not evidence when the tree contains nested repositories or ignored app
   directories.
2. **2026-09-17 — "Pay with the test card" collides with a standing prohibition on
   entering card numbers.** Proposed: a sprint step that requires a test instrument
   names it as such ("Stripe's published test card, sandbox, `livemode: false`"), so the
   instruction carries its own justification.
3. **2026-09-17 — A real delta-audit does not fit FEATURE.md's 80-line guideline.**

(`2026-09-16 — [core] An ad-hoc off `main` had no check command to gate its commits` is
also `pending`, but belongs to `core`, not this sprint.)
