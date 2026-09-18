# Sprint 2 — step plans (`stripe`)

<!-- Written by /plan-step, one section per step. A section's status moves
     awaiting approval → approved, in progress → implemented, awaiting close →
     closed. Never edit another step's section. -->

## Plan — Sprint 2, Step 1: `GET /admin/topups` and `GET /admin/stripe/status`   (status: approved, in progress)

### Branch
`stripe/sprint-2-admin-endpoints` ← `stripe/sprint-2` ← `main`, in **`backend/` and the
root documentation repository**. `stripe/sprint-2` does not exist yet in any
repository: `/do-step` creates it from `main` in these two. `admin/` gets its
`stripe/sprint-2` from its own `main` when Step 2 starts. `frontend/` is untouched all
sprint (`SPRINT-2.md` → Branch).

**No shared code is modified.** Every code file is the feature's own (`app/stripe/`).
`Permissions::require_admin`, `Wallet_Service`'s constants and `Stripe_Client` are
*called*, not changed. `wp_pc_transactions` (owned by `core`) is **read** the way
`AdminWithdrawalController::list_withdrawals` reads it. It is never written. The
`core`-owned files this step touches are documentation only: `docs/CONTRACTS.md`,
`docs/ARCHITECTURE.md` and `docs/PROJECT-TREE.md`.

### Tasks (ordered)
Docs are committed with the code they describe (core rule 5), so there is no trailing
documentation task. Each task leaves `backend/bin/check` green on its own.

- [x] **1. `GET /pc/v1/admin/topups`.** Create `app/stripe/AdminTopupController.php`:
  `final class AdminTopupController extends WP_REST_Controller` in namespace `PC`, with
  `namespace = 'pc/v1'` and `rest_base = 'admin/topups'`. `app/stripe/bootstrap.php`
  requires it and registers it on the existing `rest_api_init` hook, next to the
  webhook controller. Its docblock line "The admin top-ups controller joins them in
  Sprint 2" becomes present tense.
  - Route: `WP_REST_Server::READABLE`, `permission_callback => [ Permissions::class, 'require_admin' ]`.
    Args follow withdrawals: `status` string (default `all`), `page` integer (default 1),
    `per_page` integer (default 50).
  - Paging is copied from `list_withdrawals`: `page = max(1, …)`,
    `per_page = min(100, max(1, …))`, `offset = (page − 1) × per_page`.
  - Status filter. The allowed values are `Wallet_Service::STATUS_PENDING`,
    `STATUS_COMPLETED` and `STATUS_FAILED`, plus the literal `all`.
    - An empty value or `all` means no status condition.
    - Any other value, **`refunded` included**, answers
      **`invalid_transaction_status` 400** with the message
      "status must be one of: pending, completed, failed, all.".
    - `refunded` is excluded because only `reject_withdrawal` ever writes it, so no
      top-up can be in that state. The step lists exactly the four values.
    - The shape comes from the admin filter precedents (`AdminSupportController`
      `invalid_ticket_status`, `AdminChatController` `invalid_message_status`).
      Withdrawals has no such code: it silently falls back to `pending`.
  - Query: `WHERE type = %s` (`Wallet_Service::TYPE_TOPUP`), plus `AND status = %s`
    when a status is given, both through `$wpdb->prepare`. A `COUNT(*)` gives `total`.
    Rows are `ORDER BY created_at DESC, id DESC LIMIT %d OFFSET %d`.
    - **The `id DESC` tie-breaker is the one deliberate difference from withdrawals.**
      `created_at` has one-second resolution, so two rows created in the same second
      could swap places between two page requests. The page boundary could then show
      one row twice and skip another. "Newest first" stays true.
  - Item shape (private `serialize_row`). The step lists exactly these 12 keys, in this
    order, and nothing more (no `type`, no `consumed_lots`):
    - `id` and `user_id` are int.
    - `user_email` and `user_nickname` come from `get_userdata`. The nickname is
      `nickname ?: display_name`, as withdrawals does it (WP core's user field, not a
      `pc` meta key). Both are `null` for a deleted user.
    - `amount_money` and `unit_price` are **the decimal strings exactly as `$wpdb`
      returns them**, never cast.
    - `amount_coins` is int.
    - `status` is as stored.
    - `external_ref` is `null` for a row parked `failed` before Stripe answered. The
      local database has one.
    - `notes`, `created_at` and `settled_at` are as stored. `settled_at` is `null`
      unless the row is `completed`.
  - The body is `{ items, total, page, per_page }`, as in withdrawals.
  - Docs in the same change:
    - `docs/CONTRACTS.md` → Endpoints — current: a `GET /pc/v1/admin/topups` section
      placed after the withdrawals block. It covers the query, the response example,
      that rows of any provider render (`pc-topup-N` next to `cs_…`), and that the list
      is read-only.
    - `docs/CONTRACTS.md` → Error code registry: an `invalid_transaction_status` 400
      row (`admin/topups`).
    - `docs/ARCHITECTURE.md` → `pc` theme — REST layer: the controller count becomes
      "19 controllers: 17 in `app/rest-api/` plus the `stripe` feature's
      `StripeWebhookController` and `AdminTopupController`". The **Admin
      (`require_admin`)** route list gains `GET /admin/topups`.
    - `docs/PROJECT-TREE.md`: an `AdminTopupController.php` line under `app/stripe/`.
  → backend commit `feat(stripe): GET /admin/topups — the read-only list of top-ups`;
    docs commit `docs(stripe): GET /admin/topups in CONTRACTS, ARCHITECTURE, PROJECT-TREE`

