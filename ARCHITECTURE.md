# Pusher Coin — Architecture

Pusher Coin is a real-time, browser-based gambling/slot-machine application split into three independent applications that live side by side in this repository:

- `frontend/` — a Vue 3 single-page application (SPA) that the player interacts with.
- `admin/` — a separate Vue 3 SPA used by operators to manage rooms, schedules, and (in later phases) machine state and support tickets. Introduced in Phase 3. See `ADMIN-DECISION.md` for the rationale.
- `backend/` — a WordPress installation that exposes a JSON REST API used by both SPAs (the WordPress admin/HTML side is not the user-facing product).

The three apps are decoupled: separate `.git` repositories, separate deploy pipelines, and communication exclusively over HTTPS/JSON.

---

## High-level shape

```
┌──────────────────────────┐
│  Player SPA (frontend/)  │ ─┐
└──────────────────────────┘  │      HTTPS / JSON          ┌─────────────────────────────┐
                              ├───────────────────────────▶│  WordPress (DDEV / FTP)     │
┌──────────────────────────┐  │ ◀──────────────────────────│  Custom `pc` theme exposes  │
│  Admin SPA (admin/)      │ ─┘    JWT bearer auth         │  /wp-json/pc/v1 endpoints   │
└──────────────────────────┘                               └─────────────────────────────┘
                                                                       │
                                                                       │ JWT plugin
                                                                       ▼
                                                            /wp-json/jwt-auth/v1
```

Both SPAs share the same auth primitives (JWT pair, refresh rotation,
2FA-by-email-code) but live behind distinct localStorage keys
(`pusher_coin_*` for the player, `pc_admin_*` for the admin). The admin
SPA additionally probes `GET /pc/v1/admin/me` after sign-in to bounce
non-admins.

Data flow at a glance:

1. The SPA boots from `frontend/src/main.js`, mounts `App.vue`, installs Pinia + Vue Router.
2. The router (`src/router/index.js`) decides which `views/*.vue` to render and gates protected routes against the auth store.
3. View components compose `components/*.vue` and call `services/*.js` (an Axios client) which appends the JWT bearer token from `localStorage` to every request.
4. Requests hit the WordPress REST API at `https://pusher-coin.ddev.site/wp-json/pc/v1` (configured per environment via `VITE_API_BASE_URL`).
5. Inside WordPress, the custom theme `pc` registers REST controllers that handle sign-up, email-verification 2FA, Google OAuth login, and JWT issuance (delegating to the `jwt-authentication-for-wp-rest-api` plugin where appropriate).

---

## Frontend (`frontend/`)

A Vue 3 + Vite SPA. Uses the Composition API throughout.

**Stack**
- Vue 3.5 with `<script setup>` Single-File Components.
- Vite 5 as dev server / bundler.
- Vue Router 4 for client-side routing.
- Pinia 3 for state management.
- Axios for HTTP.
- IMask for input masking (phone, codes, etc.).
- ESLint + Prettier for code quality.
- Deployed via `vercel.json`; environment split between `.env` (local DDEV) and `.env.production`.

**Layout (`src/`)**

| Folder | Responsibility |
| --- | --- |
| `main.js` | App entry point. Creates the Vue app, registers Pinia and the router, mounts `#app`. |
| `App.vue` | Root layout: header + `NavigationToggle`, side `Navigation`, `<RouterView />`, footer. |
| `router/index.js` | Route table and `beforeEach` guard. Routes carry `meta.requiresAuth` / `meta.requiresGuest`; the guard awaits `authStore.initializeAuth()` before deciding. |
| `views/` | Page-level components mapped 1:1 to routes (`RoomsView`, `RoomView`, `SignInView`, `SignUpView`, `AccountView`, `HistoryView`, `SupportView`, `AboutView`). |
| `components/` | Reusable building blocks: `Navigation`, `Chat`, `Rooms`, `Queue`, `PlaceBet`, `ReplenishmentBalance`, `SignInForm`, `SignUpForm`, `GoogleSignInButton`, `UserControls`, `LanguageSwitcher`, `Overlay`, plus an `icons/` set of single-purpose SVG components. |
| `stores/` | Pinia stores. `authentication.js` is the central one (token + user, persisted to `localStorage`, with Google 2FA state). `chat.js`, `navigation.js`, `counter.js`, `user.js` hold UI/feature state. |
| `services/` | API layer. `api.js` is a configured Axios instance with request/response interceptors (auto-attaches JWT, broadcasts a `auth:token-expired` event on 401). `authService.js`, `googleAuthService.js`, `userService.js` wrap concrete endpoints. |
| `assets/` | Global CSS (`main.css`, `styles/colors.css`, block-scoped CSS in `styles/blocks/`), images, the brand SVG logo. |
| `public/` | Static files served verbatim by Vite (`favicon.ico`). |

