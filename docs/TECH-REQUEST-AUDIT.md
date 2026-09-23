# Tech request audit — the customer's ТЗ against the code

<!-- Written 2026-09-22 in a Cowork discovery chat (solution architect, no
     code changed). Input for the next Feature-mode chat. This file is an
     audit and a backlog candidate list — it is NOT a FEATURE.md, NOT a
     sprint file and NOT a decision. Every "Gap" below becomes work only
     after a Feature-mode chat turns it into DECISIONS.md entries and a
     confirmed sprint plan. Superseded by those files once they exist. -->

## What this is

The customer wrote a technical request (Russian, "Тз на разработку проекта",
30 screen mockups embedded). Its text is preserved as `docs/TECH-REQUEST.md`
(images dropped — the original file with the mockups stays with the owner).
This document walks that request line by line and says, for each demand,
whether the code does it today. The tags are checked against the code on
the user's machine, all three repositories on `main` (root docs `41b6e68`,
`backend` `aa27fe22`, `frontend` `2d525d4`, `admin` `e18ff60`), and against
`docs/ROADMAP.md` (last audited 2026-09-15, re-tagged through 2026-09-22),
`docs/CONTRACTS.md` (the current endpoint list), `docs/DECISIONS.md`,
`docs/DOMAIN.md` and `docs/DESIGN.md` → Screens.

**The ТЗ and the project's own to-do list are not the same document.**
`ROADMAP.md` tracks 17 items from an earlier list (`[#1]`–`[#17]`), and all
17 are `[done]` or `[partial]` with the residue named. The ТЗ is wider: it
adds phone/SMS, Google Authenticator, password recovery, the game-turn
choreography (countdown, auto-toss, "continue or collect"), a games tab in
History, player settings and notifications, three languages, a whole set of
admin surfaces (players, blocked users, demo users, admin accounts, static
pages, error texts), stream recording and a watermark, analytics. None of
that is in `ROADMAP.md`, so none of it has ever been planned — that is what
the next Feature-mode chat is for.

### Legend

| Tag | Meaning |
|---|---|
| **done** | The code does what the ТЗ line asks, checked in the code (file named). |
| **partial** | Part of the line is built; the missing part is named. |
| **differs** | Built, but not the way the ТЗ says — and the difference is a *recorded decision* (`DECISIONS.md` / `ROADMAP.md`). The Feature chat must confirm the ТЗ or the decision; it must not silently rebuild. |
| **gap** | Nothing in the code or the docs. |
| **not verified** | A UI nuance (layout, animation, exact mobile behaviour) that a code read cannot settle; needs a look at the running SPA against the mockup. |

Where the ТЗ writes a number that the business may change (10 s, 15 s, 5 min,
12 h, 10 coins, 7 days), the future work must make it a WP option / admin
setting, never a constant (root `CLAUDE.md` core rule 3).

---

## 1. The machine and its integration

