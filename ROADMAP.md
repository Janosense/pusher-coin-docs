# Pusher Coin — Roadmap

This roadmap turns the project to-do list into a sequenced plan. It is organised
into phases that build on each other: each phase delivers something the next one
relies on, so we can ship and validate slices instead of one big release.

Status legend used inline:
- `[done]` — already implemented in the codebase.
- `[partial]` — scaffolding exists, behaviour is incomplete.
- `[todo]` — not started.

References to the source to-do list are given as `[#N]` (e.g. `[#5]` = item 5
on the list provided).

---

## Phase 0 — Foundations & inventory (pre-work)

Before adding new features, lock down the contract between frontend and
backend so later phases don't churn it.

- **Audit current state** of the SPA, the `pc/v1` REST namespace, and the
  `player` role. Capture what is `[done]` vs `[partial]`.
- **Define the canonical API contract** (request/response shapes, error
  envelopes, pagination) for everything the UI will need: auth, account,
  rooms, balance, transactions, support, admin. One source of truth before
  controllers proliferate.
- **Establish data model** for the new entities introduced by later phases:
  `room`, `room_schedule`, `wallet`, `transaction`, `bet/session`,
  `support_ticket`, `support_subject`, machine settings.
- **Set up admin-panel approach** — decide whether to use the WordPress admin
  with custom screens (ACF / custom post types / settings pages) or a separate
  admin SPA. Recommendation: extend WP admin to keep one source of truth.
- **Wire CI** for the frontend (lint + build) and a simple PHP lint for the
  theme; both repos already deploy on push but neither is gated.

Exit criteria: documented API contract, a list of new database tables / CPTs,
and a green CI pipeline on both repos.

---

## Phase 1 — Authentication & session hardening `[#1, #2]` — DONE

Finish the auth surface so every later feature can rely on a verified, uniquely
identified player.

1. **Email/password login** `[done]` — JWT issued after email verification.
   Phase 1 added rate-limiting (5 / 15min per IP for code requests; 10 / 24h
   per IP for sign-up), an audit table, and the refresh-token rotation.
2. **Google sign-in** `[done]` — end-to-end flow lands the user on
   `/choose-nickname` on first login; the email-code 2FA is mandatory.
3. **Apple sign-in** `[partial]` — `AppleAuthController` and
   `AppleSignInButton.vue` ship; gated on `APPLE_CLIENT_ID` /
   `APPLE_TEAM_ID` / `APPLE_KEY_ID` / `APPLE_PRIVATE_KEY`. Returns
   `apple_not_configured` until Apple Developer enrollment is complete;
   the SPA hides the button when `VITE_APPLE_CLIENT_ID` is empty.
4. **Terms-and-conditions checkbox** `[done]` — required at sign-up and via
   the blocking `/accept-terms` view on first social login. `terms_accepted_at`
   and `terms_accepted_version` user meta plus the `pc_terms_current_version`
   option drive the gate.
5. **Unique nickname** `[done]` — first-time social-login users land on a
   blocking `/choose-nickname` view (auto-suggesting `User-<6 hex>`,
   server-side uniqueness check). `nickname_chosen` user meta marks
   completion.
6. **Logout with confirmation** `[done]` — `LogoutConfirmModal.vue` opens
   from the navigation; `/auth/logout` revokes the refresh token in
   `wp_pc_refresh_tokens`.
7. **Inactivity auto-logout** `[done]` — `services/sessionService.js` runs a
   15-minute idle timer (mouse / keyboard / click / focus / visibility);
   on timeout it calls `authStore.logout(true)`. Access-token TTL is now
   15 minutes (`pc_access_token_ttl_seconds`); 401s trigger a single
   transparent refresh in `services/api.js`.

Exit criteria: a guest can sign up via email, Google, or (stub) Apple;
cannot reach play / top-up screens without 2FA + accepted terms + unique
nickname; can log out explicitly with confirmation and is logged out
automatically after 15 minutes of inactivity.

---

## Phase 2 — Player account & verification gates `[#3, #14]` — DONE

Build the account page and the verification rules that gate play and top-ups.

1. **Account page** `[done]` — `AccountView.vue` rewritten against
   `/user/me`. Ships:
   - Faceless avatar (`FacelessAvatar.vue`, deterministic 5×5 SVG mosaic).
   - Click-to-edit nickname (server-side uniqueness via Phase 1
     `/user/set-nickname`).
   - Balance + "Add balance" button (disabled until email-verified;
     destination wired in Phase 4).
   - Withdrawal button (same gate; flow in Phase 4).
