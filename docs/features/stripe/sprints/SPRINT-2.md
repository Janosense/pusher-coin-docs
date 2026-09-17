# SPRINT 2 — The operator sees every top-up (after Sprint 1 → open)

**Branch:** `stripe/sprint-2` ← `main` (after Sprint 1 merged), task branches `stripe/sprint-2-{short-name}` merged `--no-ff`; the same name in `backend/`, `admin/` and the root docs repository. The player SPA is untouched this sprint.
**Goal:** In the admin SPA the operator opens **Top-ups** and sees every top-up ever made — LiqPay-era rows and Stripe rows side by side — filtered by status, paged, read-only. **Settings** tells the operator whether Stripe is configured and whether the keys are `test` or `live`, without ever showing a key.

## Fixed decisions
- The admin gets a read-only Top-ups list, mirroring `admin/withdrawals` — `DECISIONS.md` 2026-09-17. No approve / refund / block action, whatever the screen makes tempting.
- Only `configured` and `mode` leave the server; no key, no fragment — `DECISIONS.md` 2026-09-17 "Stripe configuration: two wp-config constants, a derived status in Settings".
- No UI design; **Top-ups** is built from the Withdrawals layout — `DECISIONS.md` 2026-09-17 "`stripe` has no UI design".
- Chargebacks and refunds change nothing — `DECISIONS.md` 2026-09-16. A `failed` row is `failed`; there is no `disputed` state to invent.
- Product UI lives in the admin SPA, never in `/wp-admin/` — `docs/TECH-STACK.md` → ANTI-PATTERNS.

## Steps

### [ ] Step 1 — `GET /admin/topups` and `GET /admin/stripe/status`
- **Tasks:**
  - `app/stripe/AdminTopupController.php`, registered by the existing `app/stripe/bootstrap.php`. `GET /pc/v1/admin/topups` behind `Permissions::require_admin`: `?status=pending|completed|failed|all` (default `all` — unlike withdrawals, there is no queue to work), `?page&per_page` (defaults 1 / 50), rows of `type = topup` newest first, read the way `AdminWithdrawalController::list_withdrawals` reads them; items carry `id`, `user_id`, `user_email`, `user_nickname`, `amount_money`, `amount_coins`, `unit_price`, `status`, `external_ref`, `notes`, `created_at`, `settled_at`; `total`, `page`, `per_page` as in withdrawals. Invalid status → `invalid_transaction_status` 400 (reuse the existing code if withdrawals has one).
  - `GET /pc/v1/admin/stripe/status` behind `require_admin`: `{ "configured": bool, "mode": "test" | "live" | null }` from `Stripe_Client::is_configured()` and `mode()`. Nothing else in the body.
  - Register both in `docs/CONTRACTS.md` as current.
- **Tests:** Read-only, no money mutation: a WP-CLI eval script is not required. `backend/bin/check` must pass; a manual `curl` with an admin bearer covers the shapes.
- **Verification (manual):** With an admin bearer, `GET /pc/v1/admin/topups` lists the local top-ups including the seeded LiqPay-era rows (`pc-topup-N` refs) and the Stripe ones (`cs_…`), `?status=failed` narrows correctly, a non-admin bearer gets 401/403. `GET /pc/v1/admin/stripe/status` answers `configured: true, mode: "test"`; with a constant removed, `configured: false, mode: null`.
- **Docs to update:** `docs/CONTRACTS.md` (both endpoints, current); `docs/PROJECT-TREE.md`; `docs/features/stripe/FEATURE.md` → Interfaces if a shape changed.
- **Depends on:** —

### [ ] Step 2 — The Top-ups screen and the Stripe badge in Settings
- **Tasks:**
  - `admin/src/services/adminTopupService.js` (`listTopups({ status, page, perPage })`, `getStripeStatus()`); `admin/src/views/TopupsView.vue` built from `WithdrawalsView.vue`'s filter tabs and table, minus the action column and dialogs — columns: player (nickname + email), amount (UAH, decimal string as-is), coins × unit price, status, reference (`external_ref`), created, settled; empty and loading states as Withdrawals has them.
  - Route `/topups` (name `topups`) in `admin/src/router/index.js` next to `/withdrawals`; a nav entry in `components/AdminLayout.vue`. *Touches shared code:* the router and the layout.
  - `views/SettingsView.vue`: the static Stripe section from Sprint 1 gains a live badge from `getStripeStatus()` — green `Configured · test` / `Configured · live`, red `Not configured`, with the same red treatment the captcha panel uses. *Touches shared code.*
  - `docs/DESIGN.md` → Screens: add the **Top-ups** row (admin, `/topups`, `TopupsView.vue`, states: filter tabs, empty, paged) and update the Settings row's states (`Stripe status`).
- **Tests:** `npm run lint && npm run build` in `admin/` pass; no automated UI tests exist in this project (`docs/TECH-STACK.md`).
- **Verification (manual):** On **Top-ups**, the tabs `All / Pending / Completed / Failed` change the list; a LiqPay-era row and a Stripe row both render with their references; nothing on the screen can be clicked to change a row. On **Settings**, the Stripe badge reads `Configured · test` with the local keys; with a constant removed and the page reloaded it turns red.
- **Docs to update:** `docs/DESIGN.md` → Screens; `docs/ARCHITECTURE.md` → admin routes table; `docs/PROJECT-TREE.md`; `docs/features/stripe/FEATURE.md` → UI.
- **Depends on:** Step 1

## Definition of Done
- [ ] Every step closed via /close-step (report + verification guide + worklog)
- [ ] The check command (`docs/TECH-STACK.md` → Check command) exits 0 on the sprint branch in `backend/`; `admin/` passes `npm run lint && npm run build`
- [ ] Docs match reality (CONTRACTS, DESIGN, ARCHITECTURE, DECISIONS current)
- [ ] Sprint boundary: the sprint's work merged into `main` per the git model and `backend` `main` pushed (the production FTP release — the two new admin endpoints go live); the admin SPA has no deploy target (`CLAUDE.md` → Deploy), so its screen is verified from the local build against production's API
- [ ] Tymofii saw LiqPay-era and Stripe rows together on **Top-ups**, filtered by status, with no action available on any row
- [ ] Tymofii saw the Stripe badge on **Settings** read `test` against the keys in use

## Out of scope
- Any action on a top-up row (refund, block the player, mark disputed) — out of v1; a later feature with its own `DECISIONS.md` entry.
- Recording disputes or refunds — out of v1 (`DECISIONS.md` 2026-09-16).
- A deploy target for the admin SPA — Phase 8 launch readiness, unhoused.
- An operations dashboard aggregating Top-ups with withdrawals and the machine — a future `ops` feature (`realtime` FEATURE.md names it).

## Risks / notes
- The admin SPA is local-only; "verified in production" for this sprint means the local build talking to the production API with an admin account.
- If Sprint 1's release runs on test keys, the badge will read `test` in production by design — that is the warning the decision intended, not a defect.
- `WithdrawalsView.vue` is the template; if its structure changed in a `realtime` sprint by then, copy the current one, not the one discovery saw.