| # | ТЗ asks | Status | Where / what is missing |
|---|---|---|---|
| 1.1 | One button press = one coin tossed into the machine | **done** | `Machine_Service::toss_coin()` presses `input_button.toss_a_coin`; `POST /rooms/{id}/play` debits one coin FIFO and re-credits on anything but HTTP 200 (`RoomQueueController.php`, CONTRACTS → `/play`). |
| 1.2 | A counter that reports the player's winnings (coins in the central tray) to our server | **done** | `sensor.coin` counts coins paid out since the last toss; `Machine_Poller` reads its Home Assistant history every 60 s and delivers each change to `POST /machine/events`; `Machine_Ingest_Service` credits the player holding the turn (`realtime` Sprint 1). Measured latency ≈ 65 s (`DECISIONS.md` 2026-09-18). |
| 1.3 | Bonus wheel (3 / 5 / 8 / 10 coins + stations) and the 7-station "coin train" | **partial** | A 12-entry bonus map exists (`GET/PUT /admin/machine/bonus-map`, `pc_machine_bonus_map`) and `ingest_bonus` can credit it — but **no bonus was ever observed** in ten days of machine history (`sensor.lc01_12` stayed 0), so how the machine reports one, and whether train coins also pass through `sensor.coin`, is unknown. Stations (progress 1 → 7) are not modelled anywhere. Open question, not a code gap, until a bonus is seen. |
| 1.4 | Reset the previous player's win counter when the turn passes | **differs** | The spike found that **every toss resets `sensor.coin` to 0** within ~2 s, and Home Assistant exposes no separate "reset" command (`PUSHER-COIN-COMMANDS.txt`). Attribution is by the open bet session instead: a payout is credited to whoever holds the turn at that moment, and nobody otherwise (`DOMAIN.md`). Confirm with the venue that no explicit reset is needed. |
| 1.5 | Ping the machine | **done** | `Machine_Service::is_online()`; admin Machine screen shows the probe; `Realtime_Outage_Watch` emails the operator when the machine is unreachable during a broadcast window (`realtime` Sprint 3 Step 1). |
| 1.6 | Every command gets a success answer or an error code | **done** | Typed `WP_Error`s (`machine_offline`, `machine_call_failed`, `machine_unauthorized`, `machine_not_configured`) mapped to 502 / 503; `CONTRACTS.md` error registry. |
| 1.7 | The coin hopper refills itself ("fountain") | **n/a** | Venue hardware; the software neither knows nor needs to. |

## 2. Player site — guest

| # | ТЗ asks | Status | Where / what is missing |
|---|---|---|---|
| 2.1 | Main screen = rooms list; header shows total players playing across all rooms; side menu; logo slot | **partial** | `RoomsView` / `RoomList.vue` with status badge and next-broadcast countdown per tile. The **global** "players online" total in the header is not built — the online counter exists per room only (`Queue_Service::state()`, "online players = queued players"). Logo slot: not verified. |
| 2.2 | Guest ≠ online player | **done** | Online counts queued players, never watchers (ROADMAP Phase 6 §2). |
| 2.3 | Guest can write to Support | **done** | `SupportView` guest path with email field + captcha (captcha provider **unconfigured** — launch blocker, ROADMAP Phase 7 §1). |
| 2.4 | Language switch: UA / EN / RU, more languages addable from the admin | **gap** | `LanguageSwitcher.vue` is a dead control (three hardcoded buttons UA / GER / ENG, no library, no strings externalised — `DESIGN.md` → Out of scope). No i18n anywhere in `frontend/`, `admin/` or the API's error messages. ROADMAP Phase 8 names vue-i18n as the likely pick; nothing decided. |
| 2.5 | Guest opens any room and watches the stream, sees chat and queue | **done** | `RoomView` is public: `LiveStream` (Mux LL-HLS via hls.js), read-only `RoomChat`, `RoomQueue`. |
| 2.6 | Guest clicking chat / Play / Account / History / Settings / Sign in gets the sign-in-or-sign-up dialog | **partial** | Chat and Play: done (in-page sign-in overlay seeded with a redirect, ROADMAP Phase 3 §5). Side menu: `AppNavigation.vue` **hides** Account and History from a guest (`v-if="authentication.isAuthenticated"`) and shows Sign in / Sign up instead of prompting — a small behaviour difference from the mockup. Settings does not exist (see 5.5). |
| 2.7 | "?"-icon texts "Game rules" and "Queue rules", editable from the admin | **gap** | No static-content store, no endpoint, no dialog. `UserControls` has a "rules" link target only in the ТЗ. |

## 3. Player site — registration, sign-in, recovery

