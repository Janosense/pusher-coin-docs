# Sprint 2 — step plans (`stripe`)

<!-- Written by /plan-step, one section per step. A section's status moves
     awaiting approval → approved, in progress → implemented, awaiting close →
     closed. Never edit another step's section. -->

## Plan — Sprint 2, Step 1: `GET /admin/topups` and `GET /admin/stripe/status`   (status: closed)

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

### Execution notes (for `/close-step`)
- **Commits.** Backend `8bcf2d76` (task 1), `106bdd5e` (task 2). Docs `459fef0`
  (task 1), `e1617a2` (task 2). Branch `stripe/sprint-2-admin-endpoints` ← `stripe/sprint-2`
  ← `main`, created in `backend/` and the root docs repository only.
- **Check command.** `backend/bin/check` exit 0 before each commit and once more at the
  end: `php -l: 49 files OK`, DDEV stage **run**, `39 / 54 / 53` checks passed (146).
- **Local fixture written.** Transaction **#268**: user 50 (`stripe-step3-check`),
  `topup`, `80.00` / 2 coins @ `40.00`, `failed`, `external_ref = pc-topup-268`,
  `notes = local verification fixture`. It was created through `Wallet_Service::record_transaction`
  + `set_external_ref`, moves no coins, and stays for the user's verification.
