# Feature — core

<!-- Lightweight ARCHITECTURE + DATA-MODEL for one feature. Adoption mode: `core`
     is the frozen as-is record of everything that existed when the playbook was
     adopted. It has no sprints and never gets new work — every new piece of work
     is a new feature via Feature mode. -->

## Purpose & scope

`core` is the whole product as it stood on 2026-09-15, registered as one feature so
the playbook has something to resolve paths against: authentication and session
hardening, the player account and its verification gates, rooms with weekly
schedules and a live broadcast, the wallet with FIFO coin lots and LiqPay top-ups,
manual withdrawals, the Home Assistant machine integration, the queue and the toss,
in-room chat with moderation, the support form and ticket triage, and every operator
surface in the admin SPA.

**Out of scope for `core`:** all future work. `core` is a description, not a backlog.
New features get their own `docs/features/{name}/` folder, their own sprints and
their own row in the Features table; the outstanding items listed in `docs/ROADMAP.md`
(the machine-event transport, ops alerts, the admin deploy target, i18n, the security
review, compliance, observability) are candidates for those features, not for `core`.

## Fit into the host
- **Code location:** `backend/wp-content/themes/pc/`, `frontend/src/`, `admin/src/`
- **Host area:** the repository root — the project's single code area, governed by the root `CLAUDE.md`
- **Entry point:** `backend/wp-content/themes/pc/functions.php` (composer autoload → `app/utils.php` → `app/rest-api.php`, plus the `jwt_auth_expire` filter); `frontend/src/main.js`; `admin/src/main.js`
- **Shared code it depends on:** everything under `app/utils/` is shared by every controller; `frontend/src/services/api.js` and `admin/src/services/api.js` are the shared HTTP clients; `frontend/src/assets/main.css` and `admin/src/assets/main.css` are the shared style roots. A change to any of them affects the whole app, so it is a plan task marked **"touches shared code"**.

## Data

`core` owns every persistence surface the project has. Schema details are in
`docs/DATA-MODEL.md`; what belongs to this feature:

- **Custom tables:** `wp_pc_refresh_tokens`, `wp_pc_auth_audit_log`,
  `wp_pc_room_schedules`, `wp_pc_wallets`, `wp_pc_coin_lots`, `wp_pc_transactions`,
  `wp_pc_machine_events`, `wp_pc_bet_sessions`, `wp_pc_room_queues`,
  `wp_pc_room_messages`, `wp_pc_support_tickets`.
- **CPTs:** `pc_room` (+ its `Post_Meta_Keys` meta), `pc_support_subject`.
- **User meta:** every constant in `User_Meta_Keys`.
- **WP options:** every `pc_*` option, including `pc_db_version`.
- **Transients:** the `pc_rl_*` rate-limiter namespace.
- **Browser storage:** `pusher_coin_auth_token`, `pusher_coin_user_data`,
  `pc_theme_song_enabled` (player SPA); `pc_admin_auth_token`, `pc_admin_user_data`
  (admin SPA).

A later feature that needs any of this reads it through the owning service and never
writes it directly. Anything new it stores is its own.

## Invariants

The project-wide ones are the numbered list in the root `CLAUDE.md` → Domain
invariants, and the data-level ones are `docs/DATA-MODEL.md` → Invariants. Both
apply here in full. The three that most often catch a newcomer:

1. `Wallet_Service` is the only writer of the three money tables, always under
   `SELECT … FOR UPDATE`.
2. `Machine_Service` is the only caller of Home Assistant, and its failures map to
   502 / 503 — never 401, or the SPA's refresh interceptor signs the player out.
3. At most one open bet session per room (`ended_at IS NULL`); that is what makes a
   machine payout attributable to a player.

## Interfaces

What `core` exposes for anything else to build on:

- **The `pc/v1` REST namespace** — 18 controllers, catalogued with request /
  response / error shapes in `docs/CONTRACTS.md`. Changing a shape there is
  "touches shared surface".
  *(`stripe` Sprint 1 Step 5: the top-up provider path has moved to that feature —
  `PaymentController` and its LiqPay callback are deleted, leaving 17 controllers
  in `app/rest-api/` plus the feature's own webhook controller.)*
- **Permission callbacks** — `Permissions::require_logged_in`, `require_chat_ready`,
  `require_play_ready`, `require_admin`, and `UserController::check_permission`. A new
  feature reuses these rather than writing its own gate.
- **WordPress hooks** — the filter `pc_machine_event_player` (machine event → player,
  hooked today by `Queue_Service::resolve_player_for_machine`) and the action
  `pc_machine_event_credited` (a credit happened, hooked by
  `Queue_Service::record_win`). These two are the seam a machine-event transport
  plugs into.
- **Services** — `Wallet_Service`, `Queue_Service`, `Chat_Service`, `Machine_Service`,
  `Machine_Ingest_Service`, `Support_Service`, `Refresh_Tokens`, `Rate_Limiter`,
  `Audit_Log`, `Captcha_Verifier`, `Install_Schema`, `Room_Schedule_Calculator`.
- **WP-CLI** — `wp pc seed-rooms`, `wp pc machine-ingest`.

## UI

- **Screens:** all 21, listed once in `docs/DESIGN.md` → Screens (11 player SPA
  views, 10 admin SPA views). There are no design files — `design/` is empty, and
  each screen's Design ref is the `.vue` view that defines it.
- **Reuses:** the component and token inventory in `docs/DESIGN.md` → Components /
  Tokens. Two independent systems: gold-on-navy Oswald for the player SPA, neutral
  dark with a blue accent for the admin SPA.
- **Introduces:** — (`core` *is* the existing design system; a new feature that adds
  a token or a shared component updates `docs/DESIGN.md`).

## Roadmap

`core` has **no sprints** and no `sprints/` folder: it was written from an audit of
working code, not planned. `/plan-step` never targets it. Remaining and future work
lands in new features — see `docs/ROADMAP.md` for the outstanding items and
`docs/DECISIONS.md` for what is already settled.
