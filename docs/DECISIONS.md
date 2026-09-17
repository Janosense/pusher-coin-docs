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

## 2026-09-16 — Stripe replaces LiqPay for top-ups; Stripe's prohibited-business policy is an accepted risk
- **Context:** The client asked for Stripe as the top-up provider and gave no reason. Discovery checked Stripe's own record before agreeing: the Prohibited and Restricted Businesses list (stripe.com/legal/restricted-businesses) names "games of chance including gambling, internet gambling, casino games, sweepstakes and contests … with a monetary or material prize" as prohibited; its FAQ extends that to "games of skill (or chance) with cash prizes" and names no country exception and no approval path; stripe.com/global does not list Ukraine among the countries where a business can open an account. Tymofii confirmed the risk is known to the client and the decision is to proceed.
- **Decision:** Replace LiqPay Checkout with Stripe as the only top-up provider, and treat Stripe's policy — a review that can close the account and hold its balance — as a business risk the client accepted, not as a defect of the feature.
- **Alternatives rejected:** Stripe next to LiqPay as a second provider — not what the client asked for, and two providers double the settlement surface in the money zone for no stated need. Stripe for withdrawals — out of scope; the 2026-05-13 decision (manual payouts, no KYC pipeline) stands. Declining the feature on policy grounds — the call is the client's, and it was made with the policy on the table.
- **Consequences:** The 2026-05-13 entry "LiqPay Checkout for top-ups" is superseded on the provider only; its provider-independent consequences carry over — the webhook is the only place a transaction reaches `completed`, idempotent on the provider's reference. The Stripe account, its country of registration and its owner are open until the client names them, and the feature cannot be verified end-to-end in live mode before that. There is no fallback provider: if Stripe closes the account, top-ups stop until another provider is wired. What happens to the LiqPay code and to `wp_pc_transactions` rows carrying LiqPay `external_ref`s is decided in Phase B of this feature.

---

## 2026-09-16 — LiqPay is removed, not parked
- **Context:** Stripe replaces LiqPay for top-ups (entry above). Tymofii confirmed the product is pre-launch — `ROADMAP.md` Phase 8 is open and the captcha is still a launch blocker — so no real LiqPay order is in flight when the switch ships.
- **Decision:** Remove the LiqPay integration entirely: the callback route, `liqpay-client.php`, `liqpayCheckout.js`, the `PC_LIQPAY_PRIVATE_KEY` constant, the public-key option and its Settings hint. History rows in `wp_pc_transactions` whose `external_ref` is a LiqPay `order_id` stay untouched.
- **Alternatives rejected:** Parking LiqPay behind configuration, like Google sign-in — two providers in the money zone to keep in mind, for a return nobody asked for. Keeping the callback alive for one more sprint to settle in-flight orders — there are none; the product has not launched.
- **Consequences:** Root `CLAUDE.md` (invariants 2 and 5, the test-critical zones line), `docs/DOMAIN.md` → Top-up, `docs/ARCHITECTURE.md` → Integrations, `docs/CONTRACTS.md` and `docs/DESIGN.md` → Settings and Flows all name LiqPay and change in the same commits as the code that removes it. Going back to LiqPay is a git revert plus a new feature, not a switch. "Nothing disappears" holds for the ledger: the player's History and the admin views keep rendering LiqPay-era rows.

---

## 2026-09-16 — Top-ups stay UAH end to end under Stripe
- **Context:** The Stripe account will belong to a non-Ukrainian legal entity, so settlement happens in that account's currency; the open question was what the player pays and what the wallet holds.
- **Decision:** The player sees and pays UAH; the wallet, the coin price, the lots, withdrawals and history stay UAH to the kopiyka; converting to the settlement currency is Stripe's business, not the product's.
- **Alternatives rejected:** Charging in the account's currency while the wallet stays UAH — needs an exchange-rate rule (who fixes it, when) that the domain has no owner for. Moving the wallet to another currency — rewrites the whole money zone for an unstated need.
- **Consequences:** Every amount sent to Stripe is UAH in minor units (kopiykas), derived from the `DECIMAL(12,2)` string, never from a float (invariant 7). Whether the account's country allows `uah` as a presentment currency could not be verified from the public docs — the list is per account country — and is a spike exit criterion; if it is not allowed, this decision is reopened. Stripe's minimum charge applies after conversion to the settlement currency, so the smallest top-up is bounded by the account, not by the coin price.

