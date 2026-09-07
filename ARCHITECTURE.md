# Pusher Coin — Architecture

Pusher Coin is a real-time, browser-based coin-pusher gambling application split into three independent applications that live side by side in this repository:

- `frontend/` — a Vue 3 single-page application (SPA) that the player interacts with.
- `admin/` — a separate Vue 3 SPA used by operators to manage rooms and schedules, review withdrawals, switch and monitor the physical machine, set coin pricing and the bonus map, triage support tickets, and moderate chat. Introduced in Phase 3. See `ADMIN-DECISION.md` for the rationale.
- `backend/` — a WordPress installation that exposes a JSON REST API used by both SPAs (the WordPress admin/HTML side is not the user-facing product). It is also the only component that talks to the physical machine and to the payment provider.

The three apps are decoupled: separate `.git` repositories (the root repo ignores all three and tracks only the documentation), separate deploy pipelines, and communication exclusively over HTTPS/JSON.

Phase status lives in `ROADMAP.md`. As of Phase 7 the whole loop — sign up, verify, top up, queue, toss, settle, withdraw, support — works end to end over HTTP polling. What is still missing is the real-time machine-event transport (Phase 5 steps 6–7), which is why several sections below say "polled every 3s".

---

## High-level shape

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
2FA-by-email-code) but live behind distinct localStorage keys
(`pusher_coin_*` for the player, `pc_admin_*` for the admin). The admin
SPA additionally probes `GET /pc/v1/admin/me` after sign-in to bounce
non-admins.

Data flow at a glance:

1. The SPA boots from `frontend/src/main.js`, mounts `App.vue`, installs Pinia + Vue Router.
2. The router (`src/router/index.js`) decides which `views/*.vue` to render and gates routes against the auth store: `requiresAuth` / `requiresGuest`, then the nickname and terms gates (see *Authentication flow* below).
3. View components compose `components/*.vue` and call `services/*.js` (an Axios client) which appends the JWT bearer token from `localStorage` to every request.
4. Requests hit the WordPress REST API at `VITE_API_BASE_URL` (`https://pusher-coin.ddev.site/wp-json/pc/v1` locally; the staging host in `.env.production`).
5. Inside WordPress, the custom theme `pc` registers 18 REST controllers under `pc/v1`. Controllers validate, gate, and delegate; domain logic sits in `app/utils/*` service classes (`Wallet_Service`, `Queue_Service`, `Chat_Service`, `Machine_Service`, `Machine_Ingest_Service`, `Support_Service`).
6. Side channels: LiqPay calls back `POST /payments/liqpay/callback` after checkout; the theme calls *out* to Home Assistant for every toss, power switch, and sensor read. Nothing pushes machine events *in* yet — the only producer of machine events today is the `wp pc machine-ingest` replay command.

---

## Frontend (`frontend/`)

A Vue 3 + Vite SPA. Uses the Composition API throughout.

**Stack**
- Vue 3.5 with `<script setup>` Single-File Components.
- Vite 5 as dev server / bundler.
- Vue Router 4 for client-side routing.
- Pinia 3 for state management.
- Axios for HTTP.
- IMask for input masking (phone, codes, coin quantities).
- `hls.js` for Mux LL-HLS playback, dynamically imported so it only loads when a stream plays.
- ESLint + Prettier for code quality.
- Deployed via `vercel.json`; environment split between `.env` (local DDEV) and `.env.production`. CI (`.github/workflows/ci.yml`) runs `npm ci`, lint, and build on every push and PR.

**Layout (`src/`)**

