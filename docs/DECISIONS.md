# Decisions log — Pusher Coin

<!-- One decision log per project, across ALL features and code areas.
     Append-only; a reversed decision gets a NEW entry that links the old one,
     the old entry is never edited. Read this file before proposing any architecture or
     tooling change — it may already be decided. -->

Entry format:

## {{YYYY-MM-DD}} — {{Short title}}
- **Context:** {{what forced the decision}}
- **Decision:** {{what was decided, one sentence, imperative}}
- **Alternatives rejected:** {{and the one-line reason each lost}}
- **Consequences:** {{what this commits us to; what becomes an anti-pattern}}

---

<!-- Adoption note: the entries below were reconstructed at playbook adoption
     from the pre-playbook documentation (the former root ARCHITECTURE.md,
     DATA-MODEL.md, ROADMAP.md and ADMIN-DECISION.md) and dated by the commit
     that recorded each one. -->

## 2026-05-05 — Three separate repositories, one documentation root
- **Context:** The product is a WordPress backend plus two Vue SPAs with different deploy targets and cadences.
- **Decision:** Keep `backend/`, `frontend/` and `admin/` as independent git repositories; the root repository ignores all three and versions only the documentation and the playbook.
- **Alternatives rejected:** A single monorepo — three unrelated deploy pipelines (FTP, Vercel, none) and three toolchains would have to share one history and one CI. Git submodules — the pinning ceremony buys nothing when the three are released independently.
- **Consequences:** A change spanning two apps is two commits in two repositories, with the same branch name in each and a merge at the sprint boundary. The root repository can never run a cross-app check command or CI. Cross-app consistency is a documentation duty, not a tooling guarantee.

---

## 2026-05-07 — Admin surfaces are a separate Vue SPA, not WP admin
- **Context:** Room scheduling, bonus mapping, the machine power switch, withdrawal review, support triage and a future operations dashboard all need an operator UI. The roadmap's own recommendation was to extend WordPress admin.
- **Decision:** Build every operator surface as a separate Vue 3 SPA (`admin/`) consuming a `pc/v1/admin/*` namespace, and put no product workflow in `/wp-admin/`.
- **Alternatives rejected:** Extending WP admin with custom screens — a real-time multi-panel operations view is a poor fit for it, and its nonce + cap + screen-hooks model is more surface area for the same outcome. ACF — flexible content modelling is not the problem being solved; revisit only if a feature genuinely needs it.
- **Consequences:** A third deploy target and a third `.env` are accepted (still unbuilt). Vue tooling and primitives are duplicated between the two SPAs on purpose; no shared package until the duplication actually hurts. Every admin endpoint gates on `manage_options` through one callback. `/wp-admin/` remains available for plugin, theme and emergency DB work only. Introducing an admin screen in `/wp-admin/` is now an anti-pattern.

---

## 2026-05-07 — Model active refresh tokens instead of blacklisting access JWTs
- **Context:** Logout and revocation need to be real, but access JWTs are stateless and short-lived.
- **Decision:** Store refresh tokens hashed in `wp_pc_refresh_tokens`, rotate on every `/auth/refresh`, and revoke the whole descendant chain on reuse detection.
- **Alternatives rejected:** A `wp_pc_jwt_blacklist` of access tokens (reserved in earlier drafts) — it would grow with every sign-out and still only shorten a 15-minute window.
- **Consequences:** The access token stays opaque to the SPA and is never revoked directly; revocation is always a refresh-token operation. The plaintext refresh token is never stored, only its SHA-256.

---

## 2026-05-07 — Secrets are wp-config constants, never database rows
- **Context:** The backend holds the machine bearer token, the LiqPay private key, the JWT signing key, the captcha secret and the Apple key, while operators need to edit their public counterparts from the admin SPA.
- **Decision:** Keep every secret as a constant in `wp-config.php` on the server; store only public counterparts (LiqPay public key, captcha site key, machine entity ids) as WP options.
- **Alternatives rejected:** WP options for secrets — they land in DB backups, `wp db export` and any future admin screen. An `.env` file in the repository — it is the repository.
- **Consequences:** Rotation is a wp-config edit on the server, not an admin-SPA action, and cannot be automated from the product. Anything that reads a secret from `get_option()` is a defect.

