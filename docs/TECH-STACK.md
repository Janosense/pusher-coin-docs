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
| Player SPA extras | `hls.js` 1.6.16 (dynamic import), `imask` 7.6.1, `ably` 2.28.0 (dynamic import) | — | Mux LL-HLS playback; phone / code / coin-quantity masking; the live room channel |
| Player SPA — Ably client | `ably` | 2.28.0 (locked `realtime` S2.2) | Approved under core rule 1. It does the token exchange against our `authUrl`, the reconnect backoff and the renewal on expiry — the parts this feature is actually about and the parts `frontend/` has no test runner to cover. The reason the backend has **no** SDK (`vendor/` never reaches production, `DECISIONS.md` 2026-09-17) does not apply here: Vercel builds from `package.json`. Dynamically imported, like `hls.js` and for the same reason: it is its own 212 kB / 58 kB-gzipped chunk that only a room ever fetches, leaving the main bundle at 261 kB / 92 kB — where it was before |
| Admin SPA | Vue 3.5.34, Vite 5.4.21, Pinia 3.0.4, Vue Router 4.6.4, Axios 1.16.0 | — | Same stack as the player SPA, without `hls.js` and `imask` |
| Lint / format | ESLint 8.57.1 + Prettier (3.6.2 frontend, 3.8.3 admin); `php -l` for the theme | — | No PHPCS, PHPStan or Psalm anywhere |
| Testing | **none** | — | No PHPUnit, no Vitest, no Playwright, in any of the three repositories |
| Node | 20 (CI) | — | `frontend/.github/workflows/ci.yml` |
| Local env | DDEV (nginx-fpm), host `https://pusher-coin.ddev.site` | — | — |
| Stripe CLI (developer tooling, **not** a project dependency) | `stripe` | 1.43.8 on the development machine | `stripe listen --forward-to …` delivers real test events to DDEV, which Stripe cannot reach; it also supplies the per-session `whsec_`. Installed per developer, never committed, never required by CI — `DECISIONS.md` 2026-09-17 |
| Deploy | GitHub Actions FTP sync (backend), Vercel (player SPA), none (admin SPA) | — | See `ARCHITECTURE.md` → Environments & deploy |

The two SPAs have drifted apart on patch/minor versions (Vue 3.5.21 vs 3.5.34, Axios
1.12 vs 1.16, Prettier 3.6 vs 3.8). Nothing depends on them matching today; it is
recorded here so nobody assumes they do.

### Ably free tier — what actually spends it

Written down at the end of `realtime` Sprint 2 (Step 5) so Sprint 3 has a number to
plan against. The tier is **6M messages/month, 200 concurrent connections, 200
channels** (`DECISIONS.md` 2026-09-15).

**Observed peak: none. Nothing has ever connected.** No Ably account exists, no
`PC_ABLY_KEY` is set on any install, and no dashboard has been opened — through
Sprint 2's four steps the publisher has only ever been exercised against a stub. The
first real number comes from Ably's dashboard after the key is set on the host, on a
day the venue is open; until then what follows is arithmetic, not measurement, and
should be read as an upper bound on what the design can spend rather than as what it
does spend.

| Ceiling | What spends one | Reached at |
|---|---|---|
| 200 concurrent connections | One per **signed-in player with a room open**. One connection serves every listener in that browser — the queue and the chat share it (S2.4) — so it is one per browser tab, not one per store. Guests spend none: they have no pass and keep the 3-second chat poll (`DECISIONS.md` 2026-09-21). The admin SPA spends none: `admin/src/services/realtime.js` is still unbuilt. | **200 simultaneous signed-in room viewers** — not 200 players, not 200 rooms |
| 200 channels | One per room, plus one for the machine. Never one per viewer (S2.1). | 199 rooms |
| 6M messages/month | `queue` on every join / leave / toss, `credit` on every payout, `relay` only on a real transition (at most one per 60-second poll pass), `chat` and `moderation` per message. **Chat is the highest-volume of these by some distance** and is the one to watch as rooms fill. | — |

The connection ceiling is the binding one, and it binds on *simultaneous signed-in
viewers of a room*. Two things already lean on that being the number: guests were left
on the poll rather than given a pass, precisely so a read-only viewer does not spend a
connection; and one connection is shared across a tab's listeners rather than one per
store. Both decisions get cheaper to revisit once a real peak exists, and neither
should be revisited before then.

## Check command

One committed script each in `backend/` and `frontend/`; each stops at the first failure
with a non-zero exit and runs from any directory. Commit only on exit 0.

```bash
backend/bin/check    # 1. php -l over every PHP file of the pc theme, vendor/ excluded
                     # 2. every wp-content/themes/pc/tests/*.php through `ddev wp eval-file`
frontend/bin/check   # 1. npm run lint (report-only)   2. npm run build
```

`admin/` has no check script yet; its gate is `npm run lint && npm run build`.