- [x] **2. `GET /pc/v1/admin/stripe/status`.** A second route in the same controller,
  registered by its own `register_routes()` at the explicit path `/admin/stripe/status`.
  - Why the same file: `DECISIONS.md` 2026-09-17 ("`stripe` code lives in
    `app/stripe/`…") lists the feature's backend files. `AdminTopupController.php` is
    the only admin one, so a fifth file would contradict that list.
  - Route: `READABLE`, `require_admin`, no args.
  - Body: **exactly** `{ "configured": bool, "mode": "test" | "live" | null }`.
    - `configured` is `Stripe_Client::is_configured()`.
    - `mode` is `Stripe_Client::mode()` when configured and `null` otherwise. `mode()`
      returns `''` for "no usable key", and that is also mapped to `null`.
    - **Why `null` whenever unconfigured:** the step's verification expects
      "with *a* constant removed, `configured: false, mode: null`". If `mode` were
      taken from the key alone, removing `PC_STRIPE_WEBHOOK_SECRET` would answer
      `mode: "test"`. A half-configured server takes no payment either
      (`WalletController::topup` answers `stripe_not_configured`), so a mode would
      mean nothing.
    - A key with an unrecognised prefix answers `configured: true, mode: null`.
      CONTRACTS says so.
  - Nothing else goes in the body: no key, no fragment, no prefix, no webhook URL
    (`FEATURE.md` invariant 5, `DECISIONS.md` 2026-09-17 "Stripe configuration…").
  - Docs in the same change:
    - `docs/CONTRACTS.md` → Endpoints — current: a `GET /pc/v1/admin/stripe/status`
      section with the three possible bodies and the rule that nothing else leaves the
      server. It is modelled on `GET /admin/support/captcha`'s `secret_configured`
      wording.
    - `docs/ARCHITECTURE.md` → the **Admin** route list gains `GET /admin/stripe/status`.
  → backend commit `feat(stripe): GET /admin/stripe/status — configured and mode, never a key`;
    docs commit `docs(stripe): GET /admin/stripe/status in CONTRACTS and ARCHITECTURE`

### Files to create/change
- `backend/wp-content/themes/pc/app/stripe/AdminTopupController.php` — **new** (tasks 1, 2)
- `backend/wp-content/themes/pc/app/stripe/bootstrap.php` — one `require_once`, one registration line, docblock tense (task 1)
- `docs/CONTRACTS.md` — two endpoint sections, one registry row (tasks 1, 2)
- `docs/ARCHITECTURE.md` — REST layer: controller count, Admin route list (tasks 1, 2)
- `docs/PROJECT-TREE.md` — one line (task 1)
- `docs/features/stripe/sprints/SPRINT-2-PLAN.md` — status and checkboxes only

### Tests to write
**None.** The step decides it: "Read-only, no money mutation: a WP-CLI eval script is
not required". Both routes write nothing. `Stripe_Client::is_configured()` and `mode()`
already have their own checks in `tests/stripe-client.php` (both prefixes, an unset
key, each constant missing in turn), and this step only maps their answers. No
permission rule is added or narrowed: both routes consume `require_admin` unchanged,
so no existing actor loses an ability.

What `/do-step` runs by hand before handing over, against the local DDEV. The shapes
are covered by `curl`, as the step asks. Bearers are minted the way the Sprint 1
guides do it (`\PC\AuthController::issue_access_token`) for the local admin (user 1)
and a local player:
1. **A LiqPay-era row to look at.** The local database has none (see Docs vs reality).
   One fixture is created through `Wallet_Service` itself:
   - `record_transaction(…, TYPE_TOPUP, …, STATUS_FAILED, …)` for the player who owns
     the existing top-ups, then `set_external_ref( $id, "pc-topup-$id" )`, with the note
     `local verification fixture`.
   - It is `failed`, so it claims no money the wallet never received.
   - It is the one row `/do-step` writes, and it stays for the user's own verification.
2. `GET /admin/topups` with the admin bearer answers 200. `total` is 4, newest first,
   and every item carries exactly the 12 keys. The `pc-topup-N` row and the two `cs_…`
   rows are all present, and money fields are quoted strings.
3. Each status gives the expected count:
   - `?status=failed` → 2 (the fixture and the null-ref Stripe row)
   - `?status=pending` → 1
   - `?status=completed` → 1
   - `?status=all` and `?status=` → 4
   - `?status=refunded` and `?status=bogus` → 400 `invalid_transaction_status`
   - `?per_page=1&page=2` → the second-newest row
4. No bearer → 401 `rest_forbidden`. A player bearer → 403 `rest_forbidden`, on both
   routes.
5. `GET /admin/stripe/status` with the admin bearer → `{"configured":true,"mode":"test"}`
   byte for byte. The body contains neither `sk_` nor `whsec_`.
6. `backend/bin/check` exits 0 with the DDEV stage **run**, not skipped.

The "constant removed" case (`configured: false, mode: null`) is left to the user's
manual verification. It needs a hand edit of the local `wp-config.php`, which holds the
real test keys. That file is git-ignored, so the edit can never reach a commit.

### Docs to update
- `docs/CONTRACTS.md` — both endpoints as **current**; `invalid_transaction_status` in
  the registry (the step's list)
- `docs/PROJECT-TREE.md` — `AdminTopupController.php` (the step's list)
- `docs/ARCHITECTURE.md` → `pc` theme — REST layer — not in the step's list. It is
  added by core rule 5 (see Docs vs reality).
- `docs/features/stripe/FEATURE.md` → Interfaces — **no change expected**. It already
  reads `GET /pc/v1/admin/topups` (the read-only list) and `GET /pc/v1/admin/stripe/status`
  → `{ configured, mode }`. It is edited only if `/do-step` has to change a shape.
- Not touched: `DATA-MODEL.md` (no table, column, option or meta key), `DOMAIN.md`,
  `DECISIONS.md`, `DESIGN.md` (no screen until Step 2), `TECH-STACK.md`.

### Checks
- ANTI-PATTERNS: **none violated.**
  - No money column is written: both routes are `SELECT`-only, exactly as
    `list_withdrawals` reads (the rule forbids writes outside `Wallet_Service`).
  - No float: `amount_money` and `unit_price` pass through as `$wpdb`'s decimal
    strings.
  - Both routes declare `require_admin`; no new inline gate is invented.
  - No meta-key literal: `nickname` is WP core's user field, read as withdrawals reads it.
  - No secret leaves the server: the status body is a boolean and a mode.
  - No `ENUM`, no transaction, no status change, no cron, no `/wp-admin/` screen, no
    plugin, no new option or constant.
  - The paging defaults (50, capped at 100) are API conventions copied from
    withdrawals, not operator-tunable business values.
  - `LIMIT/OFFSET` is forbidden only for chat's cursor reads; the ledger admin list
    follows withdrawals.
- Docs vs reality: **mismatch — resolved:**
  - **The local database has no LiqPay-era top-up.** The step's verification expects
    "the seeded LiqPay-era rows (`pc-topup-N` refs)". Local `wp_pc_transactions` holds
    exactly three top-ups: #95 `completed` `cs_test_…`, #59 `pending` `cs_test_…`, and
    #197 `failed` with `external_ref` NULL. No seeder exists either (`wp pc` has only
    `seed-rooms` and `machine-ingest`). Resolution: one `failed` fixture row is created
    through `Wallet_Service` (Tests to write, item 1), and the verification guide says
    where it came from. No code task: the step names no seeder.
  - **Withdrawals has no invalid-status code**, so `invalid_transaction_status` is new
    and is registered. It follows the `invalid_ticket_status` / `invalid_message_status`
    precedent. Withdrawals' silent fallback to `pending` is left as it is: `core` is
    frozen, and it is an `/adhoc` candidate if ever wanted.
  - **`Stripe_Client::mode()` returns `''`, not `null`**, and would report `test`
    with only the webhook secret missing. The mapping in task 2 settles both against the
    step's verification text.
  - **`docs/ARCHITECTURE.md` is not in the step's docs list**, but its REST layer
    counts the controllers ("18 controllers") and lists every `require_admin` route.
    Core rule 5 puts both in this change. Step 2's ARCHITECTURE item is the admin SPA
    route table, a different section.
  - `docs/TECH-STACK.md` → Check command lists only `wallet-rollback.php` among the
    eval scripts, while `stripe-client.php` and `stripe-webhook.php` also run. This
    drift predates this step and is left for `/adhoc`.
- Design: n/a. There is no screen in this step; **Top-ups** and the Settings badge are
  Step 2.
- Check command: `backend/bin/check` (`docs/TECH-STACK.md` → Check command). It exists
  and ran green on `main` during planning: `php -l: 48 files OK`, DDEV checks passed
  (`stripe-client.php`, `stripe-webhook.php`, `wallet-rollback.php`), exit 0.
- Not locally verifiable:
  - **Both endpoints in production.** They go live only with the sprint-boundary push
    of `backend` `main` (the FTP release, `SPRINT-2.md` → Definition of Done), which is
    that run's to verify. `SPRINT-1-CLOSE.md` recorded production answering HTTP 500
    and `backend` `main` 17 commits ahead of `origin/main` at close time. The boundary
    re-checks the deploy before trusting it.
  - **`mode: "live"`.** No live key exists anywhere. The `sk_live_` prefix branch is
    covered by `tests/stripe-client.php`, and this endpoint only passes the answer on.

### Questions / ambiguities
none
