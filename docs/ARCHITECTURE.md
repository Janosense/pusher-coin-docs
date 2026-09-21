# Architecture — Pusher Coin

<!-- Describes what exists. Adoption mode fills it from the codebase as-is. -->

## Overview

Pusher Coin is a real-time, browser-based coin-pusher gambling application split
into three independent applications that live side by side in this repository:

- `frontend/` — a Vue 3 single-page application (SPA) that the player interacts with.
- `admin/` — a separate Vue 3 SPA used by operators to manage rooms and schedules,
  review withdrawals, switch and monitor the physical machine, set coin pricing and
  the bonus map, triage support tickets, and moderate chat.
- `backend/` — a WordPress installation that exposes a JSON REST API used by both
  SPAs (the WordPress admin/HTML side is not the user-facing product). It is also
  the only component that talks to the physical machine and to the payment provider.

The three apps are decoupled: separate `.git` repositories (the root repo ignores
all three and tracks only the documentation and the playbook), separate deploy
pipelines, and communication exclusively over HTTPS/JSON.

The whole loop — sign up, verify, top up, queue, toss, settle, withdraw, support —
works end to end. Machine events reach WordPress by themselves, on a schedule that
polls Home Assistant's history, and reach the player's browser over a push channel:
the room's queue and its winnings are pushed, not polled. **Chat is still polled every
3s** until `realtime` Sprint 2 Step 4. Phase status lives in `ROADMAP.md`.

```
┌──────────────────────────┐
│  Player SPA (frontend/)  │ ─┐
└──────────────────────────┘  │   HTTPS / JSON, JWT bearer   ┌─────────────────────────────┐
                              ├─────────────────────────────▶│  WordPress (DDEV / FTP)     │
┌──────────────────────────┐  │ ◀────────────────────────────│  Custom `pc` theme exposes  │
│  Admin SPA (admin/)      │ ─┘                              │  /wp-json/pc/v1 endpoints   │
└──────────────────────────┘                                 └──────────┬──────────────────┘
                                                                        │
                    ┌───────────────────────────┬────────────────────────┼──────────────────────────┐
                    ▼                           ▼                        ▼                          ▼
         /wp-json/jwt-auth/v1        Home Assistant REST         Stripe Checkout            Turnstile / hCaptcha
         (plugin validates            (Machine_Service,           (hosted page out,          siteverify
          incoming bearer JWTs)        PC_MACHINE_TOKEN)           signed webhook in)        (Captcha_Verifier)
```

Both SPAs share the same auth primitives (JWT pair, refresh rotation,
2FA-by-email-code) but live behind distinct localStorage keys (`pusher_coin_*` for
the player, `pc_admin_*` for the admin). The admin SPA additionally probes
`GET /pc/v1/admin/me` after sign-in to bounce non-admins.

## Feature map

| Feature | Purpose | Docs | Code |
|---|---|---|---|
| `core` | Everything shipped before the playbook was adopted: auth and session hardening, player account and verification gates, rooms / schedules / live broadcast, wallet and payments, physical-machine integration, queue and play, in-room chat, support, and every admin surface. | `docs/features/core/FEATURE.md` | `backend/wp-content/themes/pc/`, `frontend/src/`, `admin/src/` |
| `realtime` | The inbound transport that carries machine events from Home Assistant into WordPress, the outbound channel that pushes them to the SPAs, and the attribution rules that decide whose wallet a payout lands in. | `docs/features/realtime/FEATURE.md` | `backend/wp-content/themes/pc/app/realtime/`, `frontend/src/services/realtime.js`, `admin/src/services/realtime.js` |
| `stripe` | Stripe as the only top-up provider — the hosted Checkout hand-off, the settlement webhook, the removal of LiqPay, and the admin's read-only list of top-ups. | `docs/features/stripe/FEATURE.md` | `backend/wp-content/themes/pc/app/stripe/`, `admin/src/views/TopupsView.vue`, `admin/src/services/adminTopupService.js` |

New work never goes into `core` — every new piece of work is a new feature, added
here as a row and as a `docs/features/{name}/` folder.

## Modules

### Player SPA — `frontend/`

**Responsibility.** Everything the player sees: browsing rooms, watching the
broadcast, the account and verification gates, the wallet, the queue, the toss, the
chat, the support form.

**Public surface.** The deployed site (Vercel). It consumes `pc/v1` and nothing else.

**Must never do.** Hold any secret; treat a client-side check as authoritative;
parse or trust the JWT payload (the token is opaque to it).

