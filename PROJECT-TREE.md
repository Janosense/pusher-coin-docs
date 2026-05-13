# Project Tree

WordPress core directories (`wp-admin/`, `wp-includes/`) and the standard WordPress
top-level files are listed only at the top level of `backend/` and not expanded.
Project-owned code under `wp-content/` (the custom `pc` theme) is fully expanded.

```
pusher-coin/
├── ARCHITECTURE.md
├── PROJECT-TREE.md
├── PUSHER-COIN-COMMANDS.txt                   # Home Assistant API for the physical machine
│
├── backend/                                   # WordPress installation (separate git repo)
│   ├── .ddev/
│   │   └── config.yaml
│   ├── .github/
│   │   └── workflows/
│   │       └── main.yml                       # FTP deploy on push to main
│   ├── .gitignore
│   ├── index.php                              # WordPress
│   ├── license.txt                            # WordPress
│   ├── readme.html                            # WordPress
│   ├── wp-activate.php                        # WordPress
│   ├── wp-admin/                              # WordPress core (not expanded)
│   ├── wp-blog-header.php                     # WordPress
│   ├── wp-comments-post.php                   # WordPress
│   ├── wp-config-ddev.php                     # WordPress (DDEV-generated)
│   ├── wp-config-sample.php                   # WordPress
│   ├── wp-content/
│   │   ├── index.php
│   │   ├── plugins/
│   │   │   ├── index.php
│   │   │   └── jwt-authentication-for-wp-rest-api/
│   │   │       ├── LICENSE.txt
│   │   │       ├── admin/
│   │   │       ├── includes/
│   │   │       ├── index.php
│   │   │       ├── jwt-auth.php
│   │   │       ├── languages/
│   │   │       ├── public/
│   │   │       └── readme.txt
│   │   └── themes/
│   │       ├── index.php
│   │       ├── pc/                            # Custom application theme
│   │       │   ├── GOOGLE_AUTH_SETUP.md
│   │       │   ├── app/
│   │       │   │   ├── rest-api.php           # Wires controllers into rest_api_init
│   │       │   │   ├── rest-api/
│   │       │   │   │   ├── AdminController.php      # Phase 3: /admin/me probe
│   │       │   │   │   ├── AdminRoomController.php  # Phase 3: /admin/rooms CRUD + schedule replace
│   │       │   │   │   ├── AppleAuthController.php   # Apple Sign-In (stub until enrolled)
│   │       │   │   │   ├── AuthController.php       # /auth/logout, /auth/refresh + token-pair helpers
│   │       │   │   │   ├── GoogleAuthController.php
│   │       │   │   │   ├── RoomController.php       # Phase 3: public /rooms read endpoints
│   │       │   │   │   └── UserController.php
│   │       │   │   ├── utils.php
│   │       │   │   └── utils/
│   │       │   │       ├── audit-log.php       # Audit_Log writer
│   │       │   │       ├── cpt-room.php        # Registers pc_room CPT (Phase 3)
│   │       │   │       ├── install-schema.php  # Custom-table installer
│   │       │   │       ├── permissions.php     # Permission_callback helpers
│   │       │   │       ├── post-meta-keys.php  # Post_Meta_Keys registry
│   │       │   │       ├── rate-limiter.php    # Transient-based rate limiter
│   │       │   │       ├── refresh-tokens.php  # Refresh-token issuance / rotation
│   │       │   │       ├── role-player.php     # Registers `player` role
│   │       │   │       ├── room-schedule-calculator.php  # Computes current/next broadcast windows
│   │       │   │       └── user-meta-keys.php  # User_Meta_Keys registry
│   │       │   ├── composer.json
│   │       │   ├── functions.php              # Theme bootstrap
│   │       │   ├── index.php
│   │       │   └── style.css
│   │       ├── twentytwentythree/             # Default WP theme (not expanded)
│   │       ├── twentytwentyfour/              # Default WP theme (not expanded)
│   │       └── twentytwentyfive/              # Default WP theme (not expanded)
│   ├── wp-cron.php                            # WordPress
│   ├── wp-includes/                           # WordPress core (not expanded)
│   ├── wp-links-opml.php                      # WordPress
│   ├── wp-load.php                            # WordPress
│   ├── wp-login.php                           # WordPress
│   ├── wp-mail.php                            # WordPress
│   ├── wp-settings.php                        # WordPress
│   ├── wp-signup.php                          # WordPress
│   ├── wp-trackback.php                       # WordPress
│   └── xmlrpc.php                             # WordPress
│
├── frontend/                                  # Vue 3 + Vite SPA (separate git repo)
    ├── .env                                   # Local (DDEV) API endpoints
    ├── .env.production                        # Production endpoints
    ├── .eslintrc.cjs
    ├── .gitignore
    ├── .prettierrc.json
    ├── CLAUDE.md
    ├── README.md
    ├── index.html                             # Vite entry HTML
    ├── jsconfig.json                          # `@` → `src/` alias for editors
    ├── package.json
    ├── package-lock.json
    ├── public/
    │   └── favicon.ico
    ├── src/
    │   ├── App.vue                            # Root layout
    │   ├── main.js                            # App bootstrap (Vue + Pinia + Router)
    │   ├── assets/
    │   │   ├── images/
    │   │   │   ├── icon-coin.svg
    │   │   │   ├── iconCoin.png
    │   │   │   └── room.png
    │   │   ├── logo.svg
    │   │   ├── main.css
    │   │   └── styles/
    │   │       ├── blocks/
    │   │       │   ├── body.css
    │   │       │   ├── button.css
    │   │       │   ├── content.css
    │   │       │   ├── form.css
    │   │       │   ├── view-holder.css
    │   │       │   └── wrapper.css
    │   │       └── colors.css
    │   ├── components/
    │   │   ├── AppleSignInButton.vue          # Apple Sign-In (renders only when configured)
    │   │   ├── Chat.vue
    │   │   ├── FacelessAvatar.vue              # Deterministic SVG identicon (Phase 2)
    │   │   ├── GoogleSignInButton.vue
    │   │   ├── HelloWorld.vue
    │   │   ├── LanguageSwitcher.vue
    │   │   ├── LiveStream.vue                  # Transport-agnostic stream container (Phase 3)
    │   │   ├── LogoutConfirmModal.vue          # Confirm-before-logout overlay
    │   │   ├── Navigation.vue
    │   │   ├── NavigationToggle.vue
    │   │   ├── NextBroadcastCountdown.vue      # 1Hz local countdown to next room window (Phase 3)
    │   │   ├── Overlay.vue
    │   │   ├── PlaceBet.vue
    │   │   ├── Queue.vue
    │   │   ├── ReplenishmentBalance.vue
    │   │   ├── RoomStatusBadge.vue             # Available / maintenance / unavailable chip (Phase 3)
    │   │   ├── Rooms.vue
    │   │   ├── SignInForm.vue
    │   │   ├── SignUpForm.vue
    │   │   ├── UserControls.vue
    │   │   └── icons/
    │   │       ├── IconAccount.vue
    │   │       ├── IconChat.vue
    │   │       ├── IconClose.vue
    │   │       ├── IconCoin.vue
    │   │       ├── IconHistory.vue
    │   │       ├── IconLogIn.vue
    │   │       ├── IconLogOut.vue
    │   │       ├── IconMain.vue
    │   │       ├── IconRoomEnter.vue
    │   │       ├── IconSendMessage.vue
    │   │       ├── IconSettings.vue
    │   │       ├── IconSignUp.vue
    │   │       ├── IconSound.vue
    │   │       └── IconSupport.vue
    │   ├── router/
    │   │   └── index.js                       # Routes + auth guard
    │   ├── services/
    │   │   ├── accountService.js              # /user/me + email/password change endpoints (Phase 2)
    │   │   ├── api.js                         # Axios instance + refresh-on-401 interceptor
    │   │   ├── appleAuthService.js            # Apple Sign-In SDK wrapper
    │   │   ├── authService.js                 # /auth/* + /user/accept-terms + /user/set-nickname
    │   │   ├── googleAuthService.js
    │   │   ├── roomsService.js                # /rooms read endpoints (Phase 3)
    │   │   ├── sessionService.js              # Inactivity timer
    │   │   └── userService.js
    │   ├── stores/
    │   │   ├── authentication.js              # Token, user, Google 2FA state
    │   │   ├── chat.js
    │   │   ├── counter.js
    │   │   ├── navigation.js
    │   │   ├── rooms.js                       # Rooms list + 30s cache (Phase 3)
    │   │   └── user.js
    │   └── views/
    │       ├── AboutView.vue
    │       ├── AcceptTermsView.vue            # Phase 1 gate view
    │       ├── AccountView.vue                # Data-driven account surface (Phase 2)
    │       ├── ChooseNicknameView.vue         # Phase 1 gate view (after first social login)
    │       ├── ConfirmEmailView.vue           # Email-confirmation landing page (Phase 2)
    │       ├── HistoryView.vue
    │       ├── RoomView.vue
    │       ├── RoomsView.vue
    │       ├── SignInView.vue
    │       ├── SignUpView.vue
    │       └── SupportView.vue
    ├── vercel.json                            # Vercel deploy config
    └── vite.config.js                         # `@` → `src/` alias, Vue plugin

└── admin/                                     # Vue 3 + Vite admin SPA (Phase 3, separate deploy)
    ├── .env.example                           # VITE_API_BASE_URL → same pc/v1 backend
    ├── .eslintrc.cjs
    ├── .gitignore
    ├── .prettierrc.json
    ├── index.html
    ├── jsconfig.json
    ├── package.json
    ├── vite.config.js                         # Port 5174 so player + admin can run side-by-side
    ├── public/
    └── src/
        ├── App.vue
        ├── main.js
        ├── assets/
        │   └── main.css
        ├── components/
        │   └── AdminLayout.vue                # Header + nav + slot
        ├── router/
        │   └── index.js                       # Auth guards: requiresAuth / requiresGuest
        ├── services/
        │   ├── adminAuthService.js            # Wraps /user/verify-code + /admin/me probe
        │   ├── adminRoomsService.js           # Wraps /admin/rooms CRUD + schedule replace
        │   └── api.js                         # Bearer + 401-refresh axios instance (admin-keyed localStorage)
        ├── stores/
        │   ├── auth.js                        # Two-step sign-in + /admin/me gate
        │   └── rooms.js                       # Admin rooms CRUD + schedule
        └── views/
            ├── RoomFormView.vue               # Create / edit room (shared)
            ├── RoomListView.vue               # Table + create / edit / schedule / trash actions
            ├── RoomScheduleView.vue           # Weekly rules editor (atomic replace)
            └── SignInView.vue                 # Email/password + 6-digit code form
```