---

## 2026-09-16 — Chargebacks and refunds are out of v1
- **Context:** Stripe emits dispute and refund events. Today a LiqPay `reversed` arriving after settlement changes nothing in the product.
- **Decision:** v1 does nothing on a dispute or a refund — no wallet movement, no ledger row, no operator notice; the operator handles it in the Stripe Dashboard and out of band, as today.
- **Alternatives rejected:** Recording the event and letting the operator decide — a new admin state and a new transaction status for a case the operator has not yet seen. Debiting unplayed coins automatically — a new domain rule in the money zone (what if the coins are already played?), decided without the operator.
- **Consequences:** The webhook handler acknowledges dispute and refund events with 200 and ignores them — never a non-2xx that would make Stripe retry. Every sprint of this feature lists it under Out of scope; a later feature reopens it with its own `DECISIONS.md` entry.

---

## 2026-09-17 — `stripe` has no UI design; the payment page is Stripe's
- **Context:** The playbook asks once per feature whether its UI is designed before coding. This feature changes two screens (Replenishment balance, Settings) and adds one (an admin list of top-ups).
- **Decision:** No design. The payment page is the one Stripe hosts, used as it comes; Replenishment balance and Settings keep their existing components; the admin Top-ups screen is built from the Withdrawals screen's layout and components.
- **Alternatives rejected:** Claude Design artboards for Top-ups — a filterable status table the admin SPA already has in Withdrawals. Designing the payment page — Stripe's hosted page is not ours to design.
- **Consequences:** `docs/features/stripe/design/` stays empty; `FEATURE.md` → UI names the three screens; `docs/DESIGN.md` gains a Top-ups row under Screens and its Top-up flow changes when the code lands; no new tokens or shared components. A later feature that adds a screen asks the question again for itself.

---