| Folder | Responsibility |
| --- | --- |
| `main.js` | App entry point. Creates the Vue app, registers Pinia and the router, mounts `#app`. |
| `App.vue` | Root layout: header + `NavigationToggle`, side `AppNavigation`, `<RouterView />`, footer. The header switches to a room variant on the room route. |
| `router/index.js` | Route table and `beforeEach` guard. Routes carry `meta.requiresAuth` / `meta.requiresGuest` / `meta.allowsBeforeGate`; the guard awaits `authStore.initializeAuth()` before deciding. |
| `views/` | Page-level components mapped 1:1 to routes: `RoomsView` (`/`), `RoomView` (`/room/:id`, public), `SignInView`, `SignUpView`, `AccountView`, `HistoryView`, `SupportView` (public), and the three gate / landing views `AcceptTermsView`, `ChooseNicknameView`, `ConfirmEmailView`. `AboutView` exists but is not in the route table. |
| `components/` | Reusable building blocks. Room page: `LiveStream`, `RoomChat` (live, 3s poll, owns its poll lifecycle), `RoomQueue`, `PlaceBet`, `UserControls`, `RoomStatusBadge`, `NextBroadcastCountdown`. Lists / shell: `RoomList`, `AppNavigation`, `NavigationToggle`, `LanguageSwitcher`, `ModalOverlay`, `LogoutConfirmModal`. Account / money: `FacelessAvatar`, `ReplenishmentBalance`, `WithdrawalRequest`. Auth: `SignInForm`, `SignUpForm`, `GoogleSignInButton` (hidden while parked), `AppleSignInButton` (hidden until configured). Plus an `icons/` set of single-purpose SVG components. `HelloWorld.vue` is Vite scaffold with no importers. |
| `stores/` | Pinia stores. `authentication.js` is the central one (token + user, persisted to `localStorage`, with Google 2FA state). `wallet.js` (balance, lots, pricing, top-up), `queue.js` (room queue, 3s poll that doubles as the heartbeat), `rooms.js` (room list, 30s cache), `navigation.js`, `chat.js` (panel open/closed state *plus* the conversation itself — 3s poll with an `after` cursor), and `themeSong.js` (per-room theme song; owns the `Audio` element because the toggle lives in `UserControls` while the URL arrives with the room in `RoomView`). `counter.js` and `user.js` are unused scaffold. |
| `services/` | API layer. `api.js` is a configured Axios instance with request/response interceptors (auto-attaches the JWT; refreshes once on 401 — see below). Endpoint wrappers: `authService`, `accountService`, `userService`, `googleAuthService`, `appleAuthService`, `roomsService`, `queueService`, `chatService`, `walletService`, `historyService`, `supportService`; `liqpayCheckout.js` builds and submits the hosted-checkout form; `sessionService.js` is the inactivity timer. |
| `assets/` | Global CSS (`main.css`, `styles/colors.css`, block-scoped CSS in `styles/blocks/`), images, the brand SVG logo. |
| `public/` | Static files served verbatim by Vite (`favicon.ico`). |

**Authentication flow (frontend side)**
- Token + user are stored under `pusher_coin_auth_token` / `pusher_coin_user_data` in `localStorage`; the refresh token and access-token expiry ride inside the user blob.
- The only other `localStorage` key the player SPA writes is `pc_theme_song_enabled` (`'1'` / `'0'`) — a device-local preference, never sent to the server.
- The auth store exposes `initializeAuth()` (called once by the router guard), `isAuthenticated`, `nicknameRequired`, `termsAccepted`, `emailVerified`, plus actions for password login, sign-up, email-code 2FA, Google / Apple OAuth, and `logout()` (which revokes the refresh token server-side first).
- **Refresh-on-401.** The Axios response interceptor does *not* simply redirect. On a 401 it performs one `POST /auth/refresh` (concurrent 401s share the in-flight refresh), rewrites the stored pair, dispatches `auth:token-refreshed`, and retries the failing request once. Only when there is no refresh token, the refresh call itself fails, or the retry 401s again does it clear storage and dispatch `auth:token-expired`, which the store handles by signing the user out.
- **Gate order in the router guard:** unauthenticated → `/sign-in` (with `redirect`); then, for authenticated users on any route not flagged `allowsBeforeGate`, `nicknameRequired` → `/choose-nickname`, then `!termsAccepted` → `/accept-terms`. Gate views bounce back to `/` once satisfied.
- **Play-readiness (verified email)** is enforced server-side by `Permissions::require_play_ready` on the wallet and queue endpoints, and client-side by `RoomView`'s play handler, which sends unverified users to `/account?reason=verify-email`. The router also contains a `meta.requiresPlayReady` check, but no route currently sets that meta.
- **Inactivity.** `sessionService.js` watches mouse / keyboard / click / focus / visibility and calls `authStore.logout(true)` after 15 idle minutes. The access-token TTL is also 15 minutes.

