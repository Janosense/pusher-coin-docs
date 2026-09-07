# Project Tree

WordPress core directories (`wp-admin/`, `wp-includes/`) and the standard WordPress
top-level files are listed only at the top level of `backend/` and not expanded.
Project-owned code under `wp-content/` (the custom `pc` theme) is fully expanded.

Git-ignored artefacts are omitted: `backend/wp-config.php`,
`backend/wp-content/themes/pc/composer.lock`, `backend/wp-content/uploads/`,
and every `node_modules/` / `dist/`. The root repo ignores `backend/`,
`frontend/`, and `admin/` wholesale — each is its own git repo — so the root
repo tracks only the documentation files listed first.

```
pusher-coin/
├── .claude/
│   └── settings.local.json                    # Claude Code tool permissions for this checkout (local, untracked)
├── .gitignore                                 # Ignores backend/, frontend/, admin/ (separate repos) + IDE noise
├── ADMIN-DECISION.md                          # Admin surfaces are a separate Vue SPA, not WP admin
├── API-CONTRACT.md                            # Canonical pc/v1 request / response / error shapes
├── ARCHITECTURE.md
├── CLAUDE.md                                  # Working conventions + documentation-upkeep rules
├── DATA-MODEL.md                              # Storage decision per entity (table / CPT / meta / option)
├── INVENTORY.md                               # Phase 0 audit + per-phase stub resolution log
├── PROJECT-TREE.md
├── PUSHER-COIN-COMMANDS.txt                   # Home Assistant API for the physical machine
├── ROADMAP.md                                 # Phased plan; doubles as the project status board
│
├── backend/                                   # WordPress installation (separate git repo)
│   ├── .ddev/
│   │   └── config.yaml
│   ├── .github/
│   │   └── workflows/
│   │       └── main.yml                       # `php -l` over the theme on every push/PR; FTP deploy needs lint and runs only on push to main
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
│   │       │   ├── CAPTCHA_SETUP.md            # Phase 7: Turnstile / hCaptcha key setup + rotation
│   │       │   ├── GOOGLE_AUTH_SETUP.md
│   │       │   ├── app/
│   │       │   │   ├── rest-api.php           # Wires controllers into rest_api_init
│   │       │   │   ├── rest-api/
│   │       │   │   │   ├── AdminCoinPricingController.php # Phase 4: /admin/coin-pricing read/write
│   │       │   │   │   ├── AdminController.php          # Phase 3: /admin/me probe
│   │       │   │   │   ├── AdminMachineController.php   # Phase 5: /admin/machine power + state + bonus-map
│   │       │   │   │   ├── AdminRoomController.php      # Phase 3: /admin/rooms CRUD + schedule replace
│   │       │   │   │   ├── AdminSupportController.php   # Phase 7: /admin/support tickets + subjects
│   │       │   │   │   ├── AdminWithdrawalController.php # Phase 4: /admin/withdrawals queue + approve/reject
│   │       │   │   │   ├── AppleAuthController.php   # Apple Sign-In (stub until enrolled)
│   │       │   │   │   ├── AuthController.php       # /auth/logout, /auth/refresh + token-pair helpers
│   │       │   │   │   ├── GoogleAuthController.php
│   │       │   │   │   ├── PaymentController.php    # Phase 4: LiqPay webhook
│   │       │   │   │   ├── RoomController.php       # Phase 3: public /rooms read endpoints
│   │       │   │   │   ├── RoomQueueController.php  # Phase 6: queue join/leave + play (toss)
│   │       │   │   │   ├── SupportController.php    # Phase 7: public /support subjects + tickets
│   │       │   │   │   ├── TransactionsController.php # Phase 4: GET /transactions
│   │       │   │   │   ├── UserController.php
│   │       │   │   │   └── WalletController.php     # Phase 4: GET /wallet + POST /wallet/topup + /withdraw
│   │       │   │   ├── utils.php
│   │       │   │   └── utils/
│   │       │   │       ├── audit-log.php       # Audit_Log writer
│   │       │   │       ├── captcha-verifier.php # Phase 7: Turnstile / hCaptcha siteverify
│   │       │   │       ├── cli/
│   │       │   │       │   ├── machine-ingest.php # `wp pc machine-ingest` — replay a machine event (Phase 5)
│   │       │   │       │   └── seed-rooms.php  # `wp pc seed-rooms` (Phase 3)
│   │       │   │       ├── cpt-room.php        # Registers pc_room CPT (Phase 3)
│   │       │   │       ├── cpt-support-subject.php # Registers pc_support_subject CPT (Phase 7)
│   │       │   │       ├── install-schema.php  # Custom-table installer
│   │       │   │       ├── liqpay-client.php   # Phase 4: LiqPay sign/verify/decode helper
│   │       │   │       ├── machine-events.php  # Phase 5: Machine_Event_Log writer (wp_pc_machine_events)
│   │       │   │       ├── machine-ingest-service.php # Phase 5: machine event → wallet credit (transport-agnostic)
│   │       │   │       ├── machine-service.php # Phase 5: Home Assistant REST wrapper
│   │       │   │       ├── permissions.php     # Permission_callback helpers
│   │       │   │       ├── post-meta-keys.php  # Post_Meta_Keys registry
│   │       │   │       ├── queue-service.php   # Phase 6: queue, turns, bet sessions, machine-event attribution
│   │       │   │       ├── rate-limiter.php    # Transient-based rate limiter
│   │       │   │       ├── refresh-tokens.php  # Refresh-token issuance / rotation
│   │       │   │       ├── role-player.php     # Registers `player` role
│   │       │   │       ├── room-schedule-calculator.php  # Computes current/next broadcast windows
│   │       │   │       ├── support-service.php # Phase 7: subjects CPT + tickets table + notify mail
│   │       │   │       ├── user-meta-keys.php  # User_Meta_Keys registry
│   │       │   │       └── wallet-service.php # Phase 4: atomic wallet / lot / transaction ops
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
    ├── .env                                   # Local (DDEV) API endpoints; Google client ID parked (empty)
    ├── .env.production                        # Production endpoints; Google client ID parked (empty)
    ├── .eslintrc.cjs
    ├── .github/
    │   └── workflows/
    │       └── ci.yml                         # Phase 0 CI: `npm ci` + lint + build on every push/PR
    ├── .gitignore
    ├── .prettierrc.json
    ├── CLAUDE.md
    ├── README.md
    ├── index.html                             # Vite entry HTML; GIS <script> commented out while Google is parked
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
    │   │   ├── AppNavigation.vue               # Was Navigation.vue
    │   │   ├── AppleSignInButton.vue          # Apple Sign-In (renders only when configured)
    │   │   ├── FacelessAvatar.vue              # Deterministic SVG identicon (Phase 2)
    │   │   ├── GoogleSignInButton.vue          # Hidden while VITE_GOOGLE_CLIENT_ID is empty (parked)
    │   │   ├── HelloWorld.vue                  # Vite scaffold; no importers (dead file)
    │   │   ├── LanguageSwitcher.vue            # Rendered in AppNavigation; i18n itself is Phase 8
    │   │   ├── LiveStream.vue                  # Transport-agnostic stream container (Phase 3)
    │   │   ├── LogoutConfirmModal.vue          # Confirm-before-logout overlay
    │   │   ├── ModalOverlay.vue                # Was Overlay.vue
    │   │   ├── NavigationToggle.vue
    │   │   ├── NextBroadcastCountdown.vue      # 1Hz local countdown to next room window (Phase 3)
    │   │   ├── PlaceBet.vue                    # Phase 6: declare coins / wait turn / toss
    │   │   ├── ReplenishmentBalance.vue
    │   │   ├── RoomChat.vue                    # Was Chat.vue; still placeholder messages
    │   │   ├── RoomList.vue                    # Was Rooms.vue
    │   │   ├── RoomQueue.vue                   # Was Queue.vue; Phase 6: live queue + turn highlight
    │   │   ├── RoomStatusBadge.vue             # Available / maintenance / unavailable chip (Phase 3)
    │   │   ├── SignInForm.vue
    │   │   ├── SignUpForm.vue
    │   │   ├── UserControls.vue
    │   │   ├── WithdrawalRequest.vue           # Phase 4: player coin withdrawal form (overlay body)
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
    │   │   ├── historyService.js              # Phase 4: GET /transactions
    │   │   ├── liqpayCheckout.js              # Phase 4: builds + submits the LiqPay hosted-checkout form POST
    │   │   ├── queueService.js                # Phase 6: queue + play endpoints
    │   │   ├── roomsService.js                # /rooms read endpoints (Phase 3)
    │   │   ├── sessionService.js              # Inactivity timer
    │   │   ├── supportService.js              # Phase 7: /support subjects + tickets
    │   │   ├── userService.js
    │   │   └── walletService.js               # Phase 4: /wallet + /wallet/topup + /withdraw
    │   ├── stores/
    │   │   ├── authentication.js              # Token, user, Google 2FA state
    │   │   ├── chat.js                        # Chat panel open/closed toggle only; messages are local placeholders in RoomChat.vue
    │   │   ├── counter.js                     # Vite scaffold; no importers (dead file)
    │   │   ├── navigation.js
    │   │   ├── queue.js                       # Phase 6: queue state, 3s poll + heartbeat
    │   │   ├── rooms.js                       # Rooms list + 30s cache (Phase 3)
    │   │   ├── user.js                        # Empty file; no importers (dead file)
    │   │   └── wallet.js                      # Wallet balance, lots, pricing, topup action (Phase 4)
    │   └── views/
    │       ├── AboutView.vue                  # Not in the route table; unreachable
    │       ├── AcceptTermsView.vue            # Phase 1 gate view
    │       ├── AccountView.vue                # Data-driven account surface (Phase 2)
    │       ├── ChooseNicknameView.vue         # Phase 1 gate view (after first social login)
    │       ├── ConfirmEmailView.vue           # Email-confirmation landing page (Phase 2)
    │       ├── HistoryView.vue
    │       ├── RoomView.vue
    │       ├── RoomsView.vue
    │       ├── SignInView.vue
    │       ├── SignUpView.vue
    │       └── SupportView.vue                # Phase 7: subject dropdown, guest captcha, verified-email gate
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
        │   ├── adminCoinPricingService.js     # Phase 4: GET/PUT /admin/coin-pricing
        │   ├── adminMachineService.js         # Phase 5: /admin/machine state + power + bonus-map read/write
        │   ├── adminRoomsService.js           # Wraps /admin/rooms CRUD + schedule replace
        │   ├── adminSupportService.js         # Phase 7: /admin/support tickets + subjects
        │   ├── adminWithdrawalsService.js     # Phase 4: /admin/withdrawals queue + approve/reject
        │   └── api.js                         # Bearer + 401-refresh axios instance (admin-keyed localStorage)
        ├── stores/
        │   ├── auth.js                        # Two-step sign-in + /admin/me gate
        │   ├── rooms.js                       # Admin rooms CRUD + schedule
        │   └── withdrawals.js                 # Phase 4: queue + approve/reject
        └── views/
            ├── MachineView.vue                # Phase 5: connection probe + power toggle + sensor grid (3s poll)
            ├── RoomFormView.vue               # Create / edit room (shared)
            ├── RoomListView.vue               # Table + create / edit / schedule / trash actions
            ├── RoomScheduleView.vue           # Weekly rules editor (atomic replace)
            ├── SettingsView.vue               # Phase 4: coin pricing form + LiqPay credential hints; Phase 5: bonus-map grid + relay coin count
            ├── SignInView.vue                 # Email/password + 6-digit code form
            ├── SubjectsView.vue               # Phase 7: subject list editor + guest-captcha config panel
            ├── TicketsView.vue                # Phase 7: ticket queue with status filter, search, mailto reply
            └── WithdrawalsView.vue            # Phase 4: queue with filter tabs + approve/reject dialog
```