## 2026-09-17 — Top-ups go through Stripe Checkout's hosted page
- **Context:** The payment page is Stripe's (entry above) and the amount is computed per player — `coin_qty × unit_price` — so it cannot be a fixed price.
- **Decision:** `POST /wallet/topup` creates a Checkout Session server-side — `mode=payment`, one line item "N coins @ price", `currency=uah`, the amount in kopiykas derived from the `DECIMAL(12,2)` string, `client_reference_id` = the transaction id, `success_url`/`cancel_url` on the SPA's `/account?topup=success|cancel` — and returns the session's `url`; the SPA navigates to it.
- **Alternatives rejected:** Payment Element (embedded form) — Stripe.js and a publishable key in the SPA, and more code for a page we chose not to design. Payment Links — fixed amounts; ours is computed per player. Fulfilling from the landing page as well (Stripe's optional pattern) — Checkout waits up to 10 s for the `checkout.session.completed` response before redirecting, so the wallet is normally credited before the player returns.
- **Consequences:** No Stripe.js, no publishable key anywhere, PCI scope unchanged. `checkout_url` replaces the LiqPay envelope in the `POST /wallet/topup` response. `?topup=success` stays a UX cue, never a source of truth. A session that expires gives the pending row a terminal state through `checkout.session.expired` — LiqPay never had one.

---

## 2026-09-17 — Stripe is called without an SDK
- **Context:** `backend/wp-content/themes/pc/vendor/` is git-ignored and CI FTP-syncs the checkout without `composer install`, so a Composer dependency does not reach production by itself. Core rule 1 requires approval for any new dependency; `liqpay-client.php` (85 lines) is the precedent for a hand-written client.
- **Decision:** A hand-written client on `wp_remote_post` — create and retrieve a Checkout Session, a pinned `Stripe-Version` constant — and manual webhook signature verification exactly as Stripe documents it: split `Stripe-Signature` into `t` and `v1` values, HMAC-SHA256 of `{t}.{raw body}` with the endpoint secret, constant-time comparison, `v1` only, 300 s tolerance.
- **Alternatives rejected:** `stripe/stripe-php` — correct, but needs a deploy-pipeline change (Composer in CI or a manual `vendor/` upload) before it can ship. A WordPress Stripe plugin — the theme can do it (`TECH-STACK.md` → ANTI-PATTERNS).
- **Consequences:** Signature verification is ours to test — it is a test-critical zone. The raw request body must reach the verifier unmodified. The pinned API version is set by the spike and bumped deliberately, never implicitly. Adopting the SDK later starts with a deploy decision, not a `composer require`.

---

## 2026-09-17 — The Stripe webhook settles top-ups the way the LiqPay callback did
- **Context:** Invariant 5 of the root `CLAUDE.md` names the LiqPay callback as the only place a transaction reaches `completed`. Stripe delivers events at least once, in no guaranteed order, and retries non-2xx for up to three days.
- **Decision:** `POST /pc/v1/payments/stripe/webhook` — public route, the signature is the credential. `checkout.session.completed` and `checkout.session.async_payment_succeeded` with `payment_status = paid` → `Wallet_Service::settle_topup`; `checkout.session.async_payment_failed` and `checkout.session.expired` → `failed` with a note; every other event → 200, ignored. The row is found by `external_ref` = the session id, written through a new `Wallet_Service` setter right after creation (the `$wpdb->update` bypass in `WalletController::topup` moves into `Wallet_Service` in the same step). Settlement runs only from `pending`, after the event's `amount_total` and `currency` match the row; the event id goes into `Audit_Log` metadata.
- **Alternatives rejected:** Settling on `payment_intent.succeeded` — the session is the object we created and reference. Retrieving the session from the API before settling (the docs' generic pattern) — the event carries the same session object and we verify it against our own row; an extra call per event buys nothing. Returning 4xx/5xx for an unknown or already-settled session — that only makes Stripe retry; 200 with a `note`, as LiqPay did.
- **Consequences:** Root `CLAUDE.md` invariant 5 and the test-critical zones line are rewritten to name this route. A non-2xx leaves the handler only for a missing or invalid signature. Duplicate deliveries are traceable by event id in the audit log.

---

## 2026-09-17 — Stripe configuration: two wp-config constants, a derived status in Settings
- **Context:** Hosted Checkout needs no publishable key; the only secrets are the API key and the endpoint signing secret. The operator asked to see Stripe in Settings.
- **Decision:** `PC_STRIPE_SECRET_KEY` and `PC_STRIPE_WEBHOOK_SECRET` are wp-config constants; `pc_liqpay_public_key` is removed with a `pc_db_version` bump. The server exposes only `configured` (both constants present) and `mode` (`test` or `live`, from the key prefix); Settings shows them, red when unconfigured. Unconfigured → `POST /wallet/topup` answers `stripe_not_configured` 500. Development runs on test keys Tymofii provides; the client's account is still unnamed.
- **Alternatives rejected:** A publishable-key WP option — nothing reads it with hosted Checkout. Showing a fragment of a key as a hint — nothing but presence and mode leaves the server.
- **Consequences:** Until the client names its account, production can only carry test-mode keys, and the `test` badge in Settings is the visible warning. The webhook endpoint is registered in the Stripe Dashboard by hand per environment; its secret differs between test and live.

---

## 2026-09-17 — The admin gets a read-only Top-ups list
- **Context:** The operator asked to see players' top-ups in the product, not only in the Stripe Dashboard. The Withdrawals screen already lists ledger rows with status filters.
- **Decision:** `GET /pc/v1/admin/topups?status=pending|completed|failed|all&page&per_page` mirrors `admin/withdrawals` (player, amount, coins, unit price, status, `external_ref`, notes, dates); a **Top-ups** screen at `/topups` in the admin SPA, built from the Withdrawals layout, with no actions.
- **Alternatives rejected:** Stripe Dashboard only — the operator asked for it here. Actions on a row (refund, block) — chargebacks and refunds are out of v1.
- **Consequences:** Rows of any provider render — LiqPay-era `pc-topup-N` refs next to `cs_…` ones. `docs/CONTRACTS.md` and `docs/DESIGN.md` → Screens gain the endpoint and the screen. The admin SPA still has no deploy target, so the screen is local-only until Phase 8 gives it one.

---

## 2026-09-17 — `stripe` code lives in `app/stripe/`, registered by one bootstrap line
- **Context:** `core` is frozen; `realtime` set the pattern of a feature directory with a `bootstrap.php` required once from `functions.php`.
- **Decision:** `backend/wp-content/themes/pc/app/stripe/` (`bootstrap.php`, `stripe-client.php`, `StripeWebhookController.php`, `AdminTopupController.php`); `admin/src/views/TopupsView.vue` + `admin/src/services/adminTopupService.js`. No new file in the player SPA — `ReplenishmentBalance.vue` navigates to the returned `checkout_url`. `WalletController::topup` changes in place.
- **Alternatives rejected:** Adding the webhook to `core`'s `PaymentController` — the frozen record would grow new work. A player-SPA `stripeCheckout.js` service — a one-line navigation needs no module.
- **Consequences:** One Features-table row and one `docs/ARCHITECTURE.md` feature-map row; `docs/PROJECT-TREE.md` changes in the step that creates the directory. Everything the feature touches outside `app/stripe/` and the two admin files is `core` shared code.

---

## 2026-09-17 — Webhook checks are a WP-CLI eval script; local delivery is the Stripe CLI
- **Context:** Interim money checks are WP-CLI eval scripts (2026-09-15), picked up by `backend/bin/check`. Stripe cannot reach a DDEV host, but its CLI can forward events to one.
- **Decision:** `wp-content/themes/pc/tests/stripe-webhook.php` signs its own fixtures (the verifier takes the secret as a parameter) and checks: valid, invalid and stale signatures; a re-delivered event settles once; an amount or currency mismatch settles nothing; `expired` → `failed`; a dispute event → 200 and no change. Locally, `stripe listen --forward-to https://pusher-coin.ddev.site/wp-json/pc/v1/payments/stripe/webhook` delivers real events; the Stripe CLI is developer tooling, recorded in `docs/TECH-STACK.md`, not a project dependency.
- **Alternatives rejected:** PHPUnit — the 2026-09-15 decision stands. An ngrok tunnel — the CLI needs no public URL.
- **Consequences:** The script runs on every `backend/bin/check` with DDEV up. The CLI's `whsec_` differs from the Dashboard's — the local wp-config carries the CLI's.

---

## 2026-09-17 — The check command reaches `stripe/sprint-1` as copied files, not as a cherry-pick onto `main`
- **Context:** `backend/bin/check`, `frontend/bin/check` and the report-only lint scripts exist only on `realtime/sprint-1` (its Step 1); `stripe/sprint-1` is cut from `main`, where `docs/TECH-STACK.md` still says there is none, and the playbook requires the first code step of a sprint to have a check command. Tymofii declined cherry-picking the four realtime commits onto `main`: a push of `backend` `main` is a production release, and `main` moves only at sprint boundaries.
- **Decision:** Sprint 1 Step 1 copies the files byte-for-byte from `realtime/sprint-1` into `stripe/sprint-1` — `backend/bin/check`, `frontend/bin/check`, the `lint` / `lint:fix` scripts in both `package.json`, and the matching sections of `docs/TECH-STACK.md`, `CLAUDE.md` → Commands and `docs/PROJECT-TREE.md`. `main` receives them at whichever sprint boundary comes first.
- **Alternatives rejected:** Cherry-picking onto `main` — declined: an out-of-boundary production push. Writing a second check command — two different files at one path conflict when the branches meet. Working without one until `realtime` merges — the sprint's money zone is exactly what the gate is for.
- **Consequences:** Identical content merges cleanly whichever branch lands first. If `realtime` changes its check command before then, the copy is refreshed on the stripe branch through `/adhoc`, never diverged. Until a boundary, `main` still has no check command.

---