---

## 2026-05-07 — Apple Sign-In ships as a stub
- **Context:** Apple Developer Program enrollment was not complete and the auth surface needed to be finished.
- **Decision:** Ship `AppleAuthController` returning `apple_not_configured` until the Apple constants are populated, and hide the button in the SPA.
- **Alternatives rejected:** Waiting for enrollment — it would have blocked the whole auth phase. Removing Apple entirely — the flow mirrors Google closely enough that deleting and re-adding it costs more than the stub.
- **Consequences:** The Apple meta keys and endpoints exist and are dead code until enrollment. Enabling Apple is a configuration change, not a code change.

---

## 2026-05-13 — Mux LL-HLS, played through a lazy-loaded `hls.js`
- **Context:** The venue streams over RTMP and players need low-latency playback in the browser.
- **Decision:** Ingest RTMP into Mux and play back LL-HLS; load `hls.js` by dynamic import only when an `.m3u8` URL is about to play, and let Safari use its native HLS.
- **Alternatives rejected:** WebRTC — lower latency, far more infrastructure than a one-machine venue justifies. Bundling `hls.js` statically — ~162KB gzipped on every page load for a library most page views never need.
- **Consequences:** `LiveStream.vue` sniffs the URL to choose between iframe, `<video>` and HLS, so one-off YouTube/Vimeo events still work. The playback URL is per-room post meta, not configuration.

---

## 2026-05-13 — Coins are FIFO lots that remember their purchase price
- **Context:** The coin price is operator-tunable, and a coin won has to be worth what the player paid for it — not what the price happens to be later.
- **Decision:** Store coins as a stack of `(qty, unit_price)` lots consumed oldest-first, and price every debit, payout and refund from the lot.
- **Alternatives rejected:** A single coin balance with a global current price — an operator raising the price would retroactively change what the system owes every player. Averaging the price across a wallet — cheaper to compute, impossible to explain to a player disputing a payout.
- **Consequences:** Every money path goes through `Wallet_Service` under `SELECT … FOR UPDATE`. Lot rows are never deleted, only drained to `qty = 0`. A withdrawal records the lots it consumed so a rejection can re-credit at the original prices. Machine payouts are priced at the FIFO-head lot price.

---

## 2026-05-13 — LiqPay Checkout for top-ups; withdrawals are manual
- **Context:** Ukrainian players paying in UAH, and no legal requirement yet for an automated payout pipeline.
- **Decision:** Use LiqPay hosted Checkout for top-ups, settle through the signed callback, and handle withdrawals as an operator approving and paying out of band.
- **Alternatives rejected:** Stripe Connect-style automated payouts with a KYC pipeline — deferred until legal requires it. Taking card details in the SPA — puts the product in PCI scope for no benefit.
- **Consequences:** The callback is the only place a transaction reaches `completed`, and it must stay idempotent on `(order_id, status)`. A player may have one pending withdrawal at a time. There is no automated KYC anywhere in the system.

---

## 2026-05-13 — `VARCHAR` plus PHP constants instead of `ENUM` columns
- **Context:** Several tables need a small closed set of statuses and types, and the schema is installed with `dbDelta`.
- **Decision:** Declare every status/type column as `VARCHAR`, keep the allowed values as class constants, and validate on write in the controller.
- **Alternatives rejected:** `ENUM` — `dbDelta` cannot diff an `ENUM` definition reliably, so adding a member later silently skips the migration and the column quietly rejects the new value in production.
- **Consequences:** The database enforces nothing about these columns; the constants and the controller do. Applied to `wp_pc_transactions`, `wp_pc_machine_events`, `wp_pc_support_tickets`, `wp_pc_room_messages` and `wp_pc_room_schedules`. Adding an `ENUM` anywhere is now an anti-pattern.

---