| # | ТЗ asks | Status | Where / what is missing |
|---|---|---|---|
| 3.1 | Sign-in with email **or nickname**, password, and a Google Authenticator code | **differs** | Identifier: `login` accepts either, because nickname *is* `user_login` (`UserController.php:185`) — done. Second factor: a **6-digit code sent by email** (`request-verification` → `verify-code`, 15-min TTL), not a TOTP app. No `totp` / authenticator code exists. The email code was chosen because there is no way to query Google's own 2FA (ROADMAP Phase 2 §3); TOTP would be a new user-meta secret + QR enrolment + a new gate. Decision needed. Also: `verify-code` has **no rate limit** and leaks wrong-password vs wrong-code (`BACKEND-REVIEW.md` §4) — security review item. |
| 3.2 | Sign-up fields: phone\* with input mask and **SMS confirmation**, email\* validated, unique nick\*, password\* with a rule, repeat password\*, Google Authenticator to finish | **partial** | Email, nickname (unique, `username_exists`), password (`weak_password` <6 chars — a weaker rule than the ТЗ implies), mandatory terms checkbox — done (`POST /user/sign-up/`). **Phone is optional and never verified**: `phone_verified` is hardcoded `false` (`UserController.php:557`), Account shows "Not yet supported"; phone verification was **deliberately deferred** (ROADMAP Phase 2 §5). No SMS provider, no code table, no input mask. Google Authenticator: see 3.1. Red-border highlighting of empty fields: not verified. |
| 3.3 | SMS code: 6 digits, valid 5 min, resend after 5 min, then a 12-hour lockout | **gap** | Nothing. The email-code flow has its own TTL (`pc_email_confirmation_ttl_seconds` 24 h for the link, 15 min for the login code) and rate limits (5 / 15 min per IP; 10 sign-ups / 24 h per IP) — a different scheme. All four numbers must be options. |
| 3.4 | Email confirmation link at sign-up | **done** | `request-email-confirmation` / `confirm-email`, `ConfirmEmailView` with "send a new link"; play and money are gated on `email_verified_at` (`Permissions::require_play_ready`). |
| 3.5 | Password recovery by **email** (reset link → new password → redirected home, signed in) | **gap** | No `forgot` / `reset-password` route in `CONTRACTS.md` or the code. What exists is a *change* of password for a signed-in user (`request-password-change` → code → `confirm-password-change`), which cannot help someone locked out. |
| 3.6 | Password recovery by **SMS** (6-digit code, same limits as sign-up) | **gap** | Depends on 3.3. |
| 3.7 | One free coin for a genuinely new user (new email **and** new phone) | **gap** | No welcome-credit code. Needs a `Wallet_Service::credit_lot` at a `unit_price` the operator sets (or 0?), a fraud rule that depends on verified phone (3.2), and an audit row. Decide what price a free coin carries when it is later won back or withdrawn — the FIFO-lot model (`DECISIONS.md` 2026-05-13) forces the question. |
| 3.8 | Google sign-in / Apple sign-in | *(not in the ТЗ)* | Both exist; Google is parked behind an empty client id, Apple is a stub (ROADMAP Phase 1 §2–3). The Feature chat should ask whether the customer wants them at all. |

## 4. Player site — the game

