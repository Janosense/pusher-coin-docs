# Pusher Coin — Architecture

Pusher Coin is a real-time, browser-based gambling/slot-machine application split into two independent applications that live side by side in this repository:

- `frontend/` — a Vue 3 single-page application (SPA) that the player interacts with.
- `backend/` — a WordPress installation that exposes a JSON REST API used by the frontend (the WordPress admin/HTML side is not the user-facing product).

The two apps are decoupled: they have separate `.git` repositories, separate deploy pipelines, and communicate exclusively over HTTPS/JSON.

---

## High-level shape

```
┌──────────────────────────┐         HTTPS / JSON          ┌─────────────────────────────┐
│  Browser (Vue 3 SPA)     │  ───────────────────────────▶ │  WordPress (DDEV / FTP)     │
│  Vite build, deployed    │  ◀───────────────────────────  │  Custom `pc` theme exposes  │
│  to Vercel               │     JWT bearer auth             │  /wp-json/pc/v1 endpoints   │
└──────────────────────────┘                                └─────────────────────────────┘
                                                                       │
                                                                       │ JWT plugin
                                                                       ▼
                                                            /wp-json/jwt-auth/v1
```

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
    │   ├── UserController.php          # POST sign-up, request-verification, verify-code (issues JWT)
    │   └── GoogleAuthController.php    # Google OAuth token exchange / 2FA
    ├── utils.php
    └── utils/
        └── role-player.php  # Registers a custom `player` role with a `play` capability
```

**REST API surface (namespace `pc/v1`)**

Registered in `UserController::register_routes()`:

| Method | Route | Purpose |
| --- | --- | --- |
| `POST` | `/pc/v1/user/sign-up/` | Create a new user (email/nickname/password, optional phone). Assigns the custom `player` role. |
| `POST` | `/pc/v1/user/request-verification/` | Authenticate credentials and email a 6-digit code (15-minute TTL stored in user meta). |
| `POST` | `/pc/v1/user/verify-code/` | Validate the code and return a signed JWT (HS256, 7-day expiry, signed with `JWT_AUTH_SECRET_KEY`). |

`GoogleAuthController` registers parallel routes for Google sign-in / linking. The JWT itself is issued using the `firebase/php-jwt` library bundled via Composer and is compatible with the `jwt-authentication-for-wp-rest-api` plugin's filters (`jwt_auth_expire`, `jwt_auth_token_before_sign`, `jwt_auth_token_before_dispatch`).

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
- Local: DDEV (`https://pusher-coin.ddev.site`) drives the backend; the frontend's `.env` points at it.
- Production: backend is deployed via FTP from `main`; frontend builds via Vite and deploys to Vercel using `.env.production` values (separate `VITE_API_BASE_URL`, `VITE_JWT_BASE_URL`, `VITE_GOOGLE_CLIENT_ID`).

**Trust boundaries**
- The browser is untrusted; only the JWT travels back to identify the user.
- The WordPress REST controllers validate every request; they currently use `check_permission` returning `true` for the public sign-up/login routes (token-protected routes are gated by the JWT plugin).
- Secrets (`JWT_AUTH_SECRET_KEY`, Google client secret, FTP credentials) live outside the repo — in `wp-config.php` on the server and in GitHub Actions secrets respectively.