**Path aliases**
- `@` → `src/` (configured in `vite.config.js` and `jsconfig.json`).

---

## Admin SPA (`admin/`)

Introduced in Phase 3. Same stack as `frontend/` (Vue 3 + Vite + Pinia
+ Vue Router + Axios + ESLint/Prettier). Runs on port 5174 in dev so
both SPAs can run side-by-side. Does **not** include IMask or `hls.js`.

**Auth.** Reuses the player 2FA flow (`/user/request-verification` →
`/user/verify-code`) to issue a JWT pair, then probes
`GET /pc/v1/admin/me` to confirm `manage_options`. Non-admins are
signed out server-side (refresh-token revoked) before the SPA shows the
error. Distinct localStorage keys (`pc_admin_auth_token`,
`pc_admin_user_data`) so the two SPAs can coexist on the same origin
during dev. `services/api.js` is a copy of the player interceptor with
the same refresh-on-401 behaviour (events are prefixed `admin-auth:`).

**Shape.** `App.vue` is a bare `<RouterView />`; every authenticated
view wraps itself in `components/AdminLayout.vue` (header + nav +
slot). The nav has six sections — Rooms, Withdrawals, Machine, Support,
Chat, Settings — and the router exposes:

| Route | View | Phase | What it does |
| --- | --- | --- | --- |
| `/sign-in` | `SignInView` | 3 | Two-step email/password + 6-digit code form. |
| `/rooms` | `RoomListView` | 3 | Table with status badge + per-row Edit / Schedule / Trash. |
| `/rooms/new`, `/rooms/:id/edit` | `RoomFormView` | 3 | Shared create / edit form; discriminates on `route.name`. |
| `/rooms/:id/schedule` | `RoomScheduleView` | 3 | Weekly rules editor; save is an atomic replace via `PUT /admin/rooms/{id}/schedule`, response carries the recomputed `next_window`. |
| `/withdrawals` | `WithdrawalsView` | 4 | Pending-withdrawal queue with filter tabs and an approve / reject dialog (reject asks for a reason). |
| `/machine` | `MachineView` | 5 | Connection probe, power On/Off, sensor grid (coin counter, last bonus, relay, light bitfield). Polls `GET /admin/machine/state` every 3s. |
| `/support/tickets` | `TicketsView` | 7 | Ticket queue: status filter, search, expandable message with IP / UA, status transitions, mailto reply. |
| `/support/subjects` | `SubjectsView` | 7 | Subject list editor (reorder, hide, replace-all save) plus the guest-captcha provider / site-key panel. |
| `/chat` | `ChatView` | 6 | Chat moderation queue: room / status / text filters, hide and restore a message, and a timed account-wide mute. |
| `/settings` | `SettingsView` | 4 + 5 | Coin price default / min / max, LiqPay public-key hint, bonus-map 4×3 grid, relay coin count. |

Stores: `auth.js` (two-step sign-in + `/admin/me` gate), `rooms.js`,
`withdrawals.js`. Services mirror the backend admin controllers one to
one: `adminAuthService`, `adminRoomsService`, `adminWithdrawalsService`,
`adminCoinPricingService`, `adminMachineService`, `adminSupportService`,
`adminChatService`.