**Authentication flow (frontend side)**
- Token + user are stored under `pusher_coin_auth_token` / `pusher_coin_user_data` in `localStorage`.
- The auth store exposes `initializeAuth()` (called once by the router guard), `isAuthenticated`, plus actions for password login, sign-up, email-code 2FA, and Google OAuth.
- The Axios response interceptor clears local storage and dispatches a `CustomEvent('auth:token-expired')` whenever the API returns 401, allowing the store to react and redirect to sign-in.

**Path aliases**
- `@` → `src/` (configured in `vite.config.js` and `jsconfig.json`).

---

## Admin SPA (`admin/`)

Introduced in Phase 3. Same stack as `frontend/` (Vue 3 + Vite + Pinia
+ Vue Router + Axios + ESLint/Prettier). Runs on port 5174 in dev so
both SPAs can run side-by-side. Does **not** include IMask (no masked
inputs yet).

**Auth.** Reuses the player 2FA flow (`/user/request-verification` →
`/user/verify-code`) to issue a JWT pair, then probes
`GET /pc/v1/admin/me` to confirm `manage_options`. Non-admins are
signed out server-side (refresh-token revoked) before the SPA shows the
error. Distinct localStorage keys (`pc_admin_auth_token`,
`pc_admin_user_data`) so the two SPAs can coexist on the same origin
during dev.

**Views shipped (Phase 3 step 5c)**
- `SignInView` — two-step email/password + 6-digit code form.
- `RoomListView` — table with status badge + per-row Edit / Schedule /
  Trash actions.
- `RoomFormView` — shared create / edit form. Discriminates on
  `route.name`.
- `RoomScheduleView` — weekly rules editor. Save calls
  `PUT /admin/rooms/{id}/schedule` which replaces the rule set in a
  single transaction; the response returns the freshly-recomputed
  `next_window`.