## 2026-07-24 — Machine payouts credit coin lots, never the ledger
- **Context:** The machine pays out constantly, while the player's history view is meant to show money movements they would recognise.
- **Decision:** Credit machine payouts as coin lots plus a `balance_coins` move, audited in `wp_pc_machine_events`, and keep `wp_pc_transactions` for top-ups and withdrawals only.
- **Alternatives rejected:** A ledger row per payout — it would bury the two transaction types the player cares about under a stream of machine noise, and itemise individual tosses the product deliberately does not show.
- **Consequences:** Two separate audit trails that must not be conflated: money in `wp_pc_transactions`, machine activity in `wp_pc_machine_events`. Any reconciliation has to read both.

---

## 2026-07-24 — `event_key` is the machine-event idempotency guard
- **Context:** No inbound transport had been chosen yet, and both plausible ones (a Home Assistant webhook, or polling) can deliver the same event twice.
- **Decision:** Give `wp_pc_machine_events` a unique `event_key` and require every transport to supply one; a colliding event is recorded once and credited once.
- **Alternatives rejected:** Deduplicating on `(machine_id, event_type, timestamp)` — sensor timestamps are not reliable enough to be a key. Trusting the transport to deliver exactly once — none of the candidates does.
- **Consequences:** `event_key` is nullable so keyless events still log, which means a transport that omits it gets at-least-once delivery and double credits. Choosing a transport includes choosing its key.

---

## 2026-07-25 — An unconfigured captcha is the off switch
- **Context:** The support form needed guest anti-abuse, but there was no Turnstile or hCaptcha account yet.
- **Decision:** Treat an empty provider/site-key/secret as "captcha disabled", let the guest path run unchallenged in that state, and surface it in red in the admin SPA.
- **Alternatives rejected:** Commenting out the captcha code — it rots, and re-enabling becomes a code change instead of a configuration change. Blocking guest tickets entirely until a captcha exists — it would have shipped a support form nobody can use.
- **Consequences:** The guest support path is currently unprotected, and that is a launch blocker tracked in `ROADMAP.md` and `backend/wp-content/themes/pc/CAPTCHA_SETUP.md`. Nothing about the captcha may be switched by editing code.

---

## 2026-07-28 — Google sign-in parked behind an empty client id
- **Context:** The Google flow is implemented end to end but the OAuth credentials were not in place.
- **Decision:** Leave the backend untouched and hide the button while `VITE_GOOGLE_CLIENT_ID` is empty; restoring it is a configuration edit.
- **Alternatives rejected:** Removing the code — it works and re-adding it would cost more than leaving it dormant.
- **Consequences:** Same rule as the captcha: the empty config *is* the switch, and nothing is commented out. Both parked integrations are launch items.

---

## 2026-07-30 — The room queue is persisted, and pruned on read
- **Context:** The turn decides who gets paid when the machine pays out, and machine events can arrive seconds after a player's tab is backgrounded.
- **Decision:** Persist the queue in `wp_pc_room_queues`, treat the SPA's 3-second queue poll as the heartbeat, and prune stale entries on every read.
- **Alternatives rejected:** In-memory or presence-channel state — it does not survive a page reload or a backend restart, and a lost turn means a payout credited to nobody. A cron job for pruning — it adds a moving part for housekeeping that read traffic already does; a room with no readers has nothing to prune anyway.
- **Consequences:** `coins_declared` is an intent, not a reservation — coins are debited one per toss. At most one open bet session per room (`ended_at IS NULL`) is the invariant that makes attribution possible. Adding a cron for queue housekeeping is now an anti-pattern.

---

## 2026-09-07 — Chat: gated below play, cursor-read, moderated by hiding
- **Context:** Guests watching a broadcast should see the conversation, players should be able to talk without having verified their email, and moderation has to survive a complaint review.
- **Decision:** Read `GET /rooms/{id}/messages` publicly with an `after` cursor polled every 3 seconds; write through `require_chat_ready` (terms + nickname, no email verification); moderate by flipping a `status` column and by an account-wide timed mute in user meta.
- **Alternatives rejected:** Requiring `require_play_ready` to chat — chat moves no coins, so the email gate buys nothing. `LIMIT/OFFSET` paging — a conversation that grows between two polls would re-send or skip messages. Deleting moderated messages — the author, body, IP and timestamp are exactly what a complaint review needs. A nickname column on the message — a renamed player would leave two names in one conversation. A per-room mute — someone worth silencing in one room is worth silencing in the next.
- **Consequences:** Chat works in a room under maintenance on purpose — that is where players ask what is going on. `wp_pc_room_messages.room_status_id` is the index the cursor read depends on.