**Deferred.** No `vercel.json` and no CI workflow (no deploy target
chosen — it is local-only); no shared component package with
`frontend/` (`ADMIN-DECISION.md` accepts some duplication for now); no
inactivity timer (planned, tighter timeout than the player SPA's 15
minutes); no operations dashboard that aggregates the five sections
(ROADMAP Phase 7 §3).

---

## Backend (`backend/`)

A standard WordPress installation. The product code lives only in two places; everything else is unmodified WordPress core or third-party plugins.

**Custom theme: `wp-content/themes/pc/`**

This is *the* application code on the backend — WordPress acts as an authentication, user-data, and integration service rather than a CMS. `functions.php` loads the composer autoloader, `app/utils.php` (which requires every utility below in dependency order), and `app/rest-api.php` (which instantiates every controller on `rest_api_init`).

```
themes/pc/
├── functions.php            # Bootstrap: composer autoload, utils, REST API registration, jwt_auth_expire filter
├── style.css                # Theme metadata header
├── index.php                # Empty/placeholder (no front-end rendering)
├── composer.json            # google/apiclient (Google ID-token verification); autoloader required by functions.php
├── GOOGLE_AUTH_SETUP.md     # Operator notes for Google OAuth (parked — see ROADMAP Phase 1 §2)
├── CAPTCHA_SETUP.md         # Operator notes for Turnstile / hCaptcha keys + rotation (Phase 7)
└── app/
    ├── rest-api.php         # Wires controllers into `rest_api_init`
    ├── rest-api/
    │   ├── UserController.php              # Phase 1+2 — sign-up, 2FA, /user/me, email confirmation, password change
    │   ├── AuthController.php              # Phase 1 — /auth/logout, /auth/refresh, token-pair issuance
    │   ├── GoogleAuthController.php        # Google ID-token exchange + 2FA (backend untouched while parked)
    │   ├── AppleAuthController.php         # Apple Sign-In (returns apple_not_configured until enrolled)
    │   ├── RoomController.php              # Phase 3 — public /rooms reads + schedule
    │   ├── RoomQueueController.php         # Phase 6 — /rooms/{id}/queue, join, leave, play (toss)
    │   ├── RoomChatController.php           # Phase 6 — /rooms/{id}/messages: public read, gated post
    │   ├── WalletController.php            # Phase 4 — GET /wallet, POST /wallet/topup, POST /wallet/withdraw
    │   ├── PaymentController.php           # Phase 4 — LiqPay signed webhook
    │   ├── TransactionsController.php      # Phase 4 — GET /transactions (paginated, filterable)
    │   ├── SupportController.php           # Phase 7 — public /support/subjects + /support/tickets
    │   ├── AdminController.php             # Phase 3 — /admin/me capability probe
    │   ├── AdminRoomController.php         # Phase 3 — admin /rooms CRUD + atomic schedule replace
    │   ├── AdminWithdrawalController.php   # Phase 4 — /admin/withdrawals list, approve, reject
    │   ├── AdminCoinPricingController.php  # Phase 4 — GET/PUT /admin/coin-pricing
    │   ├── AdminMachineController.php      # Phase 5 — /admin/machine state, power, bonus-map
    │   ├── AdminSupportController.php      # Phase 7 — /admin/support tickets, subjects, captcha config
    │   └── AdminChatController.php          # Phase 6 — /admin/chat: moderation queue, hide/restore, mute
    ├── utils.php
    └── utils/
        ├── role-player.php                 # Registers the `player` role (see Trust boundaries)
        ├── user-meta-keys.php              # User_Meta_Keys registry — only way to name a user meta key
        ├── post-meta-keys.php              # Post_Meta_Keys registry — pc_room meta
        ├── permissions.php                 # Permission_callback helpers: logged-in / terms / nickname / email / play-ready / admin
        ├── install-schema.php              # Custom-table installer (dbDelta, pc_db_version = 1.7.0) + default options
        ├── audit-log.php                   # Audit_Log writer → wp_pc_auth_audit_log
        ├── rate-limiter.php                # Transient-based per-IP rate limiter (pc_rl_* keys)
        ├── refresh-tokens.php              # Refresh-token issuance / rotation / reuse detection
        ├── cpt-room.php                    # Registers pc_room CPT (Phase 3)
        ├── cpt-support-subject.php         # Registers pc_support_subject CPT (Phase 7)
        ├── room-schedule-calculator.php    # Computes current_window / next_window from weekly rules
        ├── wallet-service.php              # Phase 4 — atomic wallet / coin-lot / ledger ops (SELECT … FOR UPDATE)
        ├── liqpay-client.php               # Phase 4 — LiqPay sign / verify / decode
        ├── machine-service.php             # Phase 5 — Home Assistant REST wrapper (2s timeout, typed WP_Error)
        ├── machine-events.php              # Phase 5 — Machine_Event_Log writer → wp_pc_machine_events
        ├── machine-ingest-service.php      # Phase 5 — machine event → wallet credit; transport-agnostic
        ├── queue-service.php               # Phase 6 — queue, turns, bet sessions, machine-event attribution
        ├── chat-service.php                 # Phase 6 — chat storage, posting rules, hide/mute moderation
        ├── captcha-verifier.php            # Phase 7 — Turnstile / hCaptcha siteverify
        ├── support-service.php             # Phase 7 — subjects + tickets + notification mail
        └── cli/
            ├── seed-rooms.php              # `wp pc seed-rooms` — demo rooms for local dev
            └── machine-ingest.php          # `wp pc machine-ingest` — replay / test a machine event
```

