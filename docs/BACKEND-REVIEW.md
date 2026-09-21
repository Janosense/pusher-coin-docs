# Backend Code Review

- **Date:** 2026-09-15
- **Reviewed:** `backend/` repository, `main` at `dc6873ee`
- **Scope:** the custom `pc` theme, about 5,800 lines across 45 PHP files, plus the deploy
  workflow and tracked config. WordPress core, the JWT plugin and `vendor/` libraries were
  not reviewed.
- **Type:** read-only analysis. No code was changed. All 45 files pass the `php -l` syntax
  check CI runs. There are no tests to run.

File paths below are relative to `backend/wp-content/themes/pc/app/` unless they start with
`backend/`.

## In short

The code is well organised and unusually well documented, and the everyday security basics
are done right. The serious problems are all in the areas CLAUDE.md marks as test-critical:
coin crediting, refunds, and login. None of those areas has tests, so none of these bugs
could be caught automatically.

## What's done well

- Every API route declares who may call it, and every database query uses safe parameters.
  No SQL injection was found.
- Machine events can't be credited twice. A database rule (unique index on `event_key`)
  enforces this, which is the correct approach.
- The LiqPay signature check is correct (constant-time comparison), and refresh tokens are
  stored hashed.
- Machine faults are never reported to the apps as "logged out" (mapped to 502 / 503).
- Comments explain *why* the code works the way it does, not just *what* it does.

## Must fix: money and security

### [fixed] 1. Wallet "all-or-nothing" writes aren't all-or-nothing
The wallet code expects a failed database write to trigger an undo (rollback), but WordPress
switches that signal off (`backend/wp-includes/class-wpdb.php:1964`), so the undo never runs.

- **Effect:** if adding the coins fails during a top-up, the payment is still marked
  completed and the player gets nothing. A retry can't fix it either, because the payment
  already looks settled.
- **Where:** `utils/wallet-service.php:184–475`.

### 2. A duplicate LiqPay notification can credit coins twice
The handler checks "still pending?" and then updates in a separate step. Two identical
notifications arriving together both pass the check.

- **Where:** `rest-api/PaymentController.php:74`, `utils/wallet-service.php:186`.

### 3. Rejecting a withdrawal twice refunds the coins twice
This happens with a double-click, or two operators acting at once. Approve and reject at the
same time means the player is paid *and* refunded.

- **Where:** `rest-api/AdminWithdrawalController.php:101–128`,
  `utils/wallet-service.php:395`.

### 4. The login code endpoint allows unlimited password guessing
`/user/verify-code` has no rate limit, and its error message reveals whether the password
was right. `docs/ARCHITECTURE.md:176` says it is rate-limited; it isn't.

- **Where:** `rest-api/UserController.php:366–390`.

### 5. Every rate limit can be bypassed with a fake header
The limiter trusts the `X-Forwarded-For` header, which anyone can set. That affects sign-up,
login codes, Google login and support tickets, and it means logged IP addresses (audit log,
chat, support tickets) can be forged.

- **Where:** `utils/rate-limiter.php:43`.

### 6. A real JWT signing secret is committed to git
The secret is in `backend/wp-config-ddev.php:41`, and that file gets FTP-uploaded to
production with everything else. If production uses the same key, anyone with repository
access can mint login tokens for any account, admins included.

- **Action:** rotate the key either way, and move it out of the tracked file. This also breaks
  the TECH-STACK.md rule against secrets in the repository.

### 7. Deploying to a fresh server would break the whole site
`backend/wp-content/themes/pc/vendor/` and `composer.lock` are gitignored
(`backend/.gitignore:28–29`), and the deploy workflow never installs them. Yet
`backend/wp-content/themes/pc/functions.php:7` requires `vendor/autoload.php`. Production
works only because someone uploaded `vendor/` by hand.

- The folder is 188 MB because the Google library bundles about 200 Google services just to
  verify a login token.
- Without a committed `composer.lock`, library versions can't be reproduced.

## Should fix

8. **Login and password-change codes use a predictable random generator.** They use
   `mt_rand` instead of `random_int` (`rest-api/UserController.php:321, 658`,
   `rest-api/GoogleAuthController.php:136`).
9. **Refresh-token rotation has a race.** Two simultaneous refreshes with the same token both
   succeed, so a stolen token used at the same moment escapes reuse detection
   (`utils/refresh-tokens.php:43`).
10. **Some failures are silent:**
    - If refunding a failed toss fails, the player loses the coin with no record
      (`rest-api/RoomQueueController.php:187`).
    - The 2-second machine timeout (`utils/machine-service.php:30`) means a slow but
      successful toss gets refunded, i.e. a free toss.
    - **[settled 2026-09-21 — `realtime` Sprint 1 Step 4]** A database error on a machine
      event is reported as a "duplicate" and the payout is dropped
      (`utils/machine-events.php:85`).
      *Settled:* `Machine_Event_Log::record_result()` answers `inserted`, `duplicate` or
      `failed`, and the crediting path (`Machine_Ingest_Service::settle()`) turns `failed`
      into a `machine_event_write_failed` 500 so the caller retries instead of retiring
      the event unpaid. `record()` keeps its old meaning for callers that have one.
      *Still true:* the audit-only path (`log_event()`, used by the toss endpoint) reports
      the failure as a flag and carries on — a toss whose audit row could not be written is
      still a toss, and failing it would cost the player a real coin. And `settle()` still
      writes the row outside a transaction with an unchecked `mark()`, so a crash between
      the row and the credit leaves a row every replay skips; no step names that yet.
