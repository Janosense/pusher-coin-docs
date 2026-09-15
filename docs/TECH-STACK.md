# Tech stack — Pusher Coin

## Stack

<!-- Adoption mode: these versions are what is actually installed, read from the
     lockfiles and from wp-includes/version.php — not discovery guesses. Exact
     versions live in package-lock.json / composer.lock; the majors are the
     contract. -->

| Layer | Choice | Version | Why |
|---|---|---|---|
| Backend runtime | PHP | 8.4 (DDEV local; CI lints against 8.2) | WordPress host requirement |
| Backend platform | WordPress | 6.8.2 | Pre-existing; used as an auth / user-data / integration service, not a CMS |
| Backend app code | Custom theme `pc` | — | All product code lives in `backend/wp-content/themes/pc/`; `index.php` renders nothing |
| Auth plugin | `jwt-authentication-for-wp-rest-api` | 1.4.1 | Validates incoming bearer tokens and exposes `jwt_auth_expire`; the theme issues tokens itself |
| DB | MariaDB | 10.11 (DDEV) | DDEV default for this project |
| Schema tool | `dbDelta` via `Install_Schema` | — | WordPress-native; drives the `pc_db_version` option |
| PHP deps | `google/apiclient` 2.19.3, `firebase/php-jwt` 7.0.5, `guzzlehttp/guzzle` 7.10.0 (transitive) | see `composer.lock` | Google ID-token verification only |
| Player SPA | Vue 3 (`<script setup>`, Composition API) | 3.5.21 | — |
| Player SPA build | Vite | 5.4.20 | — |
| Player SPA state / routing / HTTP | Pinia 3.0.3, Vue Router 4.5.1, Axios 1.12.0 | — | — |
| Player SPA extras | `hls.js` 1.6.16 (dynamic import), `imask` 7.6.1 | — | Mux LL-HLS playback; phone / code / coin-quantity masking |
| Admin SPA | Vue 3.5.34, Vite 5.4.21, Pinia 3.0.4, Vue Router 4.6.4, Axios 1.16.0 | — | Same stack as the player SPA, without `hls.js` and `imask` |
| Lint / format | ESLint 8.57.1 + Prettier (3.6.2 frontend, 3.8.3 admin); `php -l` for the theme | — | No PHPCS, PHPStan or Psalm anywhere |
| Testing | **none** | — | No PHPUnit, no Vitest, no Playwright, in any of the three repositories |
| Node | 20 (CI) | — | `frontend/.github/workflows/ci.yml` |
| Local env | DDEV (nginx-fpm), host `https://pusher-coin.ddev.site` | — | — |
| Deploy | GitHub Actions FTP sync (backend), Vercel (player SPA), none (admin SPA) | — | See `ARCHITECTURE.md` → Environments & deploy |

The two SPAs have drifted apart on patch/minor versions (Vue 3.5.21 vs 3.5.34, Axios
1.12 vs 1.16, Prettier 3.6 vs 3.8). Nothing depends on them matching today; it is
recorded here so nobody assumes they do.

## Check command

**There is none.** No repository has a single committed command that runs tests,
lint, static analysis and build and exits non-zero on the first failure.

What exists instead, per repository:

```bash
find wp-content/themes/pc -name '*.php' -print0 | xargs -0 -n1 php -l   # backend, also run by CI
npm run lint && npm run build                                           # frontend, also run by CI
npm run lint && npm run build                                           # admin, no CI
```

Until a real check command lands, the commit gate is: lint every app the step
touched, and build every SPA it touched. Creating the check command is the job of the
first code step of the first new feature's Sprint 1 — see `/do-step` §3,
which cannot be satisfied properly before then.

Two caveats a check command has to deal with: `npm run lint` is defined with
`--fix`, so it *mutates* files rather than only reporting; and the backend lint is a
syntax check only, not a style or static-analysis pass.

## ANTI-PATTERNS (mandatory reading before writing code)

<!-- Adoption: derived from what this codebase actually does and deliberately
     avoids. Each line is a mistake somebody could plausibly make here. -->

- **Do not add an `ENUM` column.** `dbDelta` cannot diff one, so adding a member
  later silently skips the migration. Use `VARCHAR` plus class constants, validated
  in the controller — as `wp_pc_transactions`, `wp_pc_machine_events`,
  `wp_pc_support_tickets`, `wp_pc_room_messages` and `wp_pc_room_schedules` all do.