**REST API surface (namespace `pc/v1`)**

The complete catalogue, with request / response / error shapes, lives
in `API-CONTRACT.md`. Grouped by the permission callback that gates
them:

- **Public, no auth (`__return_true`)** — `POST /google-auth/authentication`,
  `/google-auth/verify-code`, `/apple-auth/authentication`,
  `/apple-auth/verify-code`; `POST /auth/refresh` (the refresh token *is*
  the credential); `POST /user/confirm-email` (the link token is the
  credential); `GET /rooms`, `/rooms/{id}`, `/rooms/{id}/schedule`;
  `GET /support/subjects`, `POST /support/tickets` (rate-limited,
  captcha-checked for guests when configured);
  `GET /rooms/{id}/messages` (chat is readable by guests, like the room
  page it sits on);
  `POST /payments/liqpay/callback` (LiqPay signature verified in the handler).
- **Public, `UserController::check_permission`** — `POST /user/sign-up`,
  `/user/request-verification`, `/user/verify-code`. These rely on
  `Rate_Limiter` and the email code rather than a capability.
- **Logged in (`Permissions::require_logged_in`)** — `POST /auth/logout`;
  `GET/PATCH /user/me`, `POST /user/set-nickname`, `/user/accept-terms`,
  `/user/request-email-confirmation`, `/user/request-password-change`,
  `/user/confirm-password-change`; `GET /wallet`; `GET /transactions`.
- **Play-ready (`Permissions::require_play_ready` = logged in + terms
  accepted + nickname chosen + email verified)** — `POST /wallet/topup`,
  `/wallet/withdraw`; `GET /rooms/{id}/queue`, `POST /rooms/{id}/queue/join`,
  `/rooms/{id}/queue/leave`, `/rooms/{id}/play`.
- **Chat-ready (`Permissions::require_chat_ready` = logged in + terms
  accepted + nickname chosen, *without* the email-verified step)** —
  `POST /rooms/{id}/messages`. Chat moves no coins, so it is gated one
  rung below play; a nickname is required because every message renders
  with an author name.