**Deferred.** No `vercel.json` yet (no deploy target chosen); no shared
component package with `frontend/` (`ADMIN-DECISION.md` accepts some
duplication for now); no inactivity timer (planned, tighter timeout
than the player SPA's 15 minutes).

---

## Backend (`backend/`)

A standard WordPress installation. The product code lives only in two places; everything else is unmodified WordPress core or third-party plugins.

**Custom theme: `wp-content/themes/pc/`**

This is *the* application code on the backend — WordPress acts as an authentication and user-data service rather than a CMS.

```
themes/pc/
├── functions.php            # Bootstrap: composer autoload, utils, REST API registration
├── style.css                # Theme metadata header
├── index.php                # Empty/placeholder (no front-end rendering)
├── composer.json            # PHP dependencies (autoloader is required by functions.php)
├── GOOGLE_AUTH_SETUP.md     # Operator notes for configuring Google OAuth
└── app/
    ├── rest-api.php         # Wires controllers into `rest_api_init`
    ├── rest-api/
    │   ├── UserController.php          # Phase 1+2 — sign-up, 2FA, /user/me, email/password change
    │   ├── AuthController.php          # Phase 1 — /auth/logout, /auth/refresh, token-pair helpers
    │   ├── GoogleAuthController.php    # Google OAuth token exchange / 2FA
    │   ├── AppleAuthController.php     # Apple Sign-In (stub until enrollment)
    │   ├── RoomController.php          # Phase 3 — public /rooms read endpoints
    │   ├── AdminController.php         # Phase 3 — /admin/me capability probe
    │   └── AdminRoomController.php     # Phase 3 — admin /rooms CRUD + schedule replace
    ├── utils.php
    └── utils/
        ├── role-player.php             # Registers a custom `player` role with a `play` capability
        ├── user-meta-keys.php          # User_Meta_Keys registry
        ├── post-meta-keys.php          # Post_Meta_Keys registry (Phase 3 — pc_room meta)
        ├── permissions.php             # Permission_callback helpers (logged-in / play-ready / admin)
        ├── install-schema.php          # Custom-table installer (pc_db_version tracking)
        ├── audit-log.php               # Audit_Log writer
        ├── rate-limiter.php            # Transient-based rate limiter
        ├── refresh-tokens.php          # Refresh-token issuance / rotation
        ├── cpt-room.php                # Registers pc_room CPT (Phase 3)
        ├── room-schedule-calculator.php# Computes current_window / next_window
        └── cli/
            └── seed-rooms.php          # `wp pc seed-rooms` — demo data for local dev
```

**REST API surface (namespace `pc/v1`)**

The complete catalogue lives in `API-CONTRACT.md`. At a glance the
namespace currently spans:

- **Auth** — `POST /user/sign-up`, `/user/request-verification`,
  `/user/verify-code`; `POST /google-auth/*`, `/apple-auth/*` (stub);
  `POST /auth/logout`, `/auth/refresh`.
- **Account (Phase 2)** — `GET/PATCH /user/me`,
  `POST /user/request-email-confirmation`, `/user/confirm-email`,
  `/user/request-password-change`, `/user/confirm-password-change`,
  plus `POST /user/accept-terms` and `/user/set-nickname`.
- **Rooms (Phase 3, public)** — `GET /rooms`, `GET /rooms/{id}`,
  `GET /rooms/{id}/schedule`.
- **Admin (Phase 3)** — `GET /admin/me`; `GET/POST /admin/rooms`,
  `GET/PUT/DELETE /admin/rooms/{id}`,
  `PUT /admin/rooms/{id}/schedule` (atomic replace in a transaction).

Two JWTs are involved: a short-lived **access token** (HS256, 15-min
default, signed with `JWT_AUTH_SECRET_KEY`) and an opaque **refresh
token** stored hashed in `wp_pc_refresh_tokens`. Rotation happens on
every `/auth/refresh`; reuse detection revokes the whole descendant
chain. The access JWT is issued via `firebase/php-jwt` and validated by
the `jwt-authentication-for-wp-rest-api` plugin's filters
(`jwt_auth_expire`, `jwt_auth_token_before_sign`,
`jwt_auth_token_before_dispatch`).

**Plugins (`wp-content/plugins/`)**
- `jwt-authentication-for-wp-rest-api` — provides `/wp-json/jwt-auth/v1/*` endpoints, validates incoming bearer tokens, and exposes the filters the custom theme leans on.

**Environment**
- `.ddev/config.yaml` defines a DDEV-managed local environment; `wp-config-ddev.php` is the matching WordPress config.
- `.github/workflows/main.yml` deploys the entire backend tree over FTP whenever `main` is updated, using repo secrets for host, user, password, and target path.

---

## Cross-cutting concerns

**Authentication & sessions**
- Username/password login is two-step: credentials → email verification code → JWT.
- Google sign-in flows through `GoogleAuthController` and may also require 2FA (the frontend store models `pendingGoogleAuth` / `googleIdToken`).
- The frontend treats the JWT as opaque. Token expiry forces a redirect to `/sign-in` via the 401 interceptor.
- Custom `player` role is added at `init`; administrators get a `play` capability for parity.

**Configuration / environments**
- Local: DDEV (`https://pusher-coin.ddev.site`) drives the backend; both SPAs' `.env` files point at it.
- Production: backend is deployed via FTP from `main`; the player SPA builds via Vite and deploys to Vercel using `.env.production`. The admin SPA does **not** have a deploy target yet — it's local-only as of Phase 3.

**Wallet & payments (Phase 4)**

The economic surface is split across three custom tables (`wp_pc_wallets`,
`wp_pc_coin_lots`, `wp_pc_transactions`) and a single
`Wallet_Service` class that owns all atomic mutations. Coins are stored
as a FIFO stack of `(qty, unit_price)` lots so a winning coin pays
back at the same price it was bought at (Phase 6 will be the first
consumer of `debit_fifo`; Phase 4 uses it for withdrawals).

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
`consumed_lots` for refunds). An admin then approves (out-of-band
payout, mark `completed`) or rejects (re-credit at original prices,
mark `refunded`) from the admin SPA's withdrawals queue.

The currency is UAH; money columns are `DECIMAL(12,2)` and all
serialisation goes through decimal strings to avoid JS float drift.

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
- The browser is untrusted; only the JWT travels back to identify the user.
- The WordPress REST controllers validate every request; they currently use `check_permission` returning `true` for the public sign-up/login routes (token-protected routes are gated by the JWT plugin).
- Secrets (`JWT_AUTH_SECRET_KEY`, Google client secret, FTP credentials) live outside the repo — in `wp-config.php` on the server and in GitHub Actions secrets respectively.