2. **Email verification** `[done]` — link-based confirmation with 24-hour TTL.
   `request-email-confirmation` mails a `pc_spa_base_url`-rooted link;
   `confirm-email` redeems it. `ConfirmEmailView.vue` handles the
   redemption and offers "Send a new link" on failure.
3. **Google 2FA status** `[done]` — static info panel + outbound link to
   `https://myaccount.google.com/security`. We can't query Google for
   the user's 2FA state; the panel sets the requirement and points the
   user at the right settings page.
4. **Password change** `[done]` — inline two-step on the Account page.
   `request-password-change` mails a 6-digit code;
   `confirm-password-change` validates current password + code, sets the
   new password, revokes every other refresh token for the user, and
   returns a freshly-issued auth envelope.
5. **Verification gates** `[#14]` `[done]` — `Permissions::require_play_ready`
   now also enforces `email_verified_at > 0`. The router redirects
   `meta.requiresPlayReady` routes to `/account?reason=verify-email` for
   unverified users; the page scrolls to a red banner that explains how
   to fix it. Phone verification is intentionally deferred — the field
   carries a neutral "Not yet supported" badge and is not part of the
   gate this phase.

Exit criteria: an unverified user sees an account page that explicitly tells
them what to fix and cannot play or top up until they fix it.

---

## Phase 3 — Rooms, schedules & live broadcast `[#4, #9, #10]` — DONE

Make the room a real domain object with a schedule and a live stream.

1. **Room CPT / table** `[done]` — `pc_room` CPT and the four post-meta
   keys (`pc_room_status`, `pc_room_theme_song_url`, `pc_room_stream_url`,
   `pc_room_machine_id`) registered in `app/utils/cpt-room.php` /
   `Post_Meta_Keys`. Public reads via `GET /pc/v1/rooms*`; admin CRUD
   via `GET/POST/PUT/DELETE /pc/v1/admin/rooms*` (`AdminRoomController`).
2. **Schedule model** `[done]` — `wp_pc_room_schedules` installed
   (Install_Schema 1.2.0); `Room_Schedule_Calculator` computes
   `current_window` / `next_window` from `always` / `once` rules.
   Public read via `GET /pc/v1/rooms/{id}/schedule`; atomic admin
   replace via `PUT /pc/v1/admin/rooms/{id}/schedule`.
3. **Guest browsing** `[#9]` `[done]` — `Rooms.vue` now consumes
   `useRoomsStore()` (`/rooms` list, 30s cache), renders
   `RoomStatusBadge` + `NextBroadcastCountdown` per tile, and shows
   loading / error / empty states. `RoomView` is public and renders
   `LiveStream` + read-only `Chat` for guests.
4. **Live streaming** `[#10]` `[done]` — Mux LL-HLS picked. `.m3u8`
   URLs route through hls.js (dynamically imported — own 162KB gzipped
   chunk, only loaded when an HLS stream actually plays). Safari uses
   native HLS and skips the library entirely. Iframe embeds still
   honoured for one-off event streams.
5. **Guest "Play" CTA** `[done]` — `RoomView` shows a "Sign in to play"
   button for guests; clicking either it or the read-only chat opens the
   in-page sign-in overlay seeded with a redirect back to the same room.
6. **Admin scheduling UI** `[done]` — separate Vue 3 admin SPA at
   `admin/` (Phase 0 picked the standalone-SPA route — see
   `ADMIN-DECISION.md`). Sign-in reuses the player 2FA flow gated by
   `GET /admin/me`. Views: `RoomListView` (table + trash), `RoomFormView`
   (create/edit), `RoomScheduleView` (weekly rules with atomic save).
   Streaming-provider client library still pending (sub-item 4).