- **Admin (`Permissions::require_admin` = logged in + `manage_options`)** —
  `GET /admin/me`; `GET/POST /admin/rooms`, `GET/PUT/DELETE /admin/rooms/{id}`,
  `PUT /admin/rooms/{id}/schedule`; `GET /admin/withdrawals`,
  `POST /admin/withdrawals/{id}/approve`, `/reject`;
  `GET/PUT /admin/coin-pricing`; `GET /admin/machine/state`,
  `POST /admin/machine/power`, `GET/PUT /admin/machine/bonus-map`;
  `GET /admin/support/tickets`, `PATCH /admin/support/tickets/{id}`,
  `GET/PUT /admin/support/subjects`, `GET/PUT /admin/support/captcha`;
  `GET /admin/chat/messages`, `PATCH /admin/chat/messages/{id}`,
  `POST /admin/chat/mute`.

Two JWTs are involved: a short-lived **access token** (HS256, 15-min
default from `pc_access_token_ttl_seconds`, signed with
`JWT_AUTH_SECRET_KEY`) and an opaque **refresh token** stored hashed in
`wp_pc_refresh_tokens`. Rotation happens on every `/auth/refresh`; reuse
detection revokes the whole descendant chain. The theme **issues** the
access JWT itself (`AuthController::issue_access_token`, using the
`Tmeister\Firebase\JWT` copy bundled with the plugin) and the
`jwt-authentication-for-wp-rest-api` plugin **validates** incoming bearer
tokens and sets the current user. The only plugin filter the theme hooks
is `jwt_auth_expire`, so a token minted through the plugin's own
`/jwt-auth/v1/token` endpoint gets the same TTL.

**Plugins (`wp-content/plugins/`)**
- `jwt-authentication-for-wp-rest-api` — provides `/wp-json/jwt-auth/v1/*` endpoints, validates incoming bearer tokens, and exposes the `jwt_auth_expire` filter the theme uses.

**Environment**
- `.ddev/config.yaml` defines a DDEV-managed local environment; `wp-config-ddev.php` is the matching WordPress config, pulled in by the git-ignored `wp-config.php`.
- `.github/workflows/main.yml` runs `php -l` over the theme on every push and PR, then (only on push to `main`, and only if lint passed) deploys the entire backend tree over FTP using repo secrets for host, user, password, and target path.

---

## Cross-cutting concerns

**Authentication & sessions**
- Username/password login is two-step: credentials → email verification code → JWT pair.
- Google sign-in flows through `GoogleAuthController` and also requires the email-code 2FA (the frontend store models `pendingGoogleAuth` / `googleIdToken`). The whole Google path is parked behind an empty `VITE_GOOGLE_CLIENT_ID` (ROADMAP Phase 1 §2); Apple is stubbed until enrollment.
- First social login lands on `/choose-nickname`, then `/accept-terms` if the stored terms version is behind `pc_terms_current_version`.
- The frontend treats the JWT as opaque. An expired access token is refreshed transparently; only a failed refresh signs the user out.
- Custom `player` role is added at `init` with a `play` capability, and administrators are given `play` for parity. **No code checks `play` today** — every gate goes through `Permissions::*`, which checks login state, user meta, and `manage_options`. The role is effectively a label.

**Configuration / environments**
- Local: DDEV (`https://pusher-coin.ddev.site`) drives the backend; both SPAs' `.env` files point at it.
- Production: backend is deployed via FTP from `main`; the player SPA builds via Vite and deploys to Vercel using `.env.production`. The admin SPA does **not** have a deploy target yet — it's local-only.
- Operator-tunable settings are WP options (coin pricing, machine entity IDs, bonus map, support address, captcha provider / site key, queue idle timeout). Every option, table, and meta key is catalogued in `DATA-MODEL.md`; `Install_Schema` seeds the defaults.

**Wallet & payments (Phase 4)**

The economic surface is split across three custom tables (`wp_pc_wallets`,
`wp_pc_coin_lots`, `wp_pc_transactions`) and a single
`Wallet_Service` class that owns all atomic mutations under
`SELECT … FOR UPDATE`. Coins are stored as a FIFO stack of
`(qty, unit_price)` lots so a winning coin pays back at the same price
it was bought at. `debit_fifo` is consumed by withdrawals and by the
Phase 6 play endpoint (one coin per toss).