---

## 2026-09-15 — Playbook v1.16 adopted into the existing codebase
- **Context:** The project had grown a good but ad-hoc documentation set at the repository root and a `CLAUDE.md` that mostly listed upkeep duties. Work needed a repeatable step protocol.
- **Decision:** Adopt the playbook: create a root `CLAUDE.md` from `templates/CLAUDE.md`, move the existing root documentation into `docs/` under the playbook's names, register the whole existing codebase as the single feature `core`, and run all new work as new features through Feature mode.
- **Alternatives rejected:** Leaving the documentation at the repository root with `docs/` holding pointers — two places of truth and near-empty skeletons. Splitting the existing code into several features retroactively — it would be a reorganisation of working code for no functional gain, and the playbook explicitly wants `core` frozen as the as-is record. A code-area `CLAUDE.md` per repository — one area (the repository root) was chosen instead, so the root `CLAUDE.md` is the area's rules as well.
- **Consequences:** `API-CONTRACT.md` is now `docs/CONTRACTS.md`; `ADMIN-DECISION.md` became the 2026-05-07 entry above and the file is gone; `ROADMAP.md`, `INVENTORY.md`, `PROJECT-TREE.md` and `PUSHER-COIN-COMMANDS.txt` live in `docs/` as project-specific references with rows in the root `CLAUDE.md` table. The verification profile is **user-verified**, so every step writes a plain-language verification guide. The git model is chained sprint branches per repository. No new work is ever added to `core`. The stale `frontend/CLAUDE.md`, which described an unrelated workspace, was deleted.

---

## 2026-09-15 — Interim money checks are WP-CLI eval scripts, not PHPUnit
- **Context:** Review item 1 (`Wallet_Service` never rolled back, because `$wpdb` does not throw) sits in a test-critical zone, and the project has no test tooling; the real check command is deferred to the first new feature's Sprint 1.
- **Decision:** Until that check command lands, cover a test-critical fix with a dependency-free script under `backend/wp-content/themes/pc/tests/`, run by hand as `ddev wp eval-file …` against the local DDEV database, which injects faults through WordPress's `query` filter and refuses to run outside WP-CLI on DDEV.
- **Alternatives rejected:** PHPUnit with the WordPress test suite — a new dependency plus a test-database harness, too large for an ad-hoc and the job of the check-command step. A `wp pc` CLI command — it would ship a data-writing test into the production command set.
- **Consequences:** These scripts are not run by CI (it has no database) and are not the check command. The step that creates the check command decides whether to port them into a real suite or call them from it. Every script in `tests/` must keep the WP-CLI + DDEV guard, because the whole backend tree is FTP-deployed.

---