| # | ТЗ asks | Status | Where / what is missing |
|---|---|---|---|
| 4.1 | Play with no room chosen → dialog "choose a room" | **not verified** | Play lives inside a room today (`RoomView`); there is no global Play button on the rooms list. Probably moot — confirm against mockup screen 6. |
| 4.2 | Click balance → top-up screen; coin price set by the admin; pay through the payment provider; coins appear after a successful transaction | **done** | `ReplenishmentBalance.vue` → `POST /wallet/topup` → Stripe hosted Checkout → `POST /payments/stripe/webhook` settles atomically (`stripe` feature). Pricing: default / min / max in admin Settings. Zero-balance Play routes straight to top-up (ROADMAP Phase 4 §5). |
| 4.3 | Room chat: **not saved, not moderated** | **differs** | Chat **is** persisted (`wp_pc_room_messages`) and **is** moderated (hide-not-delete, timed mute, admin `ChatView`) — `DECISIONS.md` 2026-09-07 and `DOMAIN.md` ("nothing disappears"). The code exceeds the ТЗ; keep it unless the customer objects. Mobile chat show/hide toggle: not verified. |
| 4.4 | Queue panel: mobile collapsible, auto-collapses after 5 s idle, shows first + last when collapsed, ellipsis rules for 1 / 2 / 3+ players; desktop always visible with "me" pinned at the bottom with my number | **partial / not verified** | `RoomQueue.vue` exists: head pinned and banded, own row marked "(you)". The mobile collapse choreography (5-s auto-collapse, first + last, ellipsis) and the desktop "me pinned at bottom" are not in the code as far as a read shows — needs a visual check against screens 8–9. |
| 4.5 | "Online players" = players in the queue, shown top-right | **done** | `Queue_Service::state()`; `RoomQueue` header. |
| 4.6 | Head of queue highlighted with nickname and the coins they play; me highlighted differently; turning purple + **sound** when I become first | **done** | Purple band for the head; `RoomView.playTurnChime()` (synthesised `AudioContext` chime) fires on reaching the head. |
| 4.7 | Play → dialog to choose how many coins from the balance (cannot exceed balance) → button shows "N coins, position K in queue"; position updates live | **partial** | `PlaceBet.vue` declares coins at join (`POST /queue/join` with `coins`, clamped to `balance_coins`, server re-checks `insufficient_balance`). The queue store is pushed over Ably (`realtime` Sprint 2) with a 3-s poll fallback, so position is live. The position is shown as text ("You're #K in the queue with N coin(s) declared", `PlaceBet.vue:172`), not on the button itself as in screens 18–19 — cosmetic. |
| 4.8 | Cannot join a queue twice, and cannot join a second room while queued | **partial** | Same-room double join is a no-op (`Queue_Service::join()` finds the existing entry). **Cross-room is not refused**: `join()` checks only `entry($room_id, $user_id)`. Gap: one rule + one error code. |
| 4.9 | When first: the button changes state; when a toss becomes possible a **10-s countdown** (admin-set) runs; if the player does nothing the system tosses for them, one coin every 10 s, with a sound; the coin count on the button decrements | **gap** | Today the player presses **once per coin** and nothing tosses on their behalf; a silent player is simply pruned from the queue after `pc_queue_idle_timeout_seconds` (60 s). Auto-toss is a server-side scheduler (the browser cannot be trusted to fire it — the tab may be closed) that must go through `Wallet_Service` + `Machine_Service::toss_coin()` exactly like a manual toss, and a `toss` audit row that says it was automatic. Interval and countdown are options. The toss sound: nothing in the code. |
| 4.10 | Won nothing → button back to Play, can go again | **done** | Turn ends when declared coins run out; the store returns to the not-in-queue state. |
| 4.11 | Won something → dialog "continue playing or take the winnings to the balance"; 10 s (admin-set) to choose, else the game ends and the coins go to the balance with an animation + sound; the **next player waits 15 s** (admin-set) after that | **differs / gap** | Winnings are **credited to the wallet the moment the machine reports them** (`Machine_Ingest_Service` → `credit_lot`), so "take to balance" is implicit and immediate — a decision, not an accident: payouts never wait on the browser (`DOMAIN.md`). "Continue with the winnings" (re-stake won coins without leaving the head of the queue) does not exist: the turn is sized at join time. The dialog, the two timers, the handover delay, the animation and the sound are all gaps. Note also that a payout arrives ~65 s after the coins fall (1.2), so a 10-s "collect" window cannot even see most of them — the timing model must be redesigned, not decorated. |
| 4.12 | Bottom panel: avatar + nickname (→ Account), sound on/off, Balance, Wins (this turn), link to rules | **partial** | `UserControls.vue`: `FacelessAvatar`, nickname, sound button (drives the **theme song**, `stores/themeSong.js`), balance, per-turn winnings (hidden at zero). Missing: a rules link with content (2.7); a separate "game sounds" toggle (5.5). Avatar → Account: not verified. |
| 4.13 | Winnings animate from the counter into the balance / onto the Push button | **gap** | No animation. |

## 5. Player site — side menu

