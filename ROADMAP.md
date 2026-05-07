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

## Phase 2 — Player account & verification gates `[#3, #14]`

Build the account page and the verification rules that gate play and top-ups.

1. **Account page** `[partial]` — `AccountView.vue` exists; flesh out:
   - Faceless avatar (abstract generated tile per user).
   - Editable, unique nickname.
   - Current balance + "Top up" button (top-up itself in Phase 4).
   - Withdrawal button (withdrawal flow in Phase 4).
2. **Email verification** `[todo]` — confirmation link with 24h TTL; resend
   action; status badge (verified / not). Backend: add a token table or use
   user meta with expiry; controller endpoints `request-email-confirmation`
   and `confirm-email`.
3. **Google 2FA status** `[todo]` — show whether Google 2FA is enabled, and a
   link / instructions to enable it.
4. **Password change** `[todo]` — requires email confirmation + 2FA code
   before applying.
5. **Verification gates** `[#14]` `[todo]` — if email/phone is unverified, show
   a red status dot and **block** play and top-up. Enforce both server-side
   (REST permission callback) and in the SPA UI.

Exit criteria: an unverified user sees an account page that explicitly tells
them what to fix and cannot play or top up until they fix it.

---

## Phase 3 — Rooms, schedules & live broadcast `[#4, #9, #10]`

Make the room a real domain object with a schedule and a live stream.

1. **Room CPT / table** `[todo]` — fields: name, status (`available`,
   `maintenance`, `unavailable`), theme song, stream URL, machine binding.
2. **Schedule model** `[todo]` — weekly recurring rules per room with
   `weekday`, `start_time`, `end_time`, `recurrence` (`always` / `once`).
   Computed "next live window" exposed via the API.
3. **Guest browsing** `[#9]` `[partial]` — `RoomsView` + `RoomView` exist.
   Add: room status badges; countdown timer until next broadcast; "watch
   only" mode when guest is logged out.
4. **Live streaming** `[#10]` `[todo]` — pick a transport (HLS via a CDN,
   WebRTC via a media server, or third-party like Mux/Cloudflare Stream).
   Embed in `RoomView`. Decide on latency target (<2s for a coin pusher feels
   right; HLS is too laggy — favour WebRTC/LL-HLS).
5. **Guest "Play" CTA** `[todo]` — clicking Play in the room as a guest sends
   them to sign-in / sign-up.
6. **Admin scheduling UI** `[todo]` — admin can edit weekly plan from the WP
   admin; Phase 0 decided whether this is WP-native or standalone.

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
5. **Play-button balance rules**:
   - Balance 0 → Play redirects to top-up `[#11]` `[todo]`.
   - Balance ≥ 1 coin → opens a coin selector preset to 1 `[#12]` `[todo]`.
   - `+` button caps at the player's coin balance and disables when reached
     `[#12]` `[todo]`.
   - Manual numeric input clamps to the balance and rejects non-digits
     `[#12]` `[todo]`.
6. **Balance display** `[#13]` `[todo]` — the in-game "Balance" field shows
   total coins owned by the player.
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

- Streaming provider and target latency.
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