11. **LiqPay `sandbox` payments count as real money** (`rest-api/PaymentController.php:78`).
    A chargeback that arrives after coins were credited is ignored without being logged
    (`rest-api/PaymentController.php:74`).
12. **[settled 2026-09-18 for rooms that name the same machine id — `realtime` Sprint 1
    Step 3]** **Every room drives the same physical machine** (`utils/machine-service.php:38`). If two
    rooms are set "available", two separate queues control one machine, and wins can go to
    the wrong player.
    *Settled:* a non-empty `pc_room_machine_id` can back only one available room.
    `POST`/`PUT /admin/rooms` refuse a second with `machine_already_in_use`, a queue join
    into a pair already in the data is refused, and `wp pc machine-rooms` lists shared ids
    (`app/realtime/machine-rooms.php`).
    *Still true:* `Machine_Service` drives one physical machine whatever the id, so two
    available rooms with **different** or **empty** ids still share it. The guard protects
    the machine only when the id names it.
13. **Public room endpoints expose unfinished rooms.** They show draft and even trashed rooms
    (`rest-api/RoomController.php:59, 108`).
14. **Money is sometimes handled as a float,** which the TECH-STACK.md rules forbid:
    - Session winnings (`utils/queue-service.php:341, 431`).
    - Price input parsing (`rest-api/WalletController.php:197`,
      `rest-api/AdminCoinPricingController.php:97`).
15. **[settled 2026-09-21 — `realtime` Sprint 2 Step 5]** **Queue polls write to the
    database every 3 seconds per viewer.** Simultaneous polls can open duplicate game
    sessions that never close (`utils/queue-service.php:253`, now `:307`).
    *Settled:* the polls went first — since Sprint 2 Step 2 a signed-in client
    subscribes and sends one cheap heartbeat instead of a 3-second full read — but that
    only thinned the traffic, and the race was never in the polling. `sync_turn()` read
    "is there an open session?" and then inserted one, so any two simultaneous requests
    could each open a session; measured on the local install with 12 concurrent
    processes against the pre-fix logic, **all 12 opened one**. `wp_pc_bet_sessions`
    now carries `open_room_id` — the room id while the session is open, NULL once it
    closes — under `UNIQUE KEY open_room`, so the database refuses the second row and a
    caller that loses the insert adopts the winner's session. The same 12 processes
    through the fixed `sync_turn()` produce exactly one. The head's `session_id` and
    the room's open session are now kept identical, which is the half that made this
    cost money: `open_session()` answers the newest row while the room screen shows the
    row the head points at, so a payout could be banked on a session the player was not
    looking at. Sessions the old code left open are closed (never deleted) on the
    version bump, each with an audit row, and `wp pc queue-sessions` reports the state
    of any install.
    *Still true:* a guest still polls the chat every 3 seconds — the room and its chat
    are public but an Ably pass is not (`DECISIONS.md` 2026-09-21) — so guest traffic
    is unchanged. Rows written before the migration carry a NULL `open_room_id` and are
    not covered by the key until it runs; `wp pc queue-sessions` flags them.

## Cleanup

- **Copy-pasted code:**
  - The room loading and formatting helpers exist in both `rest-api/RoomController.php` and
    `rest-api/AdminRoomController.php`.
  - The email-code login flow exists in both `rest-api/UserController.php` and
    `rest-api/GoogleAuthController.php`.
  - The IP helper (`ip_binary`) is copied four times.
  - Defaults like `'40.00'`, `'2026-05'` and `900` are repeated in 3–5 places, so changing one
    and missing another is easy.
- **Rule breaks and wrong comments:**
  - `rest-api/WalletController.php:127` writes the transactions table directly, which
    TECH-STACK.md forbids (only `Wallet_Service` may).
  - Apple's private key may be read from the database
    (`rest-api/AppleAuthController.php:105`), which is against the secrets rule.
  - A comment in `set_nickname` says it only renames auto-generated logins, but it renames
    everyone's, admins included (`rest-api/UserController.php:475`).
  - Sign-up doesn't apply the same nickname rules as set-nickname
    (`rest-api/UserController.php:131` vs `:444`).
- **Other:**
  - Leftover debug output: `backend/wp-content/themes/pc/index.php:2` prints a test counter
    on the public site.
  - Timestamps mix site-local time and UTC across tables.
  - `rest-api/UserController.php` is 741 lines doing five different jobs (sign-up, login,
    profile, email confirmation, password change).

## Why this slipped through

There are no tests and no static analysis; CI checks syntax only. Items 1–3 and 9 are the
kind of bug tests written for the money and token rules would catch.

## Suggested order

1. Rotate the JWT secret (item 6).
2. Fix items 1–5, each with tests written alongside. Items 1–3 share one fix: lock or
   conditionally update the row, and check every database write's result.
3. Make deploys self-contained (item 7).
4. Work through "Should fix", then "Cleanup".

These fixes are outside the current sprint, so each one goes through `/adhoc` or becomes a
sprint step.