| # | ТЗ asks | Status | Where / what is missing |
|---|---|---|---|
| 5.1 | Account: edit email, password, phone | **partial** | Nickname (inline), password (two-step with a mailed code), phone (free-text, unverified) — done in `AccountView`. **Email change** is absent (`PATCH /user/me` carries phone; there is no change-email flow with re-verification). |
| 5.2 | Withdrawal: minimum 10 coins and a multiple of 10 (87 → 80); paid within 3 banking days; request goes to the admin; the provider confirms the payout back into the admin and it is written to history | **partial / differs** | `POST /wallet/withdraw` (one pending at a time, FIFO-priced, `consumed_lots` for atomic refunds) and the admin Withdrawals screen (approve / reject-with-reason) — done. **Missing:** the min-10 / multiple-of-10 rule (server only checks `> 0`), an operator-facing SLA (3 days is a business rule; the 24-h "waiting too long" alarm from `realtime` Sprint 3 Step 3 is the nearest thing). **Differs:** there is no "attached card in the payment system" and no provider confirmation — Stripe Checkout takes money in only; payouts are **manual, out of band**, and automated payouts / KYC are explicitly out of v1 (`DECISIONS.md` 2026-05-13, 2026-09-16; ROADMAP open questions). Confirm with the customer. |
| 5.3 | Enable Google Authenticator 2FA from Account | **gap** | See 3.1. Today the Account page shows a static "Google 2FA" info panel linking to Google's own security page (ROADMAP Phase 2 §3). |
| 5.4 | Replay the onboarding tour; delete my account | **gap** | No tour exists at all; no self-service deletion. Deletion must respect "nothing disappears" (`DOMAIN.md`): anonymise the user, keep ledger, tickets and chat rows; refuse or settle a pending withdrawal and a non-zero balance first. |
| 5.5 | Settings: background music on/off, game sounds on/off, "promotions" notifications yes/no, "game start" notifications via Telegram / Viber / email | **gap** | No Settings screen and no `settings` route (`DESIGN.md` → Screens). The theme-song preference exists but is device-local (`pc_theme_song_enabled` in browser storage) — a Settings screen would have to decide whether it becomes server-side. Notifications: no preference meta, no channel integrations (Telegram is only mentioned as a possible second alert channel for the *operator*, `alerts.php`). |
| 5.6 | History — transactions tab: id, date/time, operation (top-up / withdrawal), amount, status (**log and show an error code on a failed one**) | **partial** | `GET /transactions` + `HistoryView`: type filter, date filter, status badges, pagination — done. **Error code on a failed row: gap** — the `GET /transactions` item carries `status` only (`CONTRACTS.md`), no provider reason, so there is nothing for the SPA to show. |
| 5.7 | History — **games tab**: date/time, coins played, coins won, balance at the time | **gap** | `wp_pc_bet_sessions` already stores per-turn `coins_declared / coins_played / coins_won / value`, but there is no player-facing endpoint and no tab (`DOMAIN.md` explicitly leaves tosses un-itemised; a **per-turn** history is consistent with that rule). "Balance at the time" is not stored — either snapshot it on the session or compute it. |
| 5.8 | Support: email (auto-filled and locked for a signed-in player; required + captcha for a guest), subject from an admin-managed list, description | **done** | `SupportView`, `GET /support/subjects`, `POST /support/tickets`, admin `SubjectsView`. Captcha unconfigured (2.3). |
| 5.9 | Logout with "Are you sure?" | **done** | `LogoutConfirmModal.vue`; `/auth/logout` revokes the refresh token. |

## 6. Admin panel

