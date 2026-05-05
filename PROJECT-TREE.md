# Project Tree

WordPress core directories (`wp-admin/`, `wp-includes/`) and the standard WordPress
top-level files are listed only at the top level of `backend/` and not expanded.
Project-owned code under `wp-content/` (the custom `pc` theme) is fully expanded.

```
pusher-coin/
├── ARCHITECTURE.md
├── PROJECT-TREE.md
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
│   │       │   │   │   ├── GoogleAuthController.php
│   │       │   │   │   └── UserController.php
│   │       │   │   ├── utils.php
│   │       │   │   └── utils/
│   │       │   │       └── role-player.php    # Registers `player` role
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
└── frontend/                                  # Vue 3 + Vite SPA (separate git repo)
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
    │   │   ├── Chat.vue
    │   │   ├── GoogleSignInButton.vue
    │   │   ├── HelloWorld.vue
    │   │   ├── LanguageSwitcher.vue
    │   │   ├── Navigation.vue
    │   │   ├── NavigationToggle.vue
    │   │   ├── Overlay.vue
    │   │   ├── PlaceBet.vue
    │   │   ├── Queue.vue
    │   │   ├── ReplenishmentBalance.vue
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
    │   │   ├── api.js                         # Axios instance + JWT interceptors
    │   │   ├── authService.js
    │   │   ├── googleAuthService.js
    │   │   └── userService.js
    │   ├── stores/
    │   │   ├── authentication.js              # Token, user, Google 2FA state
    │   │   ├── chat.js
    │   │   ├── counter.js
    │   │   ├── navigation.js
    │   │   └── user.js
    │   └── views/
    │       ├── AboutView.vue
    │       ├── AccountView.vue
    │       ├── HistoryView.vue
    │       ├── RoomView.vue
    │       ├── RoomsView.vue
    │       ├── SignInView.vue
    │       ├── SignUpView.vue
    │       └── SupportView.vue
    ├── vercel.json                            # Vercel deploy config
    └── vite.config.js                         # `@` → `src/` alias, Vue plugin
```