Top-ups flow through **LiqPay** Checkout: the SPA calls
`POST /wallet/topup`, gets a signed `{ data, signature }` envelope,
form-POSTs it to `liqpay.ua/api/3/checkout`, and on settlement LiqPay
calls back `POST /payments/liqpay/callback`. The webhook handler is
the only place that flips a transaction `pending → completed`,
inserts the coin lot, and credits the wallet — atomic and idempotent
on `(order_id, status)`. The merchant's **private key lives in
`wp-config.php`** (`PC_LIQPAY_PRIVATE_KEY`) and never touches the DB.

Withdrawals are manual: `POST /wallet/withdraw` FIFO-debits the lots
into a `pending` transaction (consumed slices preserved on
`consumed_lots` for refunds; one pending withdrawal per player at a
time). An admin then approves (out-of-band payout, mark `completed`)
or rejects (re-credit at original prices, mark `refunded`) from the
admin SPA's withdrawals queue.

The currency is UAH; money columns are `DECIMAL(12,2)` and all
serialisation goes through decimal strings to avoid JS float drift.
Machine payouts credit coin lots directly and are audited in
`wp_pc_machine_events`, not in the ledger — the player's history view
shows money movements only.

**Physical machine (Phase 5)**

`Machine_Service` is the only code that talks to Home Assistant. It
reads the bearer token from the `PC_MACHINE_TOKEN` wp-config constant
(never the DB), the base URL and entity IDs from `pc_machine_*` options
with defaults matching `PUSHER-COIN-COMMANDS.txt`, uses a 2-second HTTP
timeout, and returns typed `WP_Error`s (`machine_not_configured`,
`machine_offline`, `machine_unauthorized`, `machine_call_failed`,
`machine_unavailable_state`). Callers map those to gateway statuses
(502 / 503) so a machine fault never looks like an auth failure to the
SPAs' 401 interceptors. Surface: power on/off, `toss_coin()` (enforces
HTTP 200), sensor reads (coin count, bonus number, light bitfield,
relay state), relay open/close, a soft-failing batched snapshot for the
admin view, and `is_online()`.

Inbound events are modelled but have no transport yet.
`Machine_Event_Log` writes `wp_pc_machine_events` (idempotent on
`event_key`); `Machine_Ingest_Service` turns a bonus / relay-closed /
coins-dropped event into a wallet credit, resolving the player through
the `pc_machine_event_player` filter and announcing the credit through
`pc_machine_event_credited`. The only caller today is the
`wp pc machine-ingest` CLI. Whether Home Assistant can push (webhook) or
must be polled is ROADMAP Phase 5 Step 6 — the open critical path — and
the push channel to the SPA is Step 7.

**Game loop: queue, turns, settlement (Phase 6)**

`Queue_Service` owns `wp_pc_room_queues` and `wp_pc_bet_sessions`. A
player joins a room's queue declaring how many coins they intend to
play; FIFO order by `joined_at`; the head of the queue holds an open
bet session (`ended_at IS NULL`, at most one per room). The SPA polls
`GET /rooms/{id}/queue` every 3s and that poll is the heartbeat —
entries whose `last_seen_at` is older than `pc_queue_idle_timeout_seconds`
are pruned on the next read, so the queue heals on traffic alone with
no cron.

`POST /rooms/{id}/play` is the one place a coin is spent. In order: refuse
if the caller is not at the head; refuse with 423 `relay_closed` while
the machine is mid-payout; FIFO-debit one coin; call
`Machine_Service::toss_coin()`; if the machine does not answer 200,
re-credit the exact lot price. A successful toss increments
`coins_played` on the session.

Settlement is the other direction: `Queue_Service` hooks
`pc_machine_event_player` (machine id → room via `pc_room_machine_id`
→ open session → player) and `pc_machine_event_credited` (bumps
`coins_won` / `money_won` on the session, which `UserControls` shows as
per-turn winnings). This wiring is complete but idle until machine
events have a transport.