- **Observed by hand** (admin = user 1, player = user 50, bearers from
  `\PC\AuthController::issue_access_token`):
  - `GET /admin/topups` → 200, `total` 4, order `268, 197, 95, 59`, every item exactly
    the 12 keys, `amount_money` / `unit_price` JSON strings. `pc-topup-268`, `null`
    (#197) and two `cs_test_…` refs.
  - `?status=failed` → 2 (`268`, `197`), `pending` → 1 (`59`), `completed` → 1 (`95`),
    `all` and empty → 4. `?per_page=1&page=2` → `197`, `page=5` → empty items, total 4.
  - `?status=refunded` and `?status=bogus` → 400 `invalid_transaction_status`.
  - No bearer → 401 `rest_forbidden`; player bearer → 403 `rest_forbidden`, on both routes.
  - `GET /admin/stripe/status` → `{"configured":true,"mode":"test"}` byte for byte, no
    `sk_` / `whsec_` in the body.
- **Not observed by the agent:** the unconfigured body (`configured: false, mode: null`).
  It needs a local `wp-config.php` edit, which holds the real test keys, and is left to
  the user's verification as planned.
- **One addition beyond the plan's wording, same file:** task 2 also reworded the
  `docs/PROJECT-TREE.md` comment that task 1 added, so the line names both routes the
  file now holds (core rule 5). No other deviation.
- `docs/features/stripe/FEATURE.md` → Interfaces unchanged: both shapes are as it lists them.

## Plan — Sprint 2, Step 2: The Top-ups screen and the Stripe badge in Settings   (status: implemented, awaiting close)

### Branch
`stripe/sprint-2-topups-screen` ← `stripe/sprint-2`, in **`admin/` and the root
documentation repository**.
- `admin/` has no `stripe/sprint-2` yet (Step 1 did not touch it). `/do-step` creates it
  from `admin`'s `main` (`c40773a`, the Sprint 1 merge) first.
- `backend/` is not touched: both endpoints shipped in Step 1.
- `frontend/` is untouched all sprint.

**Touches shared code — may affect other features.** Three `core`-owned admin files
change:
- `router/index.js` and `components/AdminLayout.vue`: every admin screen renders inside
  the layout and passes the router's guard.
- `views/SettingsView.vue`: it also holds the coin-pricing and bonus-map sections.

`realtime` claims none of them. Its admin touchpoint is a future
`admin/src/services/realtime.js`. The rest is the feature's own:
`views/TopupsView.vue` and `services/adminTopupService.js`, the two admin files
`DECISIONS.md` 2026-09-17 names.

### Tasks (ordered)
Docs are committed with the code they describe (core rule 5). Each task leaves
`npm run lint && npm run build` green in `admin/` on its own. The route and the nav link
land in the same task as the screen, so no link ever points at a missing page.

- [x] **1. The Top-ups screen, its route and its nav link.**
  - **`src/services/adminTopupService.js` (new).** It follows `adminWithdrawalsService`
    and exports `adminTopupService` plus a default export.
    - `listTopups({ status = 'all', page = 1, perPage = 50 } = {})` calls
      `GET /admin/topups` with `{ status, page, per_page }`. It returns
      `{ items, total, page, perPage }`.
    - A `mapTopup` maps each row to camelCase: `id`, `userId`, `userEmail`,
      `userNickname`, `amountMoney`, `amountCoins`, `unitPrice`, `status`,
      `externalRef`, `notes`, `createdAt`, `settledAt`.
    - **`amountMoney` and `unitPrice` stay the API's decimal strings. Never `Number()`**
      (root `CLAUDE.md` invariant 7). Only `amountCoins`, a count, becomes a number.
  - **`src/views/TopupsView.vue` (new).** It is built from `WithdrawalsView.vue`'s header,
    filter tabs, table, status badge, error box and empty message. The action column,
    the dialog and the Pinia store are all dropped.
    - **State:** local `ref`s in the view: `items`, `total`, `page`, `statusFilter`,
      `isLoading`, `error`. This is `ChatView`'s pattern. `DECISIONS.md` 2026-09-17
      names only the view and the service, so no store file is added.
    - **Tabs:** `All / Pending / Completed / Failed`, in that order, rendered by
      `TopupsView`, default `All`.
      - The active tab carries Withdrawals' `.active` style.
      - Clicking a tab sets the filter, resets to page 1 and refetches.
    - **Table columns, rendered by `TopupsView`:**
      - **Player:** nickname, with the email under it in the monospace style.
      - **Amount (UAH):** `₴80.00`, the decimal string as the API sends it.
      - **Coins × price:** `2 × ₴40.00`.
      - **Status:** the pill badge. Withdrawals' `.badge--failed` rule already covers
        `failed`.
      - **Reference:** `external_ref` in monospace and allowed to wrap (a `cs_…` id is
        66 characters), or `—` when there is none.
      - **Created** and **Settled:** Withdrawals' `formatDate`. WordPress runs on UTC
        locally (`gmt_offset` 0), so its `Z` suffix is right.
      - **Notes:** the note, or `—`. See Docs vs reality.
      - **No action column, no dialog, no button in any row.**
    - **Loading, empty and error, as Withdrawals has them:**
      - While a request runs, the rows already shown stay and the empty message is
        suppressed. There is no spinner.
      - `No top-ups match this filter.` shows when a finished request returned nothing.
      - A failed request shows its message in the red error box.
    - **Pager:** `ChatView`'s control, the only paged admin list. `Prev` /
      `Page {n} of {m}` / `Next`, rendered by `TopupsView`.
      - It is rendered only when there is more than one page.
      - `Prev` is disabled on page 1 and `Next` on the last page.
      - A page holds 50 rows, the endpoint's default.
    - Heading `Top-ups`. The styles are copied and scoped, in the SPA's
      duplicate-don't-abstract habit: `topups__…` for Withdrawals' `withdrawals__…`
      classes, plus ChatView's pagination block.
  - **`src/router/index.js`:** `{ path: '/topups', name: 'topups', component: TopupsView,
    meta: { requiresAuth: true } }`, placed directly after `/withdrawals`.
    *Touches shared code (the router, `core`).*
  - **`src/components/AdminLayout.vue`:** `<RouterLink :to="{ name: 'topups' }">Top-ups</RouterLink>`,
    directly after Withdrawals. *Touches shared code (the layout, `core`).*
  - Docs in the same change:
    - `docs/DESIGN.md` → Screens: a row `Top-ups | admin | /topups | TopupsView.vue |
      filter tabs, empty, paged` after Withdrawals. Out of scope: "six sections" becomes
      "seven".
    - `docs/ARCHITECTURE.md` → Admin SPA:
      - a `/topups` | `TopupsView` row in the routes table;
      - the nav sentence becomes seven sections, including Top-ups;
      - `adminTopupService` joins the services list;
      - "aggregating the six sections" becomes seven.
    - `docs/PROJECT-TREE.md`: `adminTopupService.js` and `TopupsView.vue` lines.
    - `docs/features/stripe/FEATURE.md` → UI: **Top-ups** is marked done.
  → admin commit `feat(stripe): the Top-ups screen — a read-only list at /topups`;
    docs commit `docs(stripe): the Top-ups screen in DESIGN, ARCHITECTURE, PROJECT-TREE, FEATURE`

- [x] **2. The live Stripe badge in Settings.**
  - **`src/services/adminTopupService.js`:** `getStripeStatus()` calls
    `GET /admin/stripe/status` and returns `{ configured: !!configured, mode: mode || null }`.
    Nothing else exists to map. The endpoint sends only these two fields.
  - **`src/views/SettingsView.vue`:** the existing Stripe section gains a status line
    **directly under its `<h3>Stripe</h3>`**, above the two explanatory paragraphs,
    rendered by `SettingsView`. `hydrateStripeStatus()` runs in `onMounted` next to the
    other two loaders. *Touches shared code (`SettingsView`, `core`).*
    - `configured: true` → **green** `Configured · test` or `Configured · live`.
      - A `configured: true, mode: null` answer (a key with an unrecognised prefix,
        which CONTRACTS allows) shows `Configured` alone. That comes from the same
        expression, not a separate state.
    - `configured: false` → **red** `Not configured`.
    - **The shape** is the captcha panel's status line: a tinted box with 10px 12px
      padding and a 6px radius, as `SubjectsView`'s `.captcha__state` has it.
    - **The colours** are the SPA's own `--success` / `--danger` tints, the same values
      SettingsView's success and error boxes already use. See Docs vs reality: the
      captcha panel's "off" state is amber, not red.
    - **A failed request** shows its message in the section's existing `.settings__error`
      box, the way the pricing and bonus sections report a failed load. It never shows
      `Not configured`, which would be untrue.
    - **While loading**, no status line is shown. The other sections show no loading
      text either.
    - The two paragraphs and the webhook URL below the status line stay as Sprint 1
      wrote them.
  - Docs in the same change:
    - `docs/DESIGN.md` → Screens, Settings row: add `Stripe status (configured · test /
      live; **not configured in red**)`.
    - `docs/ARCHITECTURE.md` → the `/settings` row names the live Stripe status.
    - `docs/PROJECT-TREE.md` → the `SettingsView.vue` line names the Stripe status badge.
    - `docs/features/stripe/FEATURE.md` → UI: the Settings badge is marked done.
  → admin commit `feat(stripe): Settings shows whether Stripe is configured, and on which keys`;
    docs commit `docs(stripe): the Stripe status badge in DESIGN, ARCHITECTURE, PROJECT-TREE, FEATURE`

### Files to create/change
- `admin/src/services/adminTopupService.js` — **new** (tasks 1, 2)
- `admin/src/views/TopupsView.vue` — **new** (task 1)
- `admin/src/router/index.js` — one route (task 1, shared)
- `admin/src/components/AdminLayout.vue` — one nav link (task 1, shared)
- `admin/src/views/SettingsView.vue` — status line, loader and two style rules (task 2, shared)
- `docs/DESIGN.md`, `docs/ARCHITECTURE.md`, `docs/PROJECT-TREE.md`,
  `docs/features/stripe/FEATURE.md` (tasks 1, 2)
- `docs/features/stripe/sprints/SPRINT-2-PLAN.md` — status and checkboxes only

### Tests to write
**No automated test.** The step says so: "`npm run lint && npm run build` in `admin/`
pass; no automated UI tests exist in this project".
- The plan rule asking for a test of a post-interaction state cannot be met here without
  adding a test runner (Vitest and Vue Test Utils). That is a new dependency, core rule 1,
  and it is not in the step. The step settles it, so it is not a question.
- The post-interaction states are therefore checked by hand, as below, and in the
  verification guide `/close-step` writes.

What `/do-step` checks by hand, against the local DDEV. Step 1's fixture `pc-topup-268`
is still there:
1. `npm run lint && npm run build` exit 0 in `admin/` before each commit and once more at
   the end.
2. **In a browser, if the Claude-in-Chrome extension is connected.**
   - Setup: `npm run dev` on :5174, with a session for admin user 1 placed in
     `localStorage` from a token minted by `\PC\AuthController::issue_access_token`.
     Signing in by hand needs the emailed 6-digit code.
   - **Top-ups:**
     - The nav shows `Top-ups` after `Withdrawals`, and `/topups` lists 4 rows.
     - `pc-topup-268`, two `cs_test_…` references and one `—` all render.
     - Clicking **Failed** makes that tab the active one and leaves exactly rows 268
       and 197.
     - No row has a button.
     - With 4 rows, the pager is **not** rendered.
   - **Settings:** the Stripe section reads a green `Configured · test`.

   If the extension is not connected, `/do-step` says so, and the screen checks rest on
   the guide.
3. The red `Not configured` needs the local `wp-config.php` edit that holds the real
   test keys. It is left to the user's guide, as in Step 1.

### Docs to update
- `docs/DESIGN.md` → Screens: the Top-ups row and the Settings row's Stripe status (the
  step's list). Out of scope: six sections becomes seven (core rule 5).
- `docs/ARCHITECTURE.md` → Admin SPA: the routes table (the step's list), plus the nav
  count, the services list and the dashboard sentence in the same section (core rule 5).
- `docs/PROJECT-TREE.md` — the two new files and the SettingsView line (the step's list).
- `docs/features/stripe/FEATURE.md` → UI — Top-ups and the Settings badge are done (the
  step's list).
- Not touched: `CONTRACTS.md` (no endpoint changes), `DATA-MODEL.md`, `DECISIONS.md`,
  `DOMAIN.md`, `TECH-STACK.md` (no dependency).

### Checks
- ANTI-PATTERNS: **none violated.**
  - No float: money is shown as the API's decimal strings, and only the coin count is
    numeric.
  - No secret in the SPA: the badge reads a boolean and a mode.
  - No product UI in `/wp-admin/`: the screen is an admin-SPA view against
    `pc/v1/admin/*`.
  - No shared component package: the table styles are copied and scoped, not extracted.
  - No new dependency, no hardcoded operator-tunable value. The 50-row page is a UI
    paging size, like ChatView's 20.
  - No action on a row: `DECISIONS.md` 2026-09-17 and 2026-09-16.
- Docs vs reality: **mismatch — resolved:**
  - **"The same red treatment the captcha panel uses" does not exist.**
    `SubjectsView.vue`'s unconfigured captcha state is **amber** (`#ffd970` on a yellow
    tint), and only the configured state is tinted (green). `DESIGN.md`'s Subjects row
    says "unconfigured in red", so it is inaccurate too.
    - Resolution: `DECISIONS.md` 2026-09-17 says "red when unconfigured", and DECISIONS
      comes first. The badge takes the captcha line's **shape** and the SPA's
      **`--danger` red**.
    - The Subjects row and the captcha code are left as they are (`core`, an `/adhoc`
      candidate).
  - **The Notes column.** The step's column list omits notes. But the same sentence
    says the table is Withdrawals' "minus the action column", and Withdrawals has a
    Notes column. `DECISIONS.md` 2026-09-17 also lists `notes` among what the list
    carries.
    - Resolution: DECISIONS comes first, so **Notes is kept** as the last column. It is
      also how a failed row explains itself, for example "Stripe: stripe_call_failed".
  - **Withdrawals has no pager.** It fetches one page of 100. The sprint goal and the
    step's DESIGN row still name the screen **paged**. Resolution: ChatView's pager, the
    one already shipped in the admin SPA.
  - **Withdrawals keeps its state in a Pinia store**, and DECISIONS names no store file.
    Resolution: local state in the view, as ChatView does.
  - **"Loading states as Withdrawals has them"** means no loading indicator: the empty
    message is only held back while a request runs. That is copied exactly. No spinner
    is invented.
  - `SPRINT-1-CLOSE.md` Contradiction 5 (DESIGN.md has no Top-ups row while FEATURE.md
    lists the screen) is closed by task 1.
- Design: **n/a — no design files by decision** (`DECISIONS.md` 2026-09-17 "`stripe`
  has no UI design"; `docs/features/stripe/design/` is empty). **Top-ups** follows the
  Withdrawals row of DESIGN.md → Screens, and **Settings** keeps its row with a new
  state. `FEATURE.md` → UI: Reuses "the Withdrawals view's filter tabs and table,
  existing tokens"; Introduces "—", and that stays true (no new token, no shared
  component).
- Check command: `admin/` has none. Its gate is `npm run lint && npm run build`
  (`docs/TECH-STACK.md` → Check command). It ran green on `admin` `main` during
  planning: lint clean, `vite build` 118 modules, exit 0. `backend/bin/check` does not
  apply, since no backend file changes.
- Not locally verifiable:
  - **The pager's Prev / Next.** The local database holds 4 top-ups, below one 50-row
    page, so the pager correctly never renders locally. It is exercised when the local
    build is pointed at production's API at the sprint boundary (`SPRINT-2.md` →
    Definition of Done). Production holds the LiqPay-era history, and whether that
    exceeds 50 rows per filter is unknown until then.
  - **The screens against production.** The admin SPA has no deploy target. The
    boundary check runs the local build against production, after `backend` `main` is
    pushed, which is when the two endpoints go live.

### Questions / ambiguities
none

### Execution notes (for `/close-step`)
- **Commits.** Admin `76b66d6` (task 1) and `b3d63f6` (task 2). Docs `6292a81`
  (task 1) and `d16f689` (task 2).
  - Branch `stripe/sprint-2-topups-screen` ← `stripe/sprint-2`, in `admin/` and the root
    docs repository.
  - `admin`'s `stripe/sprint-2` was created from its `main` (`c40773a`) first.
  - `backend/` was untouched and stays on `stripe/sprint-2` with Step 1's endpoints.
- **Gate.** `npm run lint && npm run build` in `admin/` exited 0 before each commit and
  once more at the end: lint clean, 121 modules built.
- **Browser check: partly done, by choice.**
  - The Claude-in-Chrome extension was connected, and the user's own admin dev server
    was already running on :5174 from `admin/`, serving the working tree. It was used
    as it was, never restarted.
  - Observed signed out: `/topups` redirected to `/sign-in?redirect=/topups`. The
    route is registered and behind the auth guard.
  - **Not done:** getting past sign-in would have meant writing an access token into
    the browser's storage to authenticate. The agent does not place credentials or
    tokens in a browser, so the minted token was deleted unused.
  - So the signed-in screen states were **not** observed by the agent: 4 rows, the
    Failed tab leaving 268 and 197, no buttons in rows, the green and red badge. They
    rest on the verification guide.
- **As planned, no deviation in code:**
  - Notes kept as the last column.
  - ChatView's pager, shown only when there is more than one page.
  - Local state in the view, with no store.
  - The badge sits directly under the Stripe heading, in the `--success` / `--danger`
    tints.
  - A failed status request shows the error box, never "Not configured".
- **One CSS detail the plan did not spell out.** The badge and its error line sit
  inside `.settings__section-header`, whose `p` rule sets a muted 12px style. So the
  two new rules are scoped as `.settings__section-header .settings__stripe-…` to win on
  specificity. The error line gets its own `color: var(--danger)` rule for the same
  reason.
- Step 1's fixture top-up **#268 `pc-topup-268`** is still in the local database. The
  guide uses it as the LiqPay-era row.
