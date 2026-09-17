# Feature — stripe

<!-- Lightweight ARCHITECTURE + DATA-MODEL for one feature. Lives at
     docs/features/stripe/FEATURE.md next to its sprints/. Root
     ARCHITECTURE.md holds only one row + a link here. Keep ≤80 lines. -->

## Purpose & scope
Replaces LiqPay with Stripe as the only way a player buys coins, and gives the
operator a list of top-ups in the admin SPA. The client asked for Stripe without
a stated reason; Stripe's own prohibited-business list names games of chance
with a monetary prize, and the client accepted that risk (`DECISIONS.md`
2026-09-16). The Stripe account, its country and its owner are still unnamed,
so v1 is built and verified on test keys Tymofii provides; live keys are a
server configuration change, not a code change.

**Out of scope:** withdrawals through Stripe (the 2026-05-13 decision — manual
payouts, no KYC — stands); anything on a chargeback or a refund (the product does
nothing, as today); any currency but UAH; a second provider next to Stripe;
parking LiqPay for a later return — it is removed.

## Fit into the host
- **Code location:** `backend/wp-content/themes/pc/app/stripe/`, `admin/src/views/TopupsView.vue`, `admin/src/services/adminTopupService.js`
- **Host area:** the repository root — the project's single code area, governed by the root `CLAUDE.md`
- **Entry point:** one `require_once TEMPLATE_DIR . '/app/stripe/bootstrap.php'` line in `functions.php`; in the admin SPA, one route and one nav link
- **Shared code it depends on** — all owned by `core`; touching any of it is a
  plan task marked **"touches shared code"**, and the money zone is
  test-critical. Delta-audit 2026-09-17 (Sprint 1 Step 1), file:line against
  `main`; the step that changes each is named:
  - Backend: `Wallet_Service` — `settle_topup:178`, `update_transaction_status:146`,
    `find_transaction_by_external_ref:137` exist and are reused as they are;
    `set_external_ref` does **not** exist, new in Step 3. `WalletController::topup:83`
    rewritten in place — the `$wpdb->update` bypass is real at `:127`, and the
    response also carries `order_id` + `liqpay` (Step 3). `PaymentController.php` is
    the LiqPay callback and **nothing else** (108 lines, one route), so it goes whole
    with `liqpay-client.php` in Step 5 — plus their loaders `app/rest-api.php:38,91`
    and `app/utils.php:15`. `Install_Schema` — `pc_liqpay_public_key:249`,
    `DB_VERSION:15` (`1.8.0`), `pc_spa_base_url:242` for the return URLs (Step 3).
    `Audit_Log` reused unchanged (Step 4).
  - Player SPA: `components/ReplenishmentBalance.vue:4,72-79`;
    `services/walletService.js:22-31` — **not previously listed**: `mapTopupResponse`
    maps `order_id` and the `liqpay` envelope, so the response-shape change lands
    here too (Steps 3, 5); `stores/wallet.js:44-61` (`startTopup` passes the result
    through — a comment edit only); `views/AccountView.vue:56-95,117` (the banner,
    unchanged); `services/liqpayCheckout.js` removed.
  - Admin SPA: `views/SettingsView.vue:209-218` (LiqPay section → Stripe, Step 5).
    `router/index.js` and `AdminLayout.vue` carry **no** LiqPay reference — Sprint 2
    touchpoints only, for the `/topups` route and its nav link.
  - Also carrying the name: `CAPTCHA_SETUP.md:46` and `app/utils/captcha-verifier.php:20`
    cite `PC_LIQPAY_PRIVATE_KEY` as the example wp-config secret — edits in Step 5.
  - **Correction:** Step 5 expects to "replace the LiqPay constant" in
    `wp-config-sample.php` / `wp-config-ddev.php`; both are tracked and neither
    defines any `PC_*` constant today, so that task **adds** the two Stripe
    constants rather than replacing anything.
  - Overlap with `realtime`: `stores/wallet.js` (their Sprint 2) and `Wallet_Service`
    (they credit lots through `Machine_Ingest_Service`). Their own audit claims
    neither `WalletController` nor the top-up path — no collision in Sprint 1.

## Data
Owns no table; writes `wp_pc_transactions` and coin lots **only through
`Wallet_Service`**. Owns the wp-config constants `PC_STRIPE_SECRET_KEY` and
`PC_STRIPE_WEBHOOK_SECRET` (never options, never logged) and the top-up `external_ref`
convention — a Checkout Session id (`cs_…`; LiqPay-era rows keep `pc-topup-N`).
Removes the WP option `pc_liqpay_public_key`. No other feature writes any of this.

## Invariants
1. **The player pays UAH and the wallet stays UAH.** Every amount sent to Stripe
   is kopiykas derived from the `DECIMAL(12,2)` string — never a float.
2. **The Stripe webhook is the only place a top-up reaches `completed`**, only
   from `pending`, only after amount and currency match the row; a re-delivered
   event settles nothing twice.
3. **A dispute or a refund changes nothing.** The webhook acknowledges those
   events and ignores them; a non-2xx leaves the handler only for a bad signature.
4. **LiqPay-era ledger rows keep rendering** in the player's History and in the
   admin views. Nothing disappears.
5. **The SPAs never see a Stripe secret** — the server exposes only `configured`
   and `mode`.

## Interfaces
- `POST /pc/v1/wallet/topup` — same request; the response carries `checkout_url`
  instead of the LiqPay envelope ("touches shared surface").
- `POST /pc/v1/payments/stripe/webhook` — public; `Stripe-Signature` is the credential.
- `GET /pc/v1/admin/topups` — the read-only list; `GET /pc/v1/admin/stripe/status`
  — `{ configured, mode }` for Settings.

## UI
- **Screens:** Replenishment balance (player, changed — hands off to Stripe's
  page); Account (player, unchanged — the `success` / `cancel` banner it already
  has); Settings (admin, changed — the LiqPay section becomes Stripe status);
  **Top-ups** (admin, new — `/topups`, built from Withdrawals). No design files:
  `design/` stays empty (`DECISIONS.md` 2026-09-17).
- **Reuses:** the Withdrawals view's filter tabs and table, existing tokens.
- **Introduces:** —

## Roadmap
- Sprint 1 — a player buys coins through Stripe's hosted page and LiqPay is gone (`sprints/SPRINT-1.md`)
- Sprint 2 — the operator sees every top-up in the admin SPA (`sprints/SPRINT-2.md`)