| # | ТЗ asks | Status | Where / what is missing |
|---|---|---|---|
| 6.1 | Admin sign-in: email + password + Google Authenticator | **differs** | Same email-code 2FA as players, gated by `GET /admin/me` (`manage_options`). TOTP: see 3.1. |
| 6.2 | Create / edit / delete admin accounts | **gap** | Admins are WordPress users with `manage_options`, managed only in `/wp-admin/` — which `DOMAIN.md` puts out of scope for product workflows. Needs an admin-users endpoint set + screen, with the rule that an admin cannot delete themselves / the last admin. |
| 6.3 | Switch a room's stream on/off, per room or all at once, by **time slots**; when off, the room tile shows a countdown to the next slot | **done** | `wp_pc_room_schedules` (weekly `always` rules + one-off `once`), `Room_Schedule_Calculator`, admin `RoomScheduleView`, player `NextBroadcastCountdown`. "All rooms at once" is per-room today — a convenience gap at most. |
| 6.4 | Rooms: create, delete, enable/disable, manage streaming; **a disabled room is not visible on the site** | **partial** | CRUD + status (`available` / `maintenance` / `unavailable`) + stream URL + theme-song URL — done (`AdminRoomController`, `RoomFormView`). But `GET /rooms` returns every room including `draft` and `unavailable` ones (`RoomController.php:59`) and the SPA shows them with a badge. Hiding is a one-line filter plus a decision on which status means "hidden". |
| 6.5 | Admin watches the stream as the players see it | **gap** | No `<video>` / hls.js in `admin/` at all. |
| 6.6 | Watermark on the stream: current date/time and the nickname of the player at the head | **gap, and not a web-app task** | The video is produced in the venue and pushed to Mux over RTMP (`DOMAIN.md`); an overlay is drawn where the video is encoded (OBS / the mini-PC), not in the browser. The app can *supply* the nickname (a small public read of the queue head, which exists), but the burn-in is venue tooling. Must be scoped explicitly in the Feature chat, or it will be assumed by both sides. |
| 6.7 | Site-wide maintenance mode with an admin bypass | **gap** | Only a per-room `maintenance` status exists. Needs a WP option, a public flag on a bootstrap read, an SPA gate and an admin-token bypass. |
| 6.8 | Per-room looping background melody: upload / replace / delete from the admin | **partial** | Playback is done (`stores/themeSong.js`, `pc_room_theme_song_url`), but the admin sets a **URL**, not a file; no upload, no media storage, and the file host must answer range requests (ROADMAP Phase 6 §1). |
| 6.9 | Every stream is recorded; kept 7 days (admin-set); auto-deleted; downloadable by the admin | **gap** | Nothing. Mux can record live streams as assets and delete them; the app would need the Mux API (a new integration + a secret), a retention option, a cron pass and a download link in the admin. Also a hosting-cost decision. |
| 6.10 | Logs and errors: loss of the machine, the camera, the mini-PC, the server | **partial** | Machine: `Realtime_Outage_Watch` (email, during broadcast windows only) + `wp pc machine-poll --status`. Toss that moved nothing: recorded. Withdrawals piling up: emailed. **Camera / stream health, mini-PC, server** are unobserved; there is no error log screen in the admin (audit rows live in `wp_pc_auth_audit_log` and `wp_pc_machine_events` with no UI). ROADMAP Phase 8 "Observability" (Sentry etc.) is unplanned. `wp_mail` delivery from the production host is **unproven** for every alert (WORKLOG 2026-09-22). |
| 6.11 | Players list: id, nick, email, phone, verified, balance; search by nick / email / phone; sort; filter by status | **gap** | No admin players endpoint and no view. |
| 6.12 | Player detail: transactions and games with date / time / balance / stake / result; number of support tickets; totals of top-ups and withdrawals | **gap** | Data exists in the ledger, bet sessions and tickets; no aggregate endpoint, no screen. |
| 6.13 | Delete players | **gap** | See 5.4 for the anonymise-not-delete constraint. |
| 6.14 | Blocked users database (bots, SMS abusers) | **gap** | Only a timed chat mute exists (`POST /admin/chat/mute`). No account block, no block reason, no IP list, no sign-in refusal. |
| 6.15 | Demo users — fake activity in the game | **gap** | Nothing. Note this collides with the domain: a queue entry is a real turn and a real toss costs a real coin; "fake activity" can only be presentational (fake queue rows / fake chat), and must never hold a turn or a wallet. Needs its own decision. |
| 6.16 | Edit static pages: rules, FAQ, agreements, privacy policy | **gap** | No content model. Terms exist only as a version number (`pc_terms_current_version`) that the gate compares — the terms *text* itself is not stored server-side. A small pages store (CPT or table, per language once 2.4 lands) + public read + admin editor. |
| 6.17 | All error messages and texts editable from the admin, in every language | **gap** | Error messages are English literals in PHP `WP_Error`s and in the SPAs. Depends on 2.4; scope decision needed — "every text" is the whole UI. |
| 6.18 | Google Analytics on all events; Amplitude | **gap** | No `gtag`, no analytics SDK, no event names. New dependencies (core rule 1) and a privacy/consent question. |
| 6.19 | Site banner | **struck out in the ТЗ** | Ignore. |
| 6.20 | Coin pricing, bonus map, machine on/off + sensors, withdrawals, top-ups, tickets, subjects, captcha, chat moderation | *(beyond the ТЗ)* | All built — the admin SPA's eleven screens (`DESIGN.md`). The ТЗ never asks for a bonus map or chat moderation; they stay. |