**Support & captcha (Phase 7)**

`SupportController` exposes the subject list (`pc_support_subject` CPT,
ordered by `menu_order`) and accepts tickets into `wp_pc_support_tickets`
via `Support_Service`: rate-limited per IP, audited, stored with IP /
UA / the `email_verified` state at submission time, and mailed to
`pc_support_email` with the player as `Reply-To`. Guests must pass a
captcha **only if one is configured** — `Captcha_Verifier` calls the
Turnstile or hCaptcha siteverify endpoint using the `PC_CAPTCHA_SECRET`
wp-config constant and the `pc_captcha_site_key` option; when either is
empty the guest path runs unchallenged, and the admin SPA's captcha
panel says so in red. That unconfigured state is a launch blocker
(ROADMAP Phase 7 §1, `CAPTCHA_SETUP.md`). Admins triage tickets and edit
subjects through `AdminSupportController`; retiring a subject trashes
the post so old tickets still resolve their label.

**In-room chat (Phase 6)**

`Chat_Service` owns `wp_pc_room_messages`. Reads are public and
cursor-based — `GET /rooms/{id}/messages?after=<last id>`, polled every
3s by the same store that owns the chat panel's open/closed state — so a
guest watching a broadcast sees the conversation read-only, exactly as
the component was built for in Phase 3. Writes go through
`require_chat_ready`, are capped at 500 sanitised plain-text characters,
and are rate-limited to 10 per minute per account.

Chat deliberately does not require the room to be `available` the way
the queue does: a room in maintenance is where players ask what is going
on.

Moderation is two verbs. Hiding flips a `status` column rather than
deleting the row, so the author, body, IP, and timestamp survive for
whoever reviews the complaint; muting writes a `chat_muted_until`
timestamp to user meta and is account-wide, enforced server-side on
every post. Both run from the admin SPA's `ChatView` and are audited.

**Live streaming**

`frontend/src/components/LiveStream.vue` sniffs the room's
`stream_url` to pick a transport: iframe for embed-style URLs
(YouTube / Vimeo / `/embed/...`), `<video>` for direct video files,
**Mux LL-HLS via `hls.js`** for `.m3u8` URLs (the production path).
`hls.js` is dynamically imported so it ships as its own ~162KB gzipped
chunk, only loaded when an HLS stream is about to play. Safari skips
the library and uses its native HLS support directly. RTMP ingest from
the venue → Mux → LL-HLS playback URL persisted in the room's
`pc_room_stream_url` post meta.

**Trust boundaries**
- The browser is untrusted; only the JWT travels back to identify the user. Balance rules the SPA enforces (coin-quantity clamps, zero-balance routing) are re-checked server-side.
- Every REST route names an explicit `permission_callback`. Twelve are public by design (listed above) and rely on a bearer credential in the body, a payment-provider signature, a rate limit, or a captcha instead of a session. Hardening those — nonces, captcha coverage, tighter limits — is the ROADMAP Phase 8 security-review item.
- The backend is the only party that holds machine and payment credentials; the SPAs never see the HA endpoint, the LiqPay private key, or the captcha secret.
- Secrets live outside the repo and outside the DB, as wp-config constants on the server: `JWT_AUTH_SECRET_KEY`, `GOOGLE_CLIENT_ID` (audience for ID-token checks), `APPLE_CLIENT_ID` / `APPLE_TEAM_ID` / `APPLE_KEY_ID` / `APPLE_PRIVATE_KEY`, `PC_LIQPAY_PRIVATE_KEY`, `PC_MACHINE_TOKEN`, `PC_CAPTCHA_SECRET`. FTP credentials live in GitHub Actions secrets. Rotation for any of them is a wp-config edit. Their public counterparts (LiqPay public key, captcha site key) are WP options set from the admin SPA.
