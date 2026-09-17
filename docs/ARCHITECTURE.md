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
works end to end over HTTP polling. What is still missing is the real-time
machine-event transport, which is why several sections below say "polled every 3s".
Phase status lives in `ROADMAP.md`.

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
         /wp-json/jwt-auth/v1        Home Assistant REST         LiqPay checkout            Turnstile / hCaptcha
         (plugin validates            (Machine_Service,           (form POST out,            siteverify
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
| `stores/` | Pinia stores. `authentication.js` is the central one (token + user, persisted to `localStorage`, with Google 2FA state). `wallet.js` (balance, lots, pricing, top-up), `queue.js` (room queue, 3s poll that doubles as the heartbeat), `rooms.js` (room list, 30s cache), `navigation.js`, `chat.js` (panel open/closed state *plus* the conversation itself — 3s poll with an `after` cursor), and `themeSong.js` (per-room theme song; owns the `Audio` element because the toggle lives in `UserControls` while the URL arrives with the room in `RoomView`). `counter.js` and `user.js` are unused scaffold. |
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
`components/AdminLayout.vue` (header + nav + slot). The nav has six sections —
Rooms, Withdrawals, Machine, Support, Chat, Settings:

| Route | View | What it does |
| --- | --- | --- |
| `/sign-in` | `SignInView` | Two-step email/password + 6-digit code form. |
| `/rooms` | `RoomListView` | Table with status badge + per-row Edit / Schedule / Trash. |
| `/rooms/new`, `/rooms/:id/edit` | `RoomFormView` | Shared create / edit form; discriminates on `route.name`. |
| `/rooms/:id/schedule` | `RoomScheduleView` | Weekly rules editor; save is an atomic replace via `PUT /admin/rooms/{id}/schedule`, response carries the recomputed `next_window`. |
| `/withdrawals` | `WithdrawalsView` | Pending-withdrawal queue with filter tabs and an approve / reject dialog (reject asks for a reason). |
| `/machine` | `MachineView` | Connection probe, power On/Off, sensor grid (coin counter, last bonus, relay, light bitfield). Polls `GET /admin/machine/state` every 3s. |
| `/support/tickets` | `TicketsView` | Ticket queue: status filter, search, expandable message with IP / UA, status transitions, mailto reply. |
| `/support/subjects` | `SubjectsView` | Subject list editor (reorder, hide, replace-all save) plus the guest-captcha provider / site-key panel. |
| `/chat` | `ChatView` | Chat moderation queue: room / status / text filters, hide and restore a message, and a timed account-wide mute. |
| `/settings` | `SettingsView` | Coin price default / min / max, LiqPay public-key hint, bonus-map 4×3 grid, relay coin count. |

Stores: `auth.js` (two-step sign-in + `/admin/me` gate), `rooms.js`,
`withdrawals.js`. Services mirror the backend admin controllers one to one:
`adminAuthService`, `adminRoomsService`, `adminWithdrawalsService`,
`adminCoinPricingService`, `adminMachineService`, `adminSupportService`,
`adminChatService`. Token storage: `pc_admin_auth_token`, `pc_admin_user_data`;
`services/api.js` is a copy of the player interceptor with the same refresh-on-401
behaviour (events prefixed `admin-auth:`).

Not built: no `vercel.json` and no CI workflow; no shared component package with
`frontend/` (some duplication is accepted); no inactivity timer; no operations
dashboard aggregating the six sections.

### `pc` theme — REST layer

**Location.** `backend/wp-content/themes/pc/app/rest-api.php` + `app/rest-api/`.

**Responsibility.** Validate input, declare the permission callback, delegate to a
service, shape the response. Controllers hold no domain logic.

**Public surface.** The `pc/v1` namespace — 18 controllers. The complete catalogue
with request / response / error shapes is `CONTRACTS.md`.

**Must never do.** Talk to Home Assistant, LiqPay or the captcha provider directly;
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
| `PaymentController.php` | LiqPay signed webhook |
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
  room page it sits on); `POST /payments/liqpay/callback` (LiqPay signature
  verified in the handler).
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
  `POST /admin/withdrawals/{id}/approve`, `/reject`; `GET/PUT /admin/coin-pricing`;
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
│   ├── stripe-client.php    # `ddev wp eval-file` check: kopiyka conversion, mode / configuration, webhook signature scheme (DDEV only)
│   └── wallet-rollback.php  # `ddev wp eval-file` check: every Wallet_Service write failure rolls back (DDEV only)
└── app/
    ├── rest-api.php         # Wires controllers into `rest_api_init`
    ├── rest-api/            # The 18 controllers listed above
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
        ├── liqpay-client.php               # LiqPay sign / verify / decode
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

**Feature directories.** A feature added after the playbook (`app/stripe/`, and
`app/realtime/` when it lands) owns a directory beside `utils/` and is reached through
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

*Until Sprint 1 Step 5 the player SPA still speaks the LiqPay envelope, so the
hand-off is broken between Steps 3 and 5 by design; the LiqPay callback also still
exists, but nothing creates a row it could settle.*

**Withdrawal — FIXED.** `POST /wallet/withdraw` FIFO-debits the lots into a `pending`
transaction (consumed slices preserved on `consumed_lots` for refunds; one pending
withdrawal per player at a time). An admin then approves (out-of-band payout, mark
`completed`) or rejects (re-credit at original prices, mark `refunded`) from the
admin SPA's withdrawals queue. Payouts are manual — there is no automated KYC or
payout pipeline.

**Queue and heartbeat — FIXED.** A player joins a room's queue declaring how many
coins they intend to play; FIFO order by `joined_at`; the head of the queue holds an
open bet session (`ended_at IS NULL`, at most one per room). The SPA polls
`GET /rooms/{id}/queue` every 3s and that poll *is* the heartbeat — entries whose
`last_seen_at` is older than `pc_queue_idle_timeout_seconds` are pruned on the next
read, so the queue heals on traffic alone with no cron.

**Play (the toss) — FIXED.** `POST /rooms/{id}/play` is the one place a coin is
spent. In order: refuse if the caller is not at the head; refuse with 423
`relay_closed` while the machine is mid-payout; FIFO-debit one coin; call
`Machine_Service::toss_coin()`; if the machine does not answer 200, re-credit the
exact lot price. A successful toss increments `coins_played` on the session.

**Machine-event ingest — FIXED in shape, no transport yet.** `Machine_Event_Log`
writes `wp_pc_machine_events` (idempotent on `event_key`); `Machine_Ingest_Service`
turns a bonus / relay-closed / coins-dropped event into a wallet credit, resolving
the player through the `pc_machine_event_player` filter and announcing the credit
through `pc_machine_event_credited`. `Queue_Service` hooks both (machine id → room
via `pc_room_machine_id` → open session → player; then bumps `coins_won` /
`money_won` on the session, which `UserControls` shows as per-turn winnings).
Machine payouts credit coin lots directly at the player's FIFO-head lot price and
are audited in `wp_pc_machine_events`, never in the ledger — the player's history
view shows money movements only. Nothing pushes machine events *in* yet: the only
producer today is the `wp pc machine-ingest` replay command.

**Chat.** Reads are public and cursor-based — `GET /rooms/{id}/messages?after=<last
id>`, polled every 3s by the same store that owns the chat panel's open/closed
state — so a guest watching a broadcast sees the conversation read-only. Writes go
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
  never see the HA endpoint, the LiqPay private key, or the captcha secret.
- A custom `player` role is added at `init` with a `play` capability, and
  administrators are given `play` for parity. **No code checks `play` today** — every
  gate goes through `Permissions::*`, which checks login state, user meta, and
  `manage_options`. The role is effectively a label.

## Integrations

| Service | What we use it for | Auth model | Failure / fallback | Credentials |
|---|---|---|---|---|
| **Home Assistant** | The physical machine: power on/off, `toss_coin()`, sensor reads (coin count, bonus number, light bitfield, relay state), relay open/close, a soft-failing batched snapshot for the admin view, `is_online()`. Only `Machine_Service` calls it. | Bearer token | 2s HTTP timeout; typed `WP_Error` (`machine_not_configured`, `machine_offline`, `machine_unauthorized`, `machine_call_failed`, `machine_unavailable_state`) mapped by callers to 502 / 503 so a machine fault never looks like an auth failure. The batched snapshot soft-fails per field. | `PC_MACHINE_TOKEN` (wp-config). Base URL + entity ids are `pc_machine_*` WP options, defaults matching `PUSHER-COIN-COMMANDS.txt`. |
| **Stripe** | Hosted Checkout for top-ups (UAH) and the settlement webhook — the only place a top-up reaches `completed`. Only `Stripe_Client` calls it. No SDK: plain `wp_remote_post` against a pinned API version (`2026-06-24.dahlia`). | Bearer secret key outbound; an HMAC signature over the raw body inbound | Session creation failing is `stripe_call_failed` 502 and the row is parked `failed`. Inbound: a bad signature is 401 and a rolled-back settlement is 500, both of which Stripe retries for three days; every other condition answers 200 with a `note` so Stripe stops. `adaptive_pricing` is sent `false` so the presented currency cannot be converted. | `PC_STRIPE_SECRET_KEY` and `PC_STRIPE_WEBHOOK_SECRET` (wp-config). The provider has no WP option at all. |
| **LiqPay** | *Being removed (Sprint 1 Step 5).* Its callback route still exists, but nothing creates a row it could settle — top-ups have gone through Stripe since Step 3. | HMAC signature on both directions | A callback with a bad signature is rejected; a repeated callback is a no-op (idempotent on `(order_id, status)`). | `PC_LIQPAY_PRIVATE_KEY` (wp-config). Its public-key option was deleted at `pc_db_version` 1.9.0. |
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
| **Production — backend** | The same WordPress tree on shared hosting. | `backend/.github/workflows/main.yml`: `php -l` over the theme on every push and PR, then — only on push to `main`, only if lint passed — an FTP sync of the **entire backend tree** using repo secrets for host, user, password and target path. **A push to `main` is a production release.** |
| **Production — player SPA** | Vercel, built by Vite with `.env.production`. | `vercel.json`; CI (`frontend/.github/workflows/ci.yml`) runs `npm ci`, lint and build on every push and PR. |
| **Production — admin SPA** | Does not exist. | No `vercel.json`, no workflow. Local-only. |

Operator-tunable settings are WP options (coin pricing, machine entity ids, bonus
map, support address, captcha provider / site key, queue idle timeout). Every
option, table and meta key is catalogued in `DATA-MODEL.md`; `Install_Schema` seeds
the defaults and owns `pc_db_version`.