---

## 7. Summary

Counting the rows above by their first tag (n/a, struck-out and "beyond the
ТЗ" rows excluded; 5.2 counted as partial, 4.11 as differs):

| Status | Rows |
|---|---|
| done | 15 |
| partial | 14 |
| differs (a recorded decision contradicts the ТЗ) | 5 |
| gap | 25 |
| not verified as a whole (4.1) — plus the UI nuances flagged inside 2.1, 3.2, 4.3, 4.4, 4.12 | 1 |

The **core loop is real**: a verified player buys coins, queues, tosses a
physical coin, and the machine's payout reaches their wallet — over a push
channel, with operator alerts. What the ТЗ adds on top falls into five kinds
of work, in rough order of how much they change what exists:

1. **New rules on existing flows** — withdrawal minimum / multiple of 10,
   one queue per player, hidden rooms, global online counter, failed-top-up
   error code. Small, low risk, mostly backend.
2. **The turn choreography** — countdown, server-side auto-toss, the
   continue-or-collect dialog, the 15-s handover, the animations and sounds,
   the games tab. This is the biggest *conceptual* change because it
   collides with two settled facts: winnings credit the wallet immediately,
   and a payout is seen ~65 s late. It needs a spike-level decision before
   any screen is drawn.
3. **Identity** — phone + SMS (provider, codes, limits), password recovery
   by email and SMS, TOTP instead of (or beside) the email code, the welcome
   coin, email change, account deletion, Settings + notification channels.
4. **Operator surfaces** — players list and detail, blocked users, admin
   accounts, static pages, error texts, maintenance mode, demo users, melody
   upload, an admin stream preview, an error-log view.
5. **Cross-cutting platform work** — i18n (UA / EN / RU + admin-managed
   languages and strings), analytics (GA4 + Amplitude), stream recording via
   Mux, the watermark (venue-side), plus the launch blockers already on
   record: no Ably account, captcha unconfigured, Google sign-in parked,
   `admin/` has no deploy target or CI, `wp_mail` unproven on the host, the
   `BACKEND-REVIEW.md` security items (§§2–7), no automated tests in
   `frontend/` or `admin/`.

## 8. Conflicts the Feature chat must settle first

These are places where the ТЗ says one thing and a recorded decision says
another. Per the playbook they are settled by a `DECISIONS.md` entry (keep
the decision, or supersede it citing the ТЗ) — never by quietly building
the ТЗ version.

| ТЗ | Recorded decision | Settle |
|---|---|---|
| Second factor = Google Authenticator (players and admins) | Email 6-digit code (ROADMAP Phase 1 §1, Phase 2 §3) | Replace, add as an option, or keep email? |
| Phone with SMS verification is mandatory at sign-up | Phone verification deferred, field optional (ROADMAP Phase 2 §5) | Which SMS provider; who owns the account; the lockout numbers as options |
| Chat is not saved and not moderated | Persisted and moderated (`DECISIONS.md` 2026-09-07) | Keep the stronger behaviour? |
| Winnings sit in a per-turn counter until the player "collects" or "continues" | Machine payouts credit the wallet immediately, attributed by open session (`DECISIONS.md` 2026-07-24; `DOMAIN.md`) | Whether "continue" means re-staking won coins at the head of the queue, and how a 10-s window works with a 65-s payout delay |
| Auto-toss every 10 s when the player idles | Idle player is pruned after `pc_queue_idle_timeout_seconds` (`DECISIONS.md` 2026-07-30) | Auto-toss is a server job spending real money on the player's behalf — must be stated as a domain rule |
| Reset the machine's counter when the turn passes | The toss itself resets `sensor.coin`; no reset command exists (`DECISIONS.md` 2026-09-18) | Confirm with the venue |
| Withdrawal to a card attached in the payment system, with provider confirmation | Manual out-of-band payout; automated payouts and KYC out of v1 (`DECISIONS.md` 2026-05-13, 2026-09-16) | Stays manual, or a payout provider is a new feature with its own compliance question |
| Stream watermark with the player's nickname | Video is produced in the venue and pushed to Mux (`DOMAIN.md`) | Venue tooling, not the web app; scope it out explicitly or plan an overlay integration |
| Demo users create "fake activity" | A queue entry is a real turn; a toss is a real coin (`DOMAIN.md`) | Presentational only, or drop it |
| Three languages, admin-managed | No i18n, `LanguageSwitcher` inert (`DESIGN.md`) | Library choice is a new dependency (core rule 1) |

## 9. Open questions for the customer

- Google Authenticator: is a TOTP app the requirement, or "a second factor"?
- SMS: which provider, whose account, which countries? Is a phone still
  mandatory if SMS cost is a concern?
- What price does the free welcome coin carry when it is won back or withdrawn?
- "Continue playing" with winnings: re-stake at the head of the queue, or
  re-join at the back with the new balance? The 10-s window vs the ~65-s
  payout delay must be explained to the customer before designing.
- Withdrawals: does "attached card" mean a real payout provider is expected
  in v1, or is a manual bank transfer acceptable?
- Stream recording: who pays for Mux asset storage; is a download
  really needed, or is Mux's own dashboard enough for the operator?
- Watermark: who owns the venue encoder (OBS / mini-PC)?
- Demo users: what should a viewer see, exactly?
- Which of the 30 mockups are binding? The current SPA was built with no
  design files (`DESIGN.md`); a screen-by-screen visual comparison is not
  part of this audit and is the design question the Feature chat asks.
- Google / Apple sign-in are not in the ТЗ — keep, remove, or leave parked?

## 10. Suggested feature split for the Feature-mode chat

Candidates only — the Feature chat decides names, order and sprint slicing
(one feature per chat, per the playbook):

| Candidate feature | Rows | Touches shared code |
|---|---|---|
| `game-turn` — countdown, auto-toss, continue/collect, handover, sounds, games tab, one-queue rule | 4.7–4.13, 5.7 | `queue-service.php`, `RoomQueueController.php`, `PlaceBet.vue`, `stores/queue.js`, `wallet-service.php` (read) |
| `identity` — phone + SMS, password recovery, TOTP, welcome coin, email change, delete account, Settings | 3.1–3.7, 5.1, 5.3–5.5 | `UserController.php`, `AuthController.php`, `permissions.php`, `user-meta-keys.php`, `AccountView.vue` |
| `operator` — players list/detail, blocked users, admin accounts, error log view, maintenance mode, hidden rooms, admin stream preview, melody upload | 6.2, 6.4, 6.5, 6.7, 6.8, 6.10–6.15 | `admin/` (new views), `AdminRoomController.php`, `RoomController.php` |
| `content` — static pages, rules dialogs, editable error texts | 2.7, 6.16, 6.17 | new CPT/table; every SPA string |
| `i18n` — UA / EN / RU, admin-managed languages | 2.4 (prerequisite of `content`) | every `.vue` in both SPAs, API error messages |
| `withdrawals-v2` — min / multiple rule, SLA, payout provider (if any) | 5.2 | `wallet-service.php`, `WalletController.php`, `WithdrawalsView.vue` |
| `broadcast` — recording + retention + download (Mux API), stream health | 6.9, 6.10 (camera) | new integration, new secret |
| `analytics` — GA4 + Amplitude events | 6.18 | both SPAs, new dependencies |
| small rules with no natural home — global online counter, hidden rooms, failed-top-up error code | 2.1, 6.4, 5.6 | `/adhoc`, or the first sprint of the nearest feature |
| launch blockers (no new feature — `/adhoc` and config) | ROADMAP Phase 8 | — |

`i18n` before `content`; `identity` (phone) before the welcome coin; a
`game-turn` spike before its screens. Everything else is independent.