| Folder | Responsibility |
| --- | --- |
| `main.js` | App entry point. Creates the Vue app, registers Pinia and the router, mounts `#app`. |
| `App.vue` | Root layout: header + `NavigationToggle`, side `AppNavigation`, `<RouterView />`, footer. The header switches to a room variant on the room route. |
| `router/index.js` | Route table and `beforeEach` guard. Routes carry `meta.requiresAuth` / `meta.requiresGuest` / `meta.allowsBeforeGate`; the guard awaits `authStore.initializeAuth()` before deciding. |
| `views/` | Page-level components mapped 1:1 to routes: `RoomsView` (`/`), `RoomView` (`/room/:id`, public), `SignInView`, `SignUpView`, `AccountView`, `HistoryView`, `SupportView` (public), and the three gate / landing views `AcceptTermsView`, `ChooseNicknameView`, `ConfirmEmailView`. `AboutView` exists but is not in the route table. |
| `components/` | Reusable building blocks. Room page: `LiveStream`, `RoomChat` (live, 3s poll, owns its poll lifecycle), `RoomQueue`, `PlaceBet`, `UserControls`, `RoomStatusBadge`, `NextBroadcastCountdown`. Lists / shell: `RoomList`, `AppNavigation`, `NavigationToggle`, `LanguageSwitcher`, `ModalOverlay`, `LogoutConfirmModal`. Account / money: `FacelessAvatar`, `ReplenishmentBalance`, `WithdrawalRequest`. Auth: `SignInForm`, `SignUpForm`, `GoogleSignInButton` (hidden while parked), `AppleSignInButton` (hidden until configured). Plus an `icons/` set of single-purpose SVG components. `HelloWorld.vue` is Vite scaffold with no importers. |
| `stores/` | Pinia stores. `authentication.js` is the central one (token + user, persisted to `localStorage`, with Google 2FA state). `wallet.js` (balance, lots, pricing, top-up), `queue.js` (room queue: subscribes to the room's Ably channel, re-reads on a version change, heartbeats every 20s, falls back to a 3s poll if the channel is unavailable; also holds whether the machine is out of service, from the envelope, the `relay` message and the server's refusal of a toss), `rooms.js` (room list, 30s cache), `navigation.js`, `chat.js` (panel open/closed state *plus* the conversation itself — subscribes to the room's Ably channel, appends a pushed message, drops a hidden one, re-opens the conversation on a restore, catches up through the `after` cursor on every connect, and falls back to the 3s poll for a guest or an unavailable channel), and `themeSong.js` (per-room theme song; owns the `Audio` element because the toggle lives in `UserControls` while the URL arrives with the room in `RoomView`). `counter.js` and `user.js` are unused scaffold. |
| `services/` | API layer. `api.js` is a configured Axios instance with request/response interceptors (auto-attaches the JWT; refreshes once on 401). Endpoint wrappers: `authService`, `accountService`, `userService`, `googleAuthService`, `appleAuthService`, `roomsService`, `queueService`, `chatService`, `walletService`, `historyService`, `supportService`; `sessionService.js` is the inactivity timer. The top-up hand-off needs no service of its own — `ReplenishmentBalance.vue` navigates to the `checkout_url` that `walletService.topup()` returns. |
| `assets/` | Global CSS (`main.css`, `styles/colors.css`, block-scoped CSS in `styles/blocks/`), images, the brand SVG logo. |
| `public/` | Static files served verbatim by Vite (`favicon.ico`). |

Path alias: `@` → `src/` (configured in `vite.config.js` and `jsconfig.json`).

Local storage the player SPA writes: `pusher_coin_auth_token`,
`pusher_coin_user_data` (user blob carrying the refresh token and access-token
expiry) and `pc_theme_song_enabled` (`'1'` / `'0'`, a device-local preference never
sent to the server).

### Admin SPA — `admin/`

**Responsibility.** Every operator surface. Same stack as `frontend/` (Vue 3 + Vite
+ Pinia + Vue Router + Axios + ESLint/Prettier), minus IMask and `hls.js`. Runs on
port 5174 in dev so both SPAs can run side by side.

**Public surface.** None yet — it has no deploy target and is local-only.

**Must never do.** Reach into `/wp-admin/` for product workflows; assume admin
identity from anything but `GET /pc/v1/admin/me`.

`App.vue` is a bare `<RouterView />`; every authenticated view wraps itself in
`components/AdminLayout.vue` (header + nav + slot). The nav has seven sections —
Rooms, Withdrawals, Top-ups, Machine, Support, Chat, Settings:

| Route | View | What it does |
| --- | --- | --- |
| `/sign-in` | `SignInView` | Two-step email/password + 6-digit code form. |
| `/rooms` | `RoomListView` | Table with status badge + per-row Edit / Schedule / Trash. |
| `/rooms/new`, `/rooms/:id/edit` | `RoomFormView` | Shared create / edit form; discriminates on `route.name`. |
| `/rooms/:id/schedule` | `RoomScheduleView` | Weekly rules editor; save is an atomic replace via `PUT /admin/rooms/{id}/schedule`, response carries the recomputed `next_window`. |
| `/withdrawals` | `WithdrawalsView` | Pending-withdrawal queue with filter tabs and an approve / reject dialog (reject asks for a reason). |
| `/topups` | `TopupsView` | Read-only list of every top-up (feature `stripe`): All / Pending / Completed / Failed tabs, paged 50 at a time, LiqPay-era and Stripe references side by side. No action on any row. |
| `/machine` | `MachineView` | Connection probe, power On/Off, sensor grid (coin counter, last bonus, relay, light bitfield). Polls `GET /admin/machine/state` every 3s. |
| `/support/tickets` | `TicketsView` | Ticket queue: status filter, search, expandable message with IP / UA, status transitions, mailto reply. |
| `/support/subjects` | `SubjectsView` | Subject list editor (reorder, hide, replace-all save) plus the guest-captcha provider / site-key panel. |
| `/chat` | `ChatView` | Chat moderation queue: room / status / text filters, hide and restore a message, and a timed account-wide mute. |
| `/settings` | `SettingsView` | Coin price default / min / max, Stripe configuration hint (the two wp-config constants + the webhook URL to register) with a live status badge from `GET /admin/stripe/status` (green `Configured · test` / `live`, red `Not configured`), bonus-map 4×3 grid, relay coin count. |

Stores: `auth.js` (two-step sign-in + `/admin/me` gate), `rooms.js`,
`withdrawals.js`. Services mirror the backend admin controllers one to one:
`adminAuthService`, `adminRoomsService`, `adminWithdrawalsService`,
`adminCoinPricingService`, `adminMachineService`, `adminSupportService`,
`adminChatService`, `adminTopupService`. Token storage: `pc_admin_auth_token`, `pc_admin_user_data`;
`services/api.js` is a copy of the player interceptor with the same refresh-on-401
behaviour (events prefixed `admin-auth:`).

Not built: no `vercel.json` and no CI workflow; no shared component package with
`frontend/` (some duplication is accepted); no inactivity timer; no operations
dashboard aggregating the seven sections.

### `pc` theme — REST layer

**Location.** `backend/wp-content/themes/pc/app/rest-api.php` + `app/rest-api/`.

**Responsibility.** Validate input, declare the permission callback, delegate to a
service, shape the response. Controllers hold no domain logic.

**Public surface.** The `pc/v1` namespace — 19 controllers: 17 in `app/rest-api/`
plus the `stripe` feature's `StripeWebhookController` and `AdminTopupController`,
registered from `app/stripe/bootstrap.php`. The complete catalogue
with request / response / error shapes is `CONTRACTS.md`.

**Must never do.** Talk to Home Assistant, Stripe or the captcha provider directly;
write to a custom table without going through its service; register a route without
an explicit `permission_callback`.

| Controller | Surface |
| --- | --- |
| `UserController.php` | Sign-up, 2FA, `/user/me`, email confirmation, password change |
| `AuthController.php` | `/auth/logout`, `/auth/refresh`, token-pair issuance |
| `GoogleAuthController.php` | Google ID-token exchange + 2FA (backend untouched while the integration is parked) |
| `AppleAuthController.php` | Apple Sign-In (returns `apple_not_configured` until enrolled) |
| `RoomController.php` | Public `/rooms` reads + schedule |
| `RoomQueueController.php` | `/rooms/{id}/queue`, join, leave, play (toss) |
| `RoomChatController.php` | `/rooms/{id}/messages`: public read, gated post |
| `WalletController.php` | `GET /wallet`, `POST /wallet/topup`, `POST /wallet/withdraw` |
| `TransactionsController.php` | `GET /transactions` (paginated, filterable) |
| `SupportController.php` | Public `/support/subjects` + `/support/tickets` |
| `AdminController.php` | `/admin/me` capability probe |
| `AdminRoomController.php` | Admin `/rooms` CRUD + atomic schedule replace |
| `AdminWithdrawalController.php` | `/admin/withdrawals` list, approve, reject |
| `AdminCoinPricingController.php` | `GET`/`PUT /admin/coin-pricing` |
| `AdminMachineController.php` | `/admin/machine` state, power, bonus-map |
| `AdminSupportController.php` | `/admin/support` tickets, subjects, captcha config |
| `AdminChatController.php` | `/admin/chat`: moderation queue, hide/restore, mute |

Routes grouped by the permission callback that gates them:

- **Public, no auth (`__return_true`)** — `POST /google-auth/authentication`,
  `/google-auth/verify-code`, `/apple-auth/authentication`,
  `/apple-auth/verify-code`; `POST /auth/refresh` (the refresh token *is* the
  credential); `POST /user/confirm-email` (the link token is the credential);
  `GET /rooms`, `/rooms/{id}`, `/rooms/{id}/schedule`; `GET /support/subjects`,
  `POST /support/tickets` (rate-limited, captcha-checked for guests when
  configured); `GET /rooms/{id}/messages` (chat is readable by guests, like the
  room page it sits on); `POST /payments/stripe/webhook` (the `Stripe-Signature`
  over the raw body is verified in the handler); `POST /machine/events` (a shared
  secret in `X-PC-Machine-Secret` is verified in the handler, behind a rate limit
  counted across all callers).
- **Public, `UserController::check_permission`** — `POST /user/sign-up`,
  `/user/request-verification`, `/user/verify-code`. These rely on `Rate_Limiter`
  and the email code rather than a capability.
- **Logged in (`Permissions::require_logged_in`)** — `POST /auth/logout`;
  `GET/PATCH /user/me`, `POST /user/set-nickname`, `/user/accept-terms`,
  `/user/request-email-confirmation`, `/user/request-password-change`,
  `/user/confirm-password-change`; `GET /wallet`; `GET /transactions`.
- **Play-ready (`Permissions::require_play_ready` = logged in + terms accepted +
  nickname chosen + email verified)** — `POST /wallet/topup`, `/wallet/withdraw`;
  `GET /rooms/{id}/queue`, `POST /rooms/{id}/queue/join`, `/rooms/{id}/queue/leave`,
  `/rooms/{id}/play`.
- **Chat-ready (`Permissions::require_chat_ready` = logged in + terms accepted +
  nickname chosen, *without* the email-verified step)** — `POST /rooms/{id}/messages`.
  Chat moves no coins, so it is gated one rung below play; a nickname is required
  because every message renders with an author name.
- **Admin (`Permissions::require_admin` = logged in + `manage_options`)** —
  `GET /admin/me`; `GET/POST /admin/rooms`, `GET/PUT/DELETE /admin/rooms/{id}`,
  `PUT /admin/rooms/{id}/schedule`; `GET /admin/withdrawals`,
  `POST /admin/withdrawals/{id}/approve`, `/reject`; `GET /admin/topups` (read-only),
  `GET /admin/stripe/status`; `GET/PUT /admin/coin-pricing`;
  `GET /admin/machine/state`, `POST /admin/machine/power`,
  `GET/PUT /admin/machine/bonus-map`; `GET /admin/support/tickets`,
  `PATCH /admin/support/tickets/{id}`, `GET/PUT /admin/support/subjects`,
  `GET/PUT /admin/support/captcha`; `GET /admin/chat/messages`,
  `PATCH /admin/chat/messages/{id}`, `POST /admin/chat/mute`.

### `pc` theme — domain services

**Location.** `backend/wp-content/themes/pc/app/utils/`, all required in dependency
order by `app/utils.php`, which `functions.php` loads after the composer autoloader.

**Responsibility.** All domain logic and every external call. This is where the
product actually lives — WordPress is an authentication, user-data and integration
service here, not a CMS. `index.php` renders nothing.

```
themes/pc/
├── functions.php            # Bootstrap: composer autoload, utils, feature bootstraps, REST API registration, jwt_auth_expire filter
├── style.css                # Theme metadata header
├── index.php                # Empty/placeholder (no front-end rendering)
├── composer.json            # google/apiclient (Google ID-token verification)
├── GOOGLE_AUTH_SETUP.md     # Operator notes for Google OAuth (parked)
├── CAPTCHA_SETUP.md         # Operator notes for Turnstile / hCaptcha keys + rotation
├── tests/
│   ├── machine-ingest.php   # `ddev wp eval-file` check: the ingest endpoint, idempotency and crediting (DDEV only)
│   ├── machine-poll.php     # `ddev wp eval-file` check: the history poller — arithmetic, replay, gaps, crediting (DDEV only)
│   ├── realtime-relay.php   # `ddev wp eval-file` check: the toss lock, the relay watch and what it may publish (DDEV only)
│   ├── realtime-channel.php # `ddev wp eval-file` check: the push channel — a broken publish cannot touch a payout; the token never carries the key (DDEV only)
│   ├── machine-rooms.php    # `ddev wp eval-file` check: one machine, one available room (DDEV only)
│   ├── realtime-queue.php   # `ddev wp eval-file` check: the queue rides the channel; the heartbeat holds a place and answers a version (DDEV only)
│   ├── realtime-chat.php    # `ddev wp eval-file` check: chat rides the channel; moderation travels as an id and a state (DDEV only)
│   ├── queue-sessions.php   # `ddev wp eval-file` check: one open bet session per room, enforced; the orphan cleanup (DDEV only)
│   ├── realtime-outage.php  # `ddev wp eval-file` check: the outage incident, its grace period and its window gate (DDEV only)
│   ├── stripe-client.php    # `ddev wp eval-file` check: kopiyka conversion, mode / configuration, webhook signature scheme (DDEV only)
│   └── wallet-rollback.php  # `ddev wp eval-file` check: every Wallet_Service write failure rolls back (DDEV only)
└── app/
    ├── rest-api.php         # Wires controllers into `rest_api_init`
    ├── rest-api/            # The 18 controllers listed above
    ├── realtime/            # Feature `realtime` — machine events into WordPress, and the rooms that claim a machine
    │   ├── bootstrap.php    # The feature's single entry point; one require_once in functions.php
    │   ├── machine-rooms.php         # Machine_Rooms: which rooms claim a machine
    │   ├── machine-rooms-command.php # `wp pc machine-rooms` — machine ids held by more than one room
    │   ├── machine-poller.php        # Machine_Poller: polls HA history and delivers each payout to the ingest door
    │   ├── relay-watch.php           # Realtime_Relay_Watch: reads the relay once per pass, announces a change, caches the state
    │   ├── outage-watch.php          # Realtime_Outage_Watch: is the machine answering? Unreachable inside a broadcast window is an incident
    │   ├── toss-watch.php            # Realtime_Toss_Watch: did the machine act on the toss it answered 200 to? The counter's reset is the evidence
    │   ├── alerts.php                # Realtime_Alerts: the one door an operator notification leaves through
    │   ├── machine-poll-command.php  # `wp pc machine-poll` — one pass by hand; `--dry-run` reads without crediting
    │   ├── queue-sessions-command.php # `wp pc queue-sessions` — open bet sessions, duplicates, and whether the unique key is in place
    │   ├── channels.php              # Realtime_Channels: the one place channel names are built
    │   ├── publisher.php             # Realtime_Publisher: pushes credits to Ably, fire-and-forget
    │   ├── RealtimeTokenController.php # GET /realtime/token — the scoped pass a browser gets instead of the key
    │   └── MachineIngestController.php # POST /machine/events — the shared-secret ingest door
    ├── stripe/              # Feature `stripe` — the ONLY code that talks to Stripe
    │   ├── bootstrap.php    # The feature's single entry point; one require_once in functions.php
    │   └── stripe-client.php # Stripe_Client: Checkout Session creation + webhook signature verification
    ├── utils.php
    └── utils/
        ├── role-player.php                 # Registers the `player` role
        ├── user-meta-keys.php              # User_Meta_Keys registry — only way to name a user meta key
        ├── post-meta-keys.php              # Post_Meta_Keys registry — pc_room meta
        ├── permissions.php                 # Permission_callback helpers: logged-in / terms / nickname / email / play-ready / chat-ready / admin
        ├── install-schema.php              # Custom-table installer (dbDelta, pc_db_version) + default options
        ├── audit-log.php                   # Audit_Log writer → wp_pc_auth_audit_log
        ├── rate-limiter.php                # Transient-based per-IP rate limiter (pc_rl_* keys)
        ├── refresh-tokens.php              # Refresh-token issuance / rotation / reuse detection
        ├── cpt-room.php                    # Registers pc_room CPT
        ├── cpt-support-subject.php         # Registers pc_support_subject CPT
        ├── room-schedule-calculator.php    # Computes current_window / next_window from weekly rules
        ├── wallet-service.php              # Atomic wallet / coin-lot / ledger ops (SELECT … FOR UPDATE)
        ├── machine-service.php             # Home Assistant REST wrapper (2s timeout, typed WP_Error)
        ├── machine-events.php              # Machine_Event_Log writer → wp_pc_machine_events
        ├── machine-ingest-service.php      # Machine event → wallet credit; transport-agnostic
        ├── queue-service.php               # Queue, turns, bet sessions, machine-event attribution
        ├── chat-service.php                # Chat storage, posting rules, hide/mute moderation
        ├── captcha-verifier.php            # Turnstile / hCaptcha siteverify
        ├── support-service.php             # Subjects + tickets + notification mail
        └── cli/
            ├── seed-rooms.php              # `wp pc seed-rooms` — demo rooms for local dev
            └── machine-ingest.php          # `wp pc machine-ingest` — replay / test a machine event
```

**Feature directories.** A feature added after the playbook (`app/stripe/`,
`app/realtime/`) owns a directory beside `utils/` and is reached through
exactly one `require_once` of its `bootstrap.php` in `functions.php`. Nothing else in
the theme requires a file from a feature directory directly.

**Must never do.** `Wallet_Service` is the only writer of `wp_pc_wallets`,
`wp_pc_coin_lots` and `wp_pc_transactions`; `Machine_Service` is the only caller of
Home Assistant; `Stripe_Client` is the only caller of Stripe; no controller may bypass
any of them. A meta key is never written as a
string literal — it comes from `User_Meta_Keys` or `Post_Meta_Keys`.

### Plugins

`wp-content/plugins/jwt-authentication-for-wp-rest-api` — provides
`/wp-json/jwt-auth/v1/*` endpoints, validates incoming bearer tokens and sets the
current user, and exposes the `jwt_auth_expire` filter. The theme **issues** the
access JWT itself (`AuthController::issue_access_token`, using the
`Tmeister\Firebase\JWT` copy bundled with the plugin) and hooks only
`jwt_auth_expire`, so a token minted through the plugin's own `/jwt-auth/v1/token`
endpoint gets the same TTL. Everything else under `wp-admin`, `wp-includes` and the
bundled themes is unmodified WordPress core.

## Data flows

**Sign-in and the token pair — FIXED.** Username/password login is two-step:
credentials → 6-digit email verification code → JWT pair. Two JWTs are involved: a
short-lived **access token** (HS256, 15-min default from
`pc_access_token_ttl_seconds`, signed with `JWT_AUTH_SECRET_KEY`) and an opaque
**refresh token** stored hashed in `wp_pc_refresh_tokens`. Rotation happens on every
`/auth/refresh`; reuse detection revokes the whole descendant chain. Google sign-in
flows through `GoogleAuthController` and also requires the email-code 2FA; the whole
Google path is parked behind an empty `VITE_GOOGLE_CLIENT_ID`, and Apple is stubbed
until enrollment.

**Refresh-on-401 — FIXED.** The Axios response interceptor does *not* simply
redirect. On a 401 it performs one `POST /auth/refresh` (concurrent 401s share the
in-flight refresh), rewrites the stored pair, dispatches `auth:token-refreshed`, and
retries the failing request once. Only when there is no refresh token, the refresh
call itself fails, or the retry 401s again does it clear storage and dispatch
`auth:token-expired`, which signs the user out. This is why a machine or payment
fault must never surface as a 401.

**Gate order — FIXED.** Router guard: unauthenticated → `/sign-in` (with
`redirect`); then, for authenticated users on any route not flagged
`allowsBeforeGate`, `nicknameRequired` → `/choose-nickname`, then `!termsAccepted` →
`/accept-terms`. Gate views bounce back to `/` once satisfied. First social login
lands on `/choose-nickname`, then `/accept-terms` if the stored terms version is
behind `pc_terms_current_version`. Play-readiness (verified email) is enforced
server-side by `Permissions::require_play_ready` and client-side by `RoomView`'s
play handler, which sends unverified users to `/account?reason=verify-email`. The
router also contains a `meta.requiresPlayReady` check, but no route sets that meta.

**Inactivity.** `sessionService.js` watches mouse / keyboard / click / focus /
visibility and calls `authStore.logout(true)` after 15 idle minutes. The
access-token TTL is also 15 minutes.

**Top-up — FIXED.** The SPA calls `POST /wallet/topup`. The server writes a `pending`
transaction, asks Stripe for a Checkout Session (one line item, the whole order in
kopiykas, `adaptive_pricing` off so the price cannot be converted), stores the session
id as `external_ref`, and returns `checkout_url`; the browser goes to Stripe's hosted
page. When the payment succeeds Stripe delivers `checkout.session.completed` to
`POST /payments/stripe/webhook`, whose credential is the signature over the raw body.
That handler is the only place that flips a transaction `pending → completed`, inserts
the coin lot and credits the wallet — atomic, and idempotent on the row's own status
rather than on delivery order, because Stripe delivers at least once and out of order.
The coins credited come from the transaction row, never from the event: Stripe confirms
the money, the ledger decides what was bought. An expired or failed session parks the
row as `failed`. The player's return to `/account?topup=success` settles nothing; it is
only a redirect.

**Withdrawal — FIXED.** `POST /wallet/withdraw` FIFO-debits the lots into a `pending`
transaction (consumed slices preserved on `consumed_lots` for refunds; one pending
withdrawal per player at a time). An admin then approves (out-of-band payout, mark
`completed`) or rejects (re-credit at original prices, mark `refunded`) from the
admin SPA's withdrawals queue. Payouts are manual — there is no automated KYC or
payout pipeline.

**Queue and heartbeat — FIXED.** A player joins a room's queue declaring how many
coins they intend to play; FIFO order by `joined_at`; the head of the queue holds an
open bet session (`ended_at IS NULL`, at most one per room — **enforced by the
database** since `realtime` Sprint 2 Step 5: `wp_pc_bet_sessions.open_room_id` carries
the room id while the session is open and `UNIQUE KEY open_room` refuses the second
row, so two simultaneous requests cannot each open one).

**The SPA no longer polls it.** `join`, `leave` and `play` each announce the room on
its Ably channel, and the browser re-reads `GET /rooms/{id}/queue` only when it hears
a version it has not seen. The announcement carries `{room_id, version}` and **no
queue**: the channel is readable by any signed-in account while the read is
`require_play_ready`, so push decides *when* to read and the existing permission still
decides *what* may be read.

Holding a place is now `POST /rooms/{id}/queue/heartbeat`, every
`pc_queue_idle_timeout_seconds / 3` (20s by default, so two lost beats are harmless).
It does what the poll used to do as a side-effect — touch the caller, prune entries
whose `last_seen_at` has aged past the timeout, promote the next player — and answers
a version rather than the queue. It touches **before** pruning, unlike `GET queue`: a
client that has reached the server is not idle, and a backgrounded tab throttled by
the browser would otherwise be evicted by its own heartbeat. The version it returns is
also how a player promoted by *another* player's silence finds out, since nothing was
published.

The queue still heals on traffic alone and still has no cron; the traffic is now an
explicit heartbeat rather than a 3-second full read. If the channel cannot be
established at all, the store falls back to the old 3-second poll.

**Play (the toss) — FIXED.** `POST /rooms/{id}/play` is the one place a coin is
spent. In order: refuse if the caller is not at the head; refuse with 423
`relay_closed` while the machine is mid-payout; FIFO-debit one coin; call
`Machine_Service::toss_coin()`; if the machine does not answer 200, re-credit the
exact lot price. A successful toss increments `coins_played` on the session.

**Machine-event ingest.**
`POST /pc/v1/machine/events` (`app/realtime/MachineIngestController.php`) is where
machine events come in. It is a public route whose credential is a shared secret in
`X-PC-Machine-Secret` (`PC_MACHINE_INGEST_SECRET` in wp-config), rate-limited on one
ceiling across all callers rather than per IP, and audited on every call. It requires
an `event_key`, dispatches to the three `Machine_Ingest_Service` entry points, and
answers 200 with a `status` for everything a retry could not fix — a replayed key, an
event nobody can be paid for, a bonus mapped to no coins — so a transport stops
instead of redelivering forever. `Machine_Event_Log`
writes `wp_pc_machine_events` (idempotent on `event_key`); `Machine_Ingest_Service`
turns a bonus / relay-closed / coins-dropped event into a wallet credit, resolving
the player through the `pc_machine_event_player` filter and announcing the credit
through `pc_machine_event_credited`. `Queue_Service` hooks both (machine id → the
available room carrying that `pc_room_machine_id`, which `realtime`'s one-machine rule
keeps unique, else the room that carries it → open session → player; then bumps `coins_won` /
`money_won` on the session, which `UserControls` shows as per-turn winnings).
Machine payouts credit coin lots directly at the player's FIFO-head lot price and
are audited in `wp_pc_machine_events`, never in the ledger — the player's history
view shows money movements only.

**The transport that knocks on that door** is `Machine_Poller`
(`app/realtime/machine-poller.php`). Home Assistant has no outbound-HTTP service, so
it cannot call us, and reading `sensor.coin`'s live value on a schedule straddles
whole payouts — the counter rises and the next toss resets it in between. So
WordPress asks Home Assistant's *history* what the counter did since its own cursor
(`DECISIONS.md` 2026-09-18), through `Machine_Service::get_state_history()`, and
turns each change into coins: a rise pays the difference, a fall pays what is there
now because a toss reset the counter underneath it. Each payout is delivered to the
endpoint above with `rest_do_request()` — an in-process dispatch of the real route,
secret header included, so the transport is a client of the door like every other
caller rather than a back channel around it. `event_key` is
`ha:{entity}:{last_updated}` taken verbatim from the history row.

Its whole shape follows from one asymmetry: **re-reading a window is free and
skipping one is not.** The cursor (`pc_realtime_cursor_sensor_coin`) advances only
past rows the endpoint has answered, so a 401, a 429, a 500, an unreachable machine
or a missing configuration all hold it where it was and write the reason to
`wp_pc_auth_audit_log` (`machine_poll_*`); the next pass reads the same window again
and `event_key` makes the repeat free. A lost cursor costs a bounded backfill and an
audit row, never a silent gap. Two things drive it — the WP-Cron event the feature
bootstrap schedules and `wp pc machine-poll` — and a transient lock makes overlapping
passes impossible, so one real machine event is one ingest call on either. `wp pc
machine-ingest` remains the manual replay for an event the transport dropped.

**The same pass also watches the relay.** `Realtime_Relay_Watch`
(`app/realtime/relay-watch.php`) reads `sensor.relay_on` once per pass and, when it
has moved, caches the new state in `pc_realtime_relay_state` and announces it to the
room. The relay carries no payout signal — it idles closed and follows the operator's
own relay buttons (`DECISIONS.md` 2026-09-18) — so an **open** relay means the machine
has been taken out of service by hand, which is the one thing about it worth telling a
player. It runs *before* the coin work and outside its guards: a missing ingest secret
or an unreadable history stops that pass and the relay is still watched, and a relay
that cannot be read stops nothing and leaves the cached state alone, because an
unreadable relay is not a locked one. One extra state read a minute, no second
schedule, nothing new to deploy.

**The operator hears about a fault before a player does — FIXED, `realtime` Sprint 3
Step 1.** The same poll pass asks Home Assistant whether it is answering at all
(`Machine_Service::is_online()`, a 2-second probe of the HA root) and
`Realtime_Outage_Watch` (`app/realtime/outage-watch.php`) decides what that means. The
machine is switched off by hand at the venue every day, so unreachable is the normal
state most of the time: the room's broadcast schedule (`wp_pc_room_schedules` through
`Room_Schedule_Calculator`) is the gate. Unreachable past
`pc_realtime_outage_grace_seconds` is an **incident**, recorded once as
`machine_outage_started`; it is **notified** only while the room is inside a window, so
the nightly power-off is written down and never sent, and an outage that runs into a
window alerts when the window opens. Recovery writes `machine_outage_recovered` with the
duration, which is what gives an incident an end. Delivery is one function,
`Realtime_Alerts::send()` (`app/realtime/alerts.php`) — plain-text email to
`pc_realtime_alert_email`, falling back to `pc_support_email` then `admin_email`
(`DECISIONS.md` 2026-09-21). It runs next to the crediting path, so like publishing it
is fire-and-forget: every failure is audited and swallowed, and an install with no
address is a silent no-op. No second cron, no new dependency, no new secret.

