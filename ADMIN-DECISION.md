# Pusher Coin — Admin Surface Decision

**Decision (Phase 0):** every admin surface — room scheduling, bonus mapping,
machine power switch, support ticket triage, ops dashboard — will be a
**separate Vue 3 SPA** (the "admin SPA"), consuming a new
`pc/v1/admin/*` namespace. We will **not** extend the WordPress admin with
custom screens, ACF, or per-feature settings pages.

This diverges from the recommendation in `ROADMAP.md` (which favours
extending WP admin). The roadmap's recommendation stands as guidance, not
as a binding default — Phase 0 is where the decision is made, and the
project owner has chosen the SPA route.

---

## Rationale

- **Consistent stack.** The player SPA is Vue 3 + Vite + Pinia. Admins
  get the same component library, the same router, the same real-time
  channel for machine events. No second context switch between Vue and
  PHP-rendered WP admin pages.
- **Better UX for the operations dashboard.** Phase 7's "single
  operations dashboard" (machine status, withdrawals queue, support
  inbox, alerts) is a real-time, multi-panel view. WP admin is a poor
  fit for that.
- **Cleaner permission story.** All admin endpoints live under
  `pc/v1/admin/*` with a single `permission_callback` that calls
  `current_user_can( 'manage_options' )`. WP admin's nonce + cap +
  screen-hooks model is more surface area for the same outcome.
- **Reuse of the JWT plugin.** The admin SPA authenticates the same way
  the player SPA does — bearer JWT against the existing
  `jwt-authentication-for-wp-rest-api` plugin. No duplicate session
  handling.

## Trade-offs accepted

- **A third deploy target.** Likely Vercel, parallel to the player SPA.
  One extra `vercel.json`, one extra `.env`, one extra CI workflow.
- **Duplicate Vue tooling.** `vite.config.js`, ESLint config, and
  shared component primitives (button, input, modal) will live in two
  Vue projects until a shared package is worth extracting. Phase 0 does
  **not** introduce a monorepo or shared package; we accept some
  duplication and revisit if it grows painful.
- **Admins must use the SPA, not `/wp-admin/`.** `/wp-admin/` remains
  available for emergency database access, plugin installs, and theme
  config — but no product workflow lives there. This is documented in
  the operator runbook (Phase 7 deliverable).

## Implications for later phases

- **Phase 3 (rooms & schedules).** Room scheduling UI is a Vue view in
  the admin SPA. CPT `pc_room` and table `wp_pc_room_schedules`
  (see `DATA-MODEL.md`) are still the storage; the admin SPA writes via
  `PUT /pc/v1/admin/rooms/{id}/schedule`.
- **Phase 5 (machine integration).** Power switch, bonus map editor,
  and machine state view are Vue views in the admin SPA. Backend
  exposes `POST /pc/v1/admin/machine/power`, `GET .../state`,
  `PUT .../bonus-map`.
- **Phase 7 (support & ops).**
  - Support subjects editor: Vue view writing via
    `PUT /pc/v1/admin/support/subjects`. CPT `pc_support_subject` still
    backs the data.
  - Ticket inbox: Vue view, paginated, real-time updates via the same
    channel that pushes machine events.
  - Operations dashboard: Vue view aggregating machine state, recent
    transactions, withdrawal queue, and alerts. **No WP admin
    equivalent.**
- **Permission model.** Every endpoint under `pc/v1/admin/*` declares
  `'permission_callback' => fn() => current_user_can( 'manage_options' )`.
  Player-facing endpoints continue to use `is_user_logged_in()` /
  `__return_true` per the contract.

## What is *not* changing

- The `player` role and the `play` capability (registered in
  `app/utils/role-player.php`) are unchanged. Admin SPA users are
  already `administrator`s.
- `/wp-admin/` itself is still accessible for plugin / theme / DB tasks.
  It is just not the product surface.
- ACF is not being introduced. If a future feature genuinely needs
  flexible content modelling (rich room descriptions, marketing pages),
  revisit then — not preemptively.

## Next steps (not Phase 0)

Phase 0 records the decision and stops. Scaffolding the admin SPA
(directory layout under `admin/`, deploy config, shared component story)
is **not** part of Phase 0. The earliest concrete need is Phase 3's room
scheduling UI; that work scaffolds the admin SPA when it lands.

### Update — Phase 3, Step 5b

The admin SPA is scaffolded at the top-level `admin/` directory: Vite +
Vue 3 + Pinia + Vue Router, mirroring `frontend/`'s tooling. It runs on
port 5174 by default so both SPAs can run side-by-side in dev. Auth
reuses the existing `/user/request-verification` → `/user/verify-code`
2FA flow and then probes `GET /admin/me` to bounce non-admins at
sign-in. Token storage uses distinct localStorage keys
(`pc_admin_auth_token` / `pc_admin_user_data`). Views shipped:
`SignInView` (two-step form) and `HomeView` (placeholder shell — Step
5c populates the room list + schedule editor).

Deploy target: still expected to be a separate Vercel project once it
gets a `vercel.json`; not configured yet because there's no need until
the SPA has product value to deploy.