## 2026-09-15 — Realtime machine events become the feature `realtime`; `core` stays frozen
- **Context:** The 2026-09-15 ROADMAP audit left one coherent block of genuinely unbuilt work — the inbound machine-event transport and everything it unblocks (the SPA relay lock, live winnings, ops alerts, and Phase 6's "in real time"). It needed a home, and `core` was registered at adoption as a frozen as-is record that takes no new work.
- **Decision:** Open a new feature `realtime` for the machine-event transport and its consumers; leave `core` frozen and keep using `/adhoc` for defects inside it.
- **Alternatives rejected:** A sprint inside `core` — it would reverse the adoption decision three weeks after it was made and turn the as-is record into a backlog, so the one thing that says what the codebase *was* would stop saying it. Splitting the work across `core` and a new feature — attribution, transport and the SPA consumers are one causal chain; cutting it in two puts the sprint boundary inside a race condition. One big feature covering launch readiness too — i18n, a11y, compliance and the admin deploy target share no code and no blocker with this.
- **Consequences:** `core` keeps no `sprints/` folder and `/plan-step` still never targets it. Work in `realtime` that changes `core`'s shared code (the queue store's poll, `RoomChat`'s poll, `PlaceBet`) is a plan task marked "touches shared code". Phase 8 launch-readiness items stay unhoused until someone opens a feature for them.

---

## 2026-09-15 — The browser channel is Ably, on its free tier
- **Context:** The backend is a WordPress tree FTP-synced to shared hosting, which cannot hold a long-lived process, so the push leg from server to browser has to terminate somewhere else. Three 3-second polls (queue, chat, admin machine state) are waiting to be replaced.
- **Decision:** Publish machine and relay events from WordPress to Ably over HTTP and let the SPAs subscribe; start on the free tier (6M messages/month, 200 concurrent connections, 200 channels).
- **Alternatives rejected:** Pusher Channels Sandbox — half the concurrent connections (100) and a daily message cap rather than a monthly one, for the same money. Self-hosted Soketi — free, but a fourth deploy target to run and watch for a project that has not yet built its third. SSE or WebSocket straight from WordPress — the FTP shared host cannot hold the process; this is the constraint that rules out the whole family. Keeping the polls — they are what the feature exists to remove, and they already cause a duplicate-session race.
- **Consequences:** A fourth external service with credentials: the Ably key is a wp-config constant like every other secret, never a WP option. The SPA needs a token endpoint rather than the key. Free-tier ceilings become a launch checklist item — 200 concurrent connections is the number to watch. Publishing is fire-and-forget: a failed publish must never fail the money path that triggered it.

---

## 2026-09-15 — How machine events reach WordPress is deferred to a timeboxed spike
- **Context:** Home Assistant's official docs settle what is *possible* — a WebSocket API at `/api/websocket` that a long-lived token can authenticate against and subscribe to `state_changed`, and a REST API with no event stream at all — but not what is *usable here*. The walk-through with the machine owner that was meant to answer this will not happen.
- **Decision:** Do not pick the transport from the desk. Sprint 1 opens with a timeboxed spike against the live machine whose exit criteria are: whether `GET /api/states/<entity>` carries `last_changed`; whether the long-lived token authenticates on `/api/websocket`; whether we can add an automation in that Home Assistant; and how `sensor.coin`, `sensor.lc01_12` and `sensor.relay_on` actually behave across a real toss, a real bonus and a real relay close. The spike ends in a DECISIONS entry naming the transport; its code is throwaway.
- **Alternatives rejected:** Committing to cron polling now — `sensor.lc01_12` is a *level*, not an event, so two identical consecutive bonuses are indistinguishable to a poller and the second one silently pays nobody. That is a money defect, and whether `last_changed` rescues it is exactly what is unknown. Committing to an HA-side webhook now — it is the cheapest option and needs no process anywhere, but it depends on admin access to someone else's Home Assistant, which is unconfirmed. Committing to a WebSocket worker now — the most reliable, and the only one that certainly works, but it buys a fourth deploy target before cheaper options have been ruled out.
- **Consequences:** Sprint 1 cannot fully plan its transport step until the spike closes; that step is named with its acceptance criteria and planned by `/plan-step` afterwards. Whatever wins must produce an `event_key` — the unique index is the only thing standing between a retry and a double credit.

---

## 2026-09-15 — `realtime` has no UI design
- **Context:** The playbook asks once, per feature, whether the UI should be designed before coding.
- **Decision:** No design. The feature adds no screen: it changes the behaviour of components `core` already owns — `PlaceBet` gains a pre-emptively disabled state, `UserControls` updates winnings as they land, and the queue and chat stop polling.
- **Alternatives rejected:** Designing the new states as artboards — three disabled/error states on existing controls do not need a design pass, and `docs/DESIGN.md` already records the component inventory they belong to.
- **Consequences:** `docs/features/realtime/design/` stays empty and `FEATURE.md` → UI points at the `core` screens it changes. A later feature that adds a screen asks the question again for itself.

---