**And a toss the machine did not act on is written down — `realtime` Sprint 3 Step 2.**
Home Assistant answers 200 to the toss *button*, not to the machine acting on it, so
`POST /rooms/{id}/play` cannot tell a real toss from a swallowed one. The same poll pass
settles it afterwards: `Realtime_Toss_Watch` (`app/realtime/toss-watch.php`) takes each
`toss` row of `wp_pc_machine_events` whose `pc_realtime_toss_window_seconds` has elapsed
and asks Home Assistant's history what `sensor.coin` did around it. The evidence is the
**counter's reset** — the machine zeroes it within ~2 s of accepting a toss — not the
coins, because a pusher pays nothing on most tosses. A counter that stood above zero and
never moved is the machine answering and doing nothing: one `toss_no_movement` row naming
the player, the session (`correlation_id`), the toss row and the reading on both sides,
which is what a dispute is settled with. A counter already at zero cannot be judged at
all — the reset would re-write a zero and Home Assistant records only changes — so it is
counted, not recorded, and the row keeps meaning one thing. Nothing runs inside the
player's request; a failed history read holds `pc_realtime_toss_cursor` and the next pass
re-reads, and a toss older than `pc_realtime_toss_max_age_seconds` is retired
`machine_toss_expired`. It records and does not notify: the sender above is one call away
when a step asks for it.

