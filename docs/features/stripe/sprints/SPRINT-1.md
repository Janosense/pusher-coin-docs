# SPRINT 1 — A player buys coins through Stripe (2026-09-17 → open)

**Branch:** `stripe/sprint-1` ← `main`, task branches `stripe/sprint-1-{short-name}` merged `--no-ff`. The same branch name is used in every repository a step touches (`backend/`, `frontend/`, `admin/`, the root docs repository); they merge together at the sprint boundary. `realtime/sprint-1` is open in parallel — the two features share `Wallet_Service` and `stores/wallet.js` only.
**Goal:** A player who presses "top up" lands on Stripe's hosted payment page, pays in UAH, and comes back to find the coins in their wallet — credited by the Stripe webhook, never by the return page. A re-delivered event credits nothing twice; an abandoned session ends up `failed`, not `pending` forever. LiqPay is gone from all three repositories, while every LiqPay-era row still shows in the player's History. The sprint branch carries the project's check command, and every commit goes through it.

## Fixed decisions
- Stripe replaces LiqPay for top-ups and Stripe's prohibited-business policy is an accepted business risk — `DECISIONS.md` 2026-09-16. Do not reopen the provider choice inside a step, whatever the Dashboard says during onboarding.
- LiqPay is **removed**, not parked — `DECISIONS.md` 2026-09-16 "LiqPay is removed, not parked". No `if (liqpay)` branch survives.
- The player pays UAH and the wallet stays UAH; amounts to Stripe are kopiykas from the `DECIMAL(12,2)` string — `DECISIONS.md` 2026-09-16 "Top-ups stay UAH end to end under Stripe". If Step 2 finds `uah` refused for the account's country, stop and raise it — do not switch currency in code.
- Chargebacks and refunds change nothing in v1 — `DECISIONS.md` 2026-09-16.
- Hosted Checkout, no SDK, the webhook contract, the two wp-config constants, the `app/stripe/` location, the WP-CLI test script and the Stripe CLI for local delivery — the seven `DECISIONS.md` entries of 2026-09-17. The `$wpdb->update` bypass in `WalletController::topup` moves into `Wallet_Service` in Step 3 — decided, not optional.
- The check command arrives as byte-identical copies of `realtime/sprint-1`'s files, never as a cherry-pick onto `main` — `DECISIONS.md` 2026-09-17 "The check command reaches `stripe/sprint-1` as copied files".
- Every wallet mutation goes through `Wallet_Service` under `SELECT … FOR UPDATE`; money is never a float — root `CLAUDE.md` invariants 3 and 7.

## Steps

### [x] Step 1 — Delta-audit and the check command on the sprint branch
- **Tasks:**
  - Read `docs/features/core/FEATURE.md` (Interfaces, Invariants), `docs/features/realtime/FEATURE.md` (Shared code, Roadmap) and the shared code this feature touches: `app/utils/wallet-service.php`, `app/rest-api/WalletController.php`, `app/rest-api/PaymentController.php`, `app/utils/liqpay-client.php`, `app/utils/install-schema.php`; `frontend/src/components/ReplenishmentBalance.vue`, `stores/wallet.js`, `views/AccountView.vue`, `services/liqpayCheckout.js`; `admin/src/views/SettingsView.vue`, `admin/src/router/index.js`, `admin/src/components/AdminLayout.vue`. Confirm or correct every touchpoint listed in `FEATURE.md` → Fit into the host, including the `$wpdb->update` bypass and each place `realtime` will touch too. No code changes.
  - Copy from `realtime/sprint-1`, unchanged: `backend/bin/check`, `frontend/bin/check`, the `lint` / `lint:fix` scripts in `frontend/package.json` and `admin/package.json`, and the matching text of `docs/TECH-STACK.md` → Check command, `CLAUDE.md` → Commands and `docs/PROJECT-TREE.md`. *Touches shared code:* the lint scripts and the check command gate every later commit.
  - Add the `stripe` row to the Features table of the root `CLAUDE.md` and to `docs/ARCHITECTURE.md` → Feature map if discovery's row is missing after the merge with `main`.
- **Tests:** None of its own. `backend/bin/check` on the copied files must run `tests/wallet-rollback.php` and show its checks, exactly as on the realtime branch.
- **Verification (manual):** `git diff realtime/sprint-1 -- backend/bin/check frontend/bin/check` is empty. `backend/bin/check` and `frontend/bin/check` exit 0 on the clean sprint branch. `FEATURE.md` → Shared code lists nothing the audit could not point at in the code.
- **Docs to update:** `docs/features/stripe/FEATURE.md` → Fit into the host; `docs/TECH-STACK.md` → Check command; `CLAUDE.md` → Commands; `docs/PROJECT-TREE.md`.
- **Depends on:** —