`backend/bin/check` runs stage 2 only when the DDEV project is already running
(`ddev describe` reports it running and `ddev wp core is-installed` answers). Otherwise
it prints a boxed `SKIPPED: DDEV checks did not run` notice and still exits 0 — a green
run that shows that notice has not executed the money checks. It never starts DDEV
itself, because `ddev start` rewrites `wp-config-ddev.php` (`docs/LEARNINGS.md`).

Test-critical checks that exist so far — WP-CLI eval scripts in
`backend/wp-content/themes/pc/tests/`, run by `backend/bin/check` against the local DDEV
database. CI does not run them (it has no database), and each refuses to run outside
WP-CLI on DDEV:

```bash
ddev wp eval-file wp-content/themes/pc/tests/wallet-rollback.php   # backend: every Wallet_Service write failure rolls back (53 checks)
ddev wp eval-file wp-content/themes/pc/tests/machine-poll.php     # realtime: the HA history poller — arithmetic, replay, gaps, scheduling, crediting (71 checks)
```

The list above is not exhaustive — `tests/` holds more scripts than it names, and
`backend/bin/check` runs every one of them. Keeping the list current is a standing
`/adhoc` item.

A new script in `tests/` is picked up by `backend/bin/check` without editing it, and must
keep the WP-CLI + DDEV guard (`DECISIONS.md` 2026-09-15).

**The machine transport's schedule is not covered by any check command.** WordPress
polls Home Assistant's history on a schedule (`DECISIONS.md` 2026-09-18); the poller
and its wiring are checked by `machine-poll.php`, but *whether the host actually ticks*
is a property of the host, not of the code. Two things can drive it and both are safe
together:

- **WP-Cron**, scheduled by the feature bootstrap — needs no host setup, but runs only
  when the site gets traffic, and not at all where `DISABLE_WP_CRON` is true or
  loopback requests are blocked. A player holding a turn generates traffic every 3s, so
  it ticks exactly when it matters.
- **A real cron**, if the host offers one — then `DISABLE_WP_CRON` may be set to true
  in `wp-config.php` and the line is:

  ```cron
  * * * * * cd /path/to/wordpress && wp pc machine-poll --quiet >/dev/null 2>&1
  ```

`wp pc machine-poll --status` says which of the two has been running, and when — it
reads `pc_realtime_poll_last_run`, so it answers on a host where nobody has a shell to
watch with. Verified only by a real host, never locally.

`npm run lint` only reports; `npm run lint:fix` is the one that rewrites files. CI runs
the underlying commands rather than the scripts: `php -l` over the theme in `backend`,
`npm run lint` and `npm run build` in `frontend`.

Two caveats: the backend lint is a syntax check only, not a style or static-analysis
pass; and `backend/bin/check` lints with the host's `php` (8.5 on the development
machine) while CI lints on 8.2, so CI stays the authority on syntax an older PHP rejects.

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
- **Do not expect `$wpdb` to throw.** WordPress switches mysqli error reporting off;
  a failed statement returns `false` and nothing more, so a `try/catch` around bare
  `$wpdb` calls never reaches its `ROLLBACK`. Inside a transaction, check every
  statement — `START TRANSACTION` and `COMMIT` included — as
  `Wallet_Service::ensure_written()` does.
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
- **Do not flip a transaction to `completed` anywhere but the Stripe webhook,** and
  keep that handler idempotent on the row's own `status`, never on delivery order —
  Stripe delivers at least once and out of order. The player's return from the
  hosted page settles nothing.
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
- **Do not add a cron job for queue housekeeping.** Queue pruning still happens on a
  request, and the design still heals on traffic alone — but the request is no longer
  the SPA's 3-second full read. Since `realtime` Sprint 2 Step 2 it is an explicit
  `POST /rooms/{id}/queue/heartbeat`: a cheap write that touches the caller, prunes the
  absent, promotes the next player and answers a version rather than the queue. The
  rule is unchanged; only its reason moved.
- **Do not paginate chat with `LIMIT/OFFSET`.** Reads are cursor-based on `id`
  (`?after=<last id>`); an offset would re-send or skip messages. Since `realtime`
  Sprint 2 Step 4 the gap it protects is no longer "between two polls" but "while a
  client was disconnected", which is longer and less predictable — the cursor stopped
  being the transport and became the catch-up. The rule is unchanged; only its reason
  moved, and it matters more now, not less.
- **Do not build product UI in `/wp-admin/`,** and do not introduce ACF. Every
  operator surface is a view in the admin SPA against `pc/v1/admin/*`. `/wp-admin/`
  stays available for plugin, theme and emergency DB work only.
- **Do not extract a shared component package between `frontend/` and `admin/`**
  without a decision first. Some duplication of Vue tooling and primitives is
  deliberate; a monorepo or shared package was explicitly deferred.
- **Do not put a secret in a WP option or in the repository.** wp-config constants
  only, never logged. The captcha site key is the public counterpart that is an
  option; the top-up provider has none at all — hosted Checkout needs no public key.
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