**And out to the browsers.** `Machine_Ingest_Service` fires
`pc_machine_event_credited` once the wallet has moved; `Realtime_Publisher`
(`app/realtime/publisher.php`) hooks it, resolves the machine id to its room and posts
one compact `credit` message to that room's Ably channel — `user_id`, `coins`,
`room_id`, `event_id`, `at`. **No money:** a room channel is readable by everyone
watching the room, and what another player's coins cost them is not theirs to see, so
`unit_price` stays behind the authenticated endpoints the winner's own totals come
from. Channel names are built in one place (`app/realtime/channels.php`) and handed to
the SPAs by the token endpoint, so neither hardcodes them. The publish is
fire-and-forget in the strict sense: it runs after the credit, and every failure is
logged and swallowed rather than returned (`FEATURE.md` → Invariants #2).

**Chat.** Reads are public and cursor-based — `GET /rooms/{id}/messages?after=<last
id>` — so a guest watching a broadcast sees the conversation read-only. **A signed-in
client no longer polls it.** A posted message is published on the room's channel
carrying the whole message, and moderation is published as an id and a state; the
cursor stopped being the transport and became the catch-up, run on every connect and
reconnect (`realtime` Sprint 2 Step 4). A chat body may travel where a queue entry may
not, and it is the same rule in both directions — what may travel is what the read
already gives away, and this read is public. Two cases still poll on the 3-second
interval, both deliberate: a **guest**, who has no Ably pass because the token endpoint
requires signing in, and any client whose channel cannot be established. Writes go
through `require_chat_ready`, are capped at 500 sanitised plain-text characters, and
are rate-limited to 10 per minute per account. Chat deliberately does not require the
room to be `available` the way the queue does: a room in maintenance is where players
ask what is going on. Moderation is two verbs — hiding flips a `status` column rather
than deleting the row, so the author, body, IP and timestamp survive for whoever
reviews the complaint; muting writes a `chat_muted_until` timestamp to user meta and
is account-wide, enforced server-side on every post. Both are audited.

**Support ticket.** `SupportController` exposes the subject list
(`pc_support_subject` CPT, ordered by `menu_order`) and accepts tickets into
`wp_pc_support_tickets` via `Support_Service`: rate-limited per IP (5/hour), audited
as `support_ticket_created`, stored with IP / UA / the `email_verified` state at
submission time, and mailed to `pc_support_email` with the player as `Reply-To`.
Guests must pass a captcha **only if one is configured**; when the provider or secret
is empty the guest path runs unchallenged and the admin SPA's captcha panel says so
in red. Retiring a subject trashes the post so old tickets still resolve their label.

**Live streaming.** `LiveStream.vue` sniffs the room's `stream_url` to pick a
transport: iframe for embed-style URLs (YouTube / Vimeo / `/embed/...`), `<video>`
for direct video files, **Mux LL-HLS via `hls.js`** for `.m3u8` URLs (the production
path). `hls.js` is dynamically imported so it ships as its own ~162KB gzipped chunk,
only loaded when an HLS stream is about to play. Safari skips the library and uses
its native HLS support. RTMP ingest from the venue → Mux → LL-HLS playback URL
persisted in the room's `pc_room_stream_url` post meta.

**Trust boundaries.**
- The browser is untrusted; only the JWT travels back to identify the user. Balance
  rules the SPA enforces (coin-quantity clamps, zero-balance routing) are re-checked
  server-side.
- Every REST route names an explicit `permission_callback`. Twelve are public by
  design and rely on a bearer credential in the body, a payment-provider signature, a
  rate limit, or a captcha instead of a session. Hardening those — nonces, captcha
  coverage, tighter limits — is the open security-review item in `ROADMAP.md`.
- The backend is the only party that holds machine and payment credentials; the SPAs
  never see the HA endpoint, the Stripe secret key, or the captcha secret.
- A custom `player` role is added at `init` with a `play` capability, and
  administrators are given `play` for parity. **No code checks `play` today** — every
  gate goes through `Permissions::*`, which checks login state, user meta, and
  `manage_options`. The role is effectively a label.

## Integrations

| Service | What we use it for | Auth model | Failure / fallback | Credentials |
|---|---|---|---|---|
| **Home Assistant** | The physical machine: power on/off, `toss_coin()`, sensor reads (coin count, bonus number, light bitfield, relay state), relay open/close, a soft-failing batched snapshot for the admin view, `is_online()`, and `get_state_history()` — every state an entity passed through between two instants, which is what the inbound transport polls. Only `Machine_Service` calls it. | Bearer token | 2s HTTP timeout, except history reads at 10s (a wider window is slower, and the toss path's refund threshold must not move with it); typed `WP_Error` (`machine_not_configured`, `machine_offline`, `machine_unauthorized`, `machine_call_failed`, `machine_unavailable_state`) mapped by callers to 502 / 503 so a machine fault never looks like an auth failure. The batched snapshot soft-fails per field. | `PC_MACHINE_TOKEN` (wp-config). Base URL + entity ids are `pc_machine_*` WP options, defaults matching `PUSHER-COIN-COMMANDS.txt`. |
| **Stripe** | Hosted Checkout for top-ups (UAH) and the settlement webhook — the only place a top-up reaches `completed`. Only `Stripe_Client` calls it. No SDK: plain `wp_remote_post` against a pinned API version (`2026-06-24.dahlia`). | Bearer secret key outbound; an HMAC signature over the raw body inbound | Session creation failing is `stripe_call_failed` 502 and the row is parked `failed`. Inbound: a bad signature is 401 and a rolled-back settlement is 500, both of which Stripe retries for three days; every other condition answers 200 with a `note` so Stripe stops. `adaptive_pricing` is sent `false` so the presented currency cannot be converted. | `PC_STRIPE_SECRET_KEY` and `PC_STRIPE_WEBHOOK_SECRET` (wp-config). The provider has no WP option at all. |
| **Ably** | The push channel out to the browsers: WordPress publishes machine events to a room's channel and the SPAs subscribe, replacing the 3-second polls. Free tier — 6M messages/month, 200 concurrent connections, 200 channels. Only `Realtime_Publisher` calls it, and only ever to publish. | An app key as HTTP Basic outbound. Browsers never get the key: `GET /pc/v1/realtime/token` answers an HMAC-signed Ably *token request*, scoped to `subscribe` on `{prefix}:room:*`, plus `{prefix}:machine` for admins — never `publish`. | **Fire-and-forget.** A publish runs after the wallet has already moved and cannot affect it: every failure is caught, written to `wp_pc_auth_audit_log` as `realtime_publish_failed` and swallowed, with a 2s timeout. An unconfigured key is a silent no-op, so an install with no Ably account credits players normally and simply does not push; the token endpoint then answers `realtime_not_configured` 503. | `PC_ABLY_KEY` (wp-config), in Ably's `name:secret` form. The channel prefix and token TTL are `pc_realtime_*` WP options. |
| **Mux** | LL-HLS playback of the venue's RTMP stream. | Playback URL only | `LiveStream.vue` falls back to `<video>` or an iframe by URL shape; Safari uses native HLS. | None in the app — the playback URL is `pc_room_stream_url` post meta. |
| **Turnstile / hCaptcha** | Guest anti-abuse on `POST /support/tickets`. | siteverify call with a secret | **Unconfigured is the off switch**: with an empty provider or secret the guest path runs unchallenged and the admin panel flags it in red. This is a launch blocker — see `backend/wp-content/themes/pc/CAPTCHA_SETUP.md`. | `PC_CAPTCHA_SECRET` (wp-config); `pc_captcha_site_key` + provider are WP options. |
| **Google Sign-In** | ID-token exchange, then the same email-code 2FA. | Google ID token verified with `google/apiclient` | Parked: the SPA hides the button while `VITE_GOOGLE_CLIENT_ID` is empty. Backend untouched. | `GOOGLE_CLIENT_ID` (wp-config, the audience). See `backend/wp-content/themes/pc/GOOGLE_AUTH_SETUP.md`. |
| **Apple Sign-In** | Stub. | — | Returns `apple_not_configured` until enrollment completes. | `APPLE_CLIENT_ID`, `APPLE_TEAM_ID`, `APPLE_KEY_ID`, `APPLE_PRIVATE_KEY` (wp-config). |
| **`jwt-authentication-for-wp-rest-api`** | Validates incoming bearer tokens, sets the current user, exposes `jwt_auth_expire`. | — | — | `JWT_AUTH_SECRET_KEY` (wp-config). |
| **WP mail** | Verification codes, email-confirmation links, password-change codes, support-ticket notifications. | Server MTA | No retry or queue; a failed send is not surfaced to the player. | — |

Secret rotation for any of the above is a wp-config edit on the server. FTP
credentials live in GitHub Actions secrets.

## Environments & deploy

| Environment | How it runs | Deploy |
|---|---|---|
| **Local — backend** | DDEV: `.ddev/config.yaml` + `wp-config-ddev.php`, pulled in by the git-ignored `wp-config.php`. Host `https://pusher-coin.ddev.site`. | `ddev start` |
| **Local — player SPA** | Vite dev server on `:5173`, `.env` points at the DDEV host. | `npm run dev` |
| **Local — admin SPA** | Vite dev server on `:5174`. | `npm run dev` |
| **Production — backend** | The same WordPress tree on shared hosting. **The machine transport needs two things no deploy can write:** `PC_MACHINE_INGEST_SECRET` in `wp-config.php` (FTP sync never touches that file) and `pc_realtime_poll_machine_id` set to the machine id the live rooms carry. Until both exist the poller holds its cursor and credits nothing — which is recoverable, because Home Assistant keeps ten days of history. The schedule runs on WP-Cron by default; a host with a real cron can instead set `DISABLE_WP_CRON` and add `* * * * * cd /path/to/wordpress && wp pc machine-poll --quiet`. `wp pc machine-poll --status` says which is running. | `backend/.github/workflows/main.yml`: `php -l` over the theme on every push and PR, then — only on push to `main`, only if lint passed — an FTP sync of the **entire backend tree** using repo secrets for host, user, password and target path. **A push to `main` is a production release.** |
| **Production — player SPA** | Vercel, built by Vite with `.env.production`. | `vercel.json`; CI (`frontend/.github/workflows/ci.yml`) runs `npm ci`, lint and build on every push and PR. |
| **Production — admin SPA** | Does not exist. | No `vercel.json`, no workflow. Local-only. |

Operator-tunable settings are WP options (coin pricing, machine entity ids, bonus
map, support address, captcha provider / site key, queue idle timeout). Every
option, table and meta key is catalogued in `DATA-MODEL.md`; `Install_Schema` seeds
the defaults and owns `pc_db_version`.