### [x] Step 2 — Spike: Stripe on the test keys
- **Tasks:**
  - Timeboxed to one working session. Spike code is throwaway and is not merged into `main`; the keys Tymofii provides go into the local `wp-config-ddev.php` only, never into a commit.
  - Record the account's country (Dashboard → Settings) — the presentment-currency list depends on it.
  - Create a Checkout Session with a raw HTTPS call (`curl` or a throwaway `wp eval`): `mode=payment`, one line item in `uah`, an amount in kopiykas, `client_reference_id`, `success_url` and `cancel_url` on the SPA base URL. Record the full response, including `url`, `expires_at` and `payment_status`. **The decisive question: does the API accept `uah` for this account?** If it refuses, record the exact error and stop the sprint at this step — the currency decision is reopened, not patched.
  - Pin an API version: read the account's default version in the Dashboard and the latest dated version in Stripe's changelog; choose one and record it for the `Stripe-Version` constant.
  - Run `stripe listen --forward-to https://pusher-coin.ddev.site/wp-json/pc/v1/payments/stripe/webhook` with a throwaway route that logs headers and raw body; pay the session with the `4242 4242 4242 4242` test card; record the `Stripe-Signature` header shape, the `checkout.session.completed` payload (`id`, `payment_status`, `amount_total`, `currency`, `client_reference_id`) and the delay between paying and delivery. Verify one signature by hand with the CLI's `whsec_` to prove the manual scheme works here.
  - Expire a session (`expires_at` at its minimum, or Stripe's "expire" API) and record whether `checkout.session.expired` arrives and when.
  - Write the result as a `docs/DECISIONS.md` entry: the account country, the `uah` answer with the recorded response, the pinned API version, the observed event shapes and timings.
- **Tests:** —
- **Verification (manual):** Read the new `DECISIONS.md` entry. It names the account country, quotes the API's answer to `uah` (accepted, with the session `url`; or the refusal verbatim), names one API version, and shows one real `checkout.session.completed` payload and one `checkout.session.expired` observation. If it says "should work", the step is not done.
- **Docs to update:** `docs/DECISIONS.md` (the spike entry); `docs/LEARNINGS.md` if the CLI or DDEV needed a workaround.
- **Depends on:** —

### [x] Step 3 — The Stripe client and the Checkout Session
- **Tasks:**
  - `app/stripe/bootstrap.php`, required once from `functions.php` (the single entry point). `app/stripe/stripe-client.php`: `is_configured()` (both constants present), `mode()` (`test` / `live` from the key prefix), `create_checkout_session()` over `wp_remote_post` with a Bearer key, the pinned `Stripe-Version` header, an `Idempotency-Key` of `pc-topup-{txn_id}`, and a typed `WP_Error` on any non-2xx (`stripe_call_failed`) that never echoes the key; `verify_signature( string $raw_body, string $header, string $secret, int $tolerance = 300 ): bool` per the 2026-09-17 decision.
  - `WalletController::topup` — keep the validation and the `pending` row exactly as they are; then create the session with the amount in kopiykas (`bcmul` on the decimal string, never a float), `currency=uah`, `client_reference_id = txn_id`, `success_url` = `{pc_spa_base_url}account?topup=success`, `cancel_url` = `{pc_spa_base_url}account?topup=cancel`; store the session id as `external_ref` through a new `Wallet_Service::set_external_ref( int $txn_id, string $ref ): bool` (replaces the `$wpdb->update`; checked like every other write); respond `{ transaction_id, external_ref, amount, checkout_url }`. Unconfigured → `stripe_not_configured` 500. A failed session creation marks the row `failed` with the error code as the note and answers 502. *Touches shared code:* `WalletController`, `Wallet_Service`.
  - `Install_Schema`: remove `pc_liqpay_public_key` (`delete_option` in the upgrade path), bump `DB_VERSION`. *Touches shared code.*
  - `tests/stripe-client.php` (WP-CLI eval): the kopiyka conversion for `0.01`, `10.00`, `499.99`, `123456.78`; `mode()` for `sk_test_…` and `sk_live_…`; `is_configured()` with one constant missing; `set_external_ref` on a missing row returns false.
- **Tests:** Money zone — the script above ships in this step and runs under `backend/bin/check`. Nothing in this step touches `settle_topup`.
- **Verification (manual):** With the test keys in `wp-config-ddev.php`, `POST /pc/v1/wallet/topup` from the SPA's dev server returns a `checkout_url` that opens Stripe's page showing the right number of coins and the right UAH amount; the new `wp_pc_transactions` row is `pending` with a `cs_…` `external_ref`. Remove one constant and repeat — `stripe_not_configured`, no row written.
- **Docs to update:** `docs/CONTRACTS.md` (`POST /wallet/topup` response and error codes; `liqpay_not_configured` retired); `docs/DATA-MODEL.md` (constants, the removed option, `DB_VERSION`, the `external_ref` convention); `docs/PROJECT-TREE.md` (`app/stripe/`, `tests/stripe-client.php`).
- **Depends on:** Step 1, Step 2

### [x] Step 4 — The webhook and settlement
- **Tasks:**
  - `app/stripe/StripeWebhookController.php`: `POST /pc/v1/payments/stripe/webhook`, `permission_callback` `__return_true` with the signature as the credential — read the raw body (`$request->get_body()`), verify with `PC_STRIPE_WEBHOOK_SECRET`; a missing or invalid signature is 400/401 and the only non-2xx the handler ever returns.
  - Event routing per the 2026-09-17 decision: `checkout.session.completed` and `checkout.session.async_payment_succeeded` with `payment_status = paid` → look up the row by `external_ref`; unknown → 200 `note: unknown_session`; not `pending` → 200 `note: already_settled`; `amount_total` or `currency` differing from the row → 200 `note: amount_mismatch`, `Audit_Log`, no settlement; otherwise `Wallet_Service::settle_topup`. `checkout.session.async_payment_failed` and `checkout.session.expired` → `failed` with the event type as the note (from `pending` only). Any other type → 200 `note: ignored`. Every branch writes `Audit_Log` with the event id, type and session id in metadata.
  - `tests/stripe-webhook.php` (WP-CLI eval), signing its own fixtures with a known secret: valid signature settles a pending row once and the wallet gains the lot; the same event again → `already_settled`, balance unchanged; invalid signature → rejected, nothing written; timestamp older than the tolerance → rejected; amount mismatch → no settlement; `expired` → row `failed`; `expired` on a `completed` row → unchanged; a `charge.dispute.created` fixture → 200, nothing changes.
- **Tests:** Money zone and the test-critical webhook — the script above is the gate. `settle_topup` itself is covered by the existing `wallet-rollback.php`.
- **Verification (manual):** `stripe listen` forwarding to DDEV; pay a session created through the SPA — the wallet shows the coins within seconds and the row is `completed`. In the Stripe Dashboard, resend the same event — the balance does not move and the audit log shows `already_settled`. Let a session expire — the row becomes `failed`. Send `stripe trigger charge.dispute.created` — 200 and nothing changes.
- **Docs to update:** `docs/CONTRACTS.md` (the webhook, its notes and error codes, moved into "current"); root `CLAUDE.md` invariant 5 and the test-critical zones line (LiqPay → Stripe); `docs/ARCHITECTURE.md` → Top-up data flow and Integrations row; `docs/PROJECT-TREE.md`.
- **Depends on:** Step 3

### [x] Step 5 — The SPA hand-off, and LiqPay leaves
- **Tasks:**
  - `frontend/src/components/ReplenishmentBalance.vue`: on a successful `startTopup`, `window.location.assign(result.checkout_url)`; the existing "browser navigates away" handling stays. Delete `services/liqpayCheckout.js` and every import of it. `AccountView`'s `success` / `cancel` banners stay as they are. *Touches shared code.*
  - Backend: delete `app/rest-api/PaymentController.php`'s LiqPay route (the whole file if nothing else lives there), `app/utils/liqpay-client.php`, every `PC_LIQPAY_PRIVATE_KEY` and `LiqPay_Client` reference, and the LiqPay lines of `backend/wp-content/themes/pc/README*` or setup notes if any. `wp-config-sample` / DDEV config: replace the LiqPay constant with the two Stripe ones. *Touches shared code.*
  - `admin/src/views/SettingsView.vue`: the "LiqPay credentials" section becomes a "Stripe" section with static text naming the two wp-config constants and the webhook URL to register; the live status badge is Sprint 2. *Touches shared code.*
  - Docs: `docs/DOMAIN.md` → Top-up ("through Stripe's hosted page"); `docs/ARCHITECTURE.md` → services table, Integrations (LiqPay row → Stripe row), Trust boundaries; `docs/DESIGN.md` → Flows → Top-up and the Settings row; `docs/INVENTORY.md` REST surface; `docs/ROADMAP.md` (Phase 4 top-up item re-tagged with the provider); `docs/PROJECT-TREE.md`; the `docs/TECH-STACK.md` ANTI-PATTERNS line "Do not flip a transaction to `completed` anywhere but the LiqPay callback" → the Stripe webhook; `docs/features/core/FEATURE.md` → Interfaces: one line saying the top-up provider path moved to `stripe` (the frozen record stays otherwise untouched).
- **Tests:** `frontend/bin/check` and `admin` lint + build pass with the LiqPay files gone; `backend/bin/check` passes with both Stripe scripts.
- **Verification (manual):** On the **Replenishment balance** screen, choose coins and press top up — the browser lands on Stripe's page; pay — back on **Account** with the success banner and the new balance; press the browser's back button from Stripe instead — the cancel banner, balance unchanged. `POST /pc/v1/payments/liqpay/callback` answers 404. The **History** screen still lists the LiqPay-era top-ups seeded in the local database. `grep -ri liqpay` across the three repositories finds only history: `docs/DECISIONS.md`, `docs/WORKLOG.md`, `docs/ROADMAP.md`, `docs/INVENTORY.md`, `docs/BACKEND-REVIEW.md` and the one line in `docs/features/core/FEATURE.md`.
- **Docs to update:** everything listed under Tasks; `docs/features/stripe/FEATURE.md` if a touchpoint changed.
- **Depends on:** Step 4

## Definition of Done
- [ ] Every step closed via /close-step (report + verification guide + worklog)
- [ ] The check command (`docs/TECH-STACK.md` → Check command) exits 0 on the sprint branch, in both `backend/` and `frontend/`; `admin/` passes `npm run lint && npm run build`
- [ ] Docs match reality (DATA-MODEL, ARCHITECTURE, CONTRACTS, DOMAIN, DECISIONS current; no LiqPay outside history entries)
- [ ] Sprint boundary: the sprint's work merged into `main` per the git model, `backend` `main` pushed (the production FTP release) and `frontend` `main` pushed (the Vercel build). `PC_STRIPE_SECRET_KEY` and `PC_STRIPE_WEBHOOK_SECRET` exist in production `wp-config.php` and the production webhook URL is registered in the Stripe Dashboard **before** the merge — otherwise the first release answers `stripe_not_configured`.
- [ ] Tymofii paid with a test card on the deployed stack and saw the coins in the wallet without reloading anything by hand
- [ ] The same event resent from the Stripe Dashboard credited nothing a second time, observed in that player's balance
- [ ] Step 2 closed with a `DECISIONS.md` entry that names the account country, the `uah` answer and the pinned API version

## Out of scope
- The admin **Top-ups** list, `GET /admin/topups`, `GET /admin/stripe/status` and the live status badge in Settings — Sprint 2.
- Anything on a chargeback or a refund — out of v1 (`DECISIONS.md` 2026-09-16); a later feature.
- Withdrawals through Stripe — out of v1; the 2026-05-13 decision stands.
- The client's live Stripe account and live keys — the client's, not a step; production runs on test keys until then.
- `BACKEND-REVIEW.md` items outside the top-up path — `/adhoc`, as before.

## Risks / notes
- **Stripe's policy names this business as prohibited.** A test account may be reviewed or closed at any point; if it is, the sprint pauses at the step it is in and the record says why — nothing is worked around.
- **`uah` acceptance is unknown until Step 2** and depends on the account's country. Chase it on day one; Step 3 must not start on a guess.
- Two open steps in two features: `realtime` Step 2 is waiting on the machine. Neither branch pushes `main` outside its own sprint boundary.
- `ddev start` rewrites `wp-config-ddev.php` and strips constants (`docs/LEARNINGS.md` 2026-09-15) — expect to restore `PC_STRIPE_*` too.
- The Stripe CLI's `whsec_` is not the Dashboard's; the local config carries the CLI's, production carries the Dashboard's.
- A push to `backend` `main` is a production deploy: nothing in this sprint is merged there with LiqPay half-removed or the constants missing.