Exit criteria: an unauthenticated user can open any room, see a real
broadcast (or a countdown if it isn't live), and is invited to register when
they try to interact.

---

## Phase 4 — Coins, wallet & transactions `[#8, #11, #12, #13, #15, #16]`

The economic core. Don't ship anything beyond this without Phase 1 + 2 done.

1. **Wallet model** `[todo]` — `wallet { user_id, balance_money, balance_coins }`.
   Coins and money are tracked separately because coin price can vary.
2. **Coin pricing rules** `[#8]` `[todo]`:
   - Admin sets default price (initial `$1`), upper and lower bounds.
   - Player can override per-purchase within bounds.
   - Coins purchased *carry their price*: a winning coin pays back at the
     same price it was bought at. Implies coins are stored as a stack of
     `{ qty, unit_price }` lots, or each lot is its own row.
3. **Top-up flow** `[#3, #11]` `[partial]` — `ReplenishmentBalance.vue`
   exists; integrate a real payment provider (Stripe / local processor),
   record a `transaction` row, settle into the wallet.
4. **Withdrawal flow** `[#3]` `[todo]` — request → admin approval (or
   automatic if KYC permits) → payout → transaction row.
5. **Play-button balance rules** `[done]` — `RoomView.onPlayClick`
   routes a zero-balance Play directly to the top-up overlay. `PlaceBet`
   uses `useWalletStore`: preset coin quantity = 1, `+` button caps at
   `balance_coins` and disables on reach, manual input is numeric-only
   and clamped on input + blur. The submit button stays a Phase 6
   placeholder (the toss flow is Phase 6).
6. **Balance display** `[#13]` `[done]` — `UserControls.vue` reads
   `balance_coins` from `useWalletStore` (replacing the hardcoded
   `1000`) and the player nickname from `useAuthenticationStore`
   (replacing `Player 17`). The "Winning" counter is hidden until
   Phase 6 wires it (per ROADMAP §6.1: "winnings hidden when zero").
7. **Transaction history** `[#15, #16]` `[todo]` — `HistoryView.vue` exists;
   list top-ups and withdrawals only. Game results stay out of this view.
   Server-side pagination + filtering by type / date.

Exit criteria: players can top up, the play button respects balance rules,
and transactions are auditable.

---

## Phase 5 — Physical machine integration `[#6]`

Wrap the Home Assistant API documented in `PUSHER-COIN-COMMANDS.txt` in a
thin backend service so the rest of the app never talks to it directly.

1. **`MachineService` (PHP)** `[todo]` — single class that owns the bearer
   token (stored as a WP option or environment variable, not in the repo)
   and exposes:
   - `power_on()` / `power_off()` → `switch.sonoff_10024fb618`.
   - `toss_coin()` → `input_button.toss_a_coin`, expects HTTP 200.
   - `coins_dropped()` → `sensor.coin` state.
   - `bonus_number()` → `sensor.lc01_12` state (1–12 mapped to N coins via
     admin config).
   - `light_state()` → `sensor.light_b_t` (bitwise 0–3).
   - `relay_closed()` → `sensor.relay_on` state.
   - `relay_close()` / `relay_open()` → relay control buttons.
2. **Admin power switch** `[todo]` — UI for `power_on` / `power_off`.
3. **Bonus mapping** `[todo]` — admin panel sets coins-per-bonus-id (1..12)
   and coins-per-relay-closure. Backend reads sensors and credits the
   player's wallet automatically using these mappings.
4. **Coin-throw acknowledgement** `[todo]` — every toss must verify a 200
   response; on failure, do not deduct the coin.
5. **Relay-closed lock** `[todo]` — when `sensor.relay_on` is closed, the SPA
   must disable the toss button (real-time push, not polling).
6. **Documentation walk-through with Dima** `[todo]` — open question to
   clarify state transitions and edge cases (machine offline, sensor
   debounce, coins-from-bonus vs coins-from-relay overlap).
7. **Machine-event channel** `[todo]` — websocket / SSE feed from the backend
   so the SPA reflects coin drops, bonus events, and relay state without
   polling. Likely Pusher / Ably / a self-hosted Soketi.

Exit criteria: the backend mediates every machine call, players see
real-time machine events, and bonuses settle automatically.

---

## Phase 6 — Game session, queue & in-room UX `[#5, #7]`

With wallet + machine + streaming in place, build the actual game loop.

1. **Player main screen** `[#5]` `[partial]` — `RoomView`, `Chat`, `Queue`,
   `PlaceBet` exist. Wire them to the real-time feed:
   - Live broadcast embed (Phase 3).
   - Queue side panel (item below).
   - Chat: guests read-only, players can send. Sending while a guest sends
     them to sign-in.
   - Balance + current-game winnings (winnings hidden when zero — show
     empty, never `0`).
   - Theme-song toggle per room (audio source from admin panel).
2. **Queue mechanics** `[#7]` `[todo]`:
   - "Online players" counter = users actually in queue, not just watching.
   - First player in queue highlighted with nickname + coins they'll play.
   - Self-highlight in queue.
   - When `me === currentTurn`, switch to purple + audio cue.
3. **Coin selector & play action** `[todo]` — connect Phase 4 rules to the
   `toss_coin` machine call; deduct one coin atomically per successful toss.
4. **Win settlement** `[todo]` — listen for `coins_dropped` and bonus events
   from Phase 5; credit the wallet at the same `unit_price` the player
   bought the coin at.

Exit criteria: a verified player can join a queue, take their turn, throw
coins, and see wins settle to their balance — all in real time.

---

## Phase 7 — Support & operations `[#17]`

Customer-facing support form and the admin tooling around it.

1. **Support form** `[partial]` — `SupportView.vue` exists.
   - Guest: email field required + captcha (hCaptcha or Cloudflare Turnstile).
   - Logged-in: email auto-filled, locked, must be verified to send.
   - Subject dropdown driven by an admin-configurable list.
   - Description (required, length-bounded).
2. **Backend ticket store** `[todo]` — REST endpoint, persists to a CPT or
   table, sends an email notification to support, stores the user's IP /
   UA / verified status.
3. **Admin panel surfaces** `[todo]` — manage subjects, list tickets, mark
   resolved, reply by email. Aggregate Phase 5 admin bits (machine power,
   bonus mapping, coin-price bounds, room schedules) into one operations
   dashboard.
4. **Ops alerts** `[todo]` — alert when machine offline, when coin sensor
   doesn't change after a toss, when an unusual number of withdrawals queue
   up.

Exit criteria: support tickets flow into the admin, machine and revenue
events are observable, and the admin can run the business day-to-day.

---

## Phase 8 — Polishing & launch readiness

Cross-cutting items that keep cropping up but don't fit a single phase.

- **i18n** — `LanguageSwitcher.vue` is in the repo; pick a library
  (vue-i18n) and externalise strings.
- **Accessibility & responsive QA** — keyboard / screen-reader pass over
  forms, queue, chat.
- **Performance** — bundle analysis, asset optimisation, stream warm-up.
- **Security review** — JWT rotation, rate limits, captcha coverage,
  CSRF on cookie-bearing endpoints if any get added, log scrubbing for
  secrets, fix the public `check_permission` returning `true` on auth
  routes (require nonces / rate-limits / captcha).
- **Compliance** — terms, privacy, age gating, jurisdiction restrictions.
- **Observability** — request logs, error tracking (Sentry), machine-event
  audit log.
- **Backups & disaster recovery** — DB backups, machine token rotation,
  rollback strategy for the FTP-deployed backend.

---

## Tracking matrix

| # | Item | Phase |
| --- | --- | --- |
| 1 | Registration & auth (email, Google, Apple, T&C, nickname) | 1 |
| 2 | Logout + inactivity timeout | 1 |
| 3 | Player account page | 2 |
| 4 | Guest main screen / room schedule | 3 |
| 5 | Player main screen | 6 |
| 6 | Physical machine integration | 5 |
| 7 | Queue UX | 6 |
| 8 | Coin pricing & wallet | 4 |
| 9 | Guest can browse rooms | 3 |
| 10 | Live streaming | 3 |
| 11 | Play with zero balance → top-up | 4 |
| 12 | Coin-selector input rules | 4 |
| 13 | In-game balance shows total coins | 4 |
| 14 | Verification gates play / top-up | 2 |
| 15 | Player transaction history (data) | 4 |
| 16 | History view (top-ups & withdrawals only) | 4 |
| 17 | Support form + admin subjects | 7 |

---

## Open questions to resolve before / during planning

- ~~Streaming provider and target latency.~~ Resolved Phase 3: **Mux
  LL-HLS** via `hls.js` (lazy-loaded). Safari uses native HLS. Iframe
  embeds (YouTube/Vimeo) still supported for one-off events.
- Payment provider(s) and KYC requirements per jurisdiction.
- ~~Whether the admin panel stays inside WP admin or becomes a separate SPA.~~
  Resolved Phase 0: separate admin SPA (see `ADMIN-DECISION.md`).
- ~~Apple Sign-In: do we need Apple Developer Program enrollment now?~~
  Resolved Phase 1: ship the stub now; flip on when enrollment completes.
  Backend returns `apple_not_configured` until `APPLE_CLIENT_ID` etc.
  are populated.
- Source of truth for the machine bearer token and rotation policy.
- Walk through `PUSHER-COIN-COMMANDS.txt` with Dima to confirm sensor
  semantics and edge cases (relay/bonus ordering, debounce, offline machine).