- **Do not touch a money column outside `Wallet_Service`.** No controller, no
  service, no CLI command writes `wp_pc_wallets`, `wp_pc_coin_lots` or
  `wp_pc_transactions` directly. Every mutation goes through `Wallet_Service` under
  `SELECT … FOR UPDATE`.
- **Do not put money in a float.** `DECIMAL(12,2)` / `DECIMAL(8,2)` in the DB,
  decimal strings over the wire and through PHP. `floatval` in a money path is a bug,
  not a shortcut.
- **Do not write a machine payout to the ledger.** Machine credits insert a coin lot
  and move `balance_coins`; their audit trail is `wp_pc_machine_events`. The ledger is
  top-ups and withdrawals only, because that is what the player's history shows.
- **Do not call Home Assistant from anywhere but `Machine_Service`,** and do not let
  its failures reach a SPA as a 401 — map the typed `WP_Error`s to 502 / 503, or the
  Axios interceptor will treat a dead machine as an expired session and sign the
  player out.
- **Do not debit a coin before the machine answers 200.** `toss_coin()` enforces the
  200; the caller re-credits the exact lot price on anything else.
- **Do not flip a transaction to `completed` anywhere but the LiqPay callback,** and
  keep that handler idempotent on `(order_id, status)`.
- **Do not register a REST route without an explicit `permission_callback`,** and do
  not invent a new gate inline — use `Permissions::require_logged_in` /
  `require_chat_ready` / `require_play_ready` / `require_admin`. If a route must be
  public, it carries its own credential: a bearer token in the body, a provider
  signature, a rate limit, or a captcha.
- **Do not check the `play` capability.** It exists on the `player` role but nothing
  reads it; gates are user-meta and `manage_options` based. Adding a `current_user_can('play')`
  check would silently lock out administrators-turned-testers and contradict every
  other gate.
- **Do not write a meta key as a string literal.** `User_Meta_Keys::` and
  `Post_Meta_Keys::` constants only — that registry is the reason the meta surface is
  auditable at all.
- **Do not hard-delete anything that carries evidence.** Chat moderation flips
  `status`; a retired support subject is trashed; a drained coin lot stays at
  `qty = 0`; a revoked refresh token keeps its row. A `DELETE` in a moderation or
  money path is almost certainly wrong.
- **Do not add a cron job for queue housekeeping.** Queue pruning happens on read;
  the SPA's 3s poll is the heartbeat. The design deliberately heals on traffic alone.
- **Do not paginate chat with `LIMIT/OFFSET`.** Reads are cursor-based on `id`
  (`?after=<last id>`); an offset would re-send or skip messages between two polls.
- **Do not build product UI in `/wp-admin/`,** and do not introduce ACF. Every
  operator surface is a view in the admin SPA against `pc/v1/admin/*`. `/wp-admin/`
  stays available for plugin, theme and emergency DB work only.
- **Do not extract a shared component package between `frontend/` and `admin/`**
  without a decision first. Some duplication of Vue tooling and primitives is
  deliberate; a monorepo or shared package was explicitly deferred.
- **Do not put a secret in a WP option or in the repository.** wp-config constants
  only, never logged. Public counterparts (LiqPay public key, captcha site key) are
  options.
- **Do not hardcode an operator-tunable value.** Coin price bounds, machine entity
  ids, the bonus map, the queue idle timeout, TTLs, the support address and the
  captcha provider are all WP options with defaults in `Install_Schema`.
- **Do not "re-enable" the parked Google sign-in or the guest captcha by editing
  code.** Both are off because their configuration is empty — that empty config *is*
  the switch. Nothing is commented out; nothing should be.
- **Do not assume machine events arrive.** Nothing pushes them in yet; the only
  producer is `wp pc machine-ingest`. Code that depends on live events must say so.
- **Do not add a WordPress plugin to solve something the theme can do.** The only
  plugin in the product path is the JWT one, and even that is used for validation
  only — the theme mints its own access tokens.

## Dependency policy

New dependencies (runtime AND dev/tooling) only after explicit user approval — see
CLAUDE.md core rule 1. Record approved additions here with one line of justification.

The stack table above is the as-is inventory at adoption time and predates this
policy; it is not a list of pre-approvals for anything beyond what is already
installed.

## Approved dependencies log

| Date | Package | Why |
|---|---|---|
