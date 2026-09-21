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
## 2026-09-17 — Spike: Stripe accepts `uah`, on a provisional US sandbox; pin `2026-06-24.dahlia` and turn Adaptive Pricing off
- **Context:** `stripe` Sprint 1 Step 2, the timeboxed spike. Step 3 must not start on a guess, and the decisive question was whether Stripe accepts `uah` at all. **Account used: `acct_1TtSrOElMyJqvLDl`, display name "Ask Debt Pros Sandbox sandbox", country `US`, `default_currency` `usd`, `charges_enabled: false`, `sk_test_` key expiring 2026-10-13** — the Stripe CLI on the development machine was already logged into it. This is **not** the client's account (the client's is still unnamed, `FEATURE.md` → Purpose), so every number below is evidence from a US sandbox, not from the account that will run production. Approved as the provisional account for this spike (Sprint 1 Step 2 plan, Question 1, resolved as recommended).
- **Decision:** Build Step 3 on these observations.
  - **`uah` is accepted.** `POST /v1/checkout/sessions` with `line_items[0][price_data][currency]=uah` and `unit_amount=12000` answered **HTTP 200** with `"currency": "uah"`, `"amount_total": 12000`, `expires_at` 24 h out, and a `cs_test_…` id. The hosted page rendered **“UAH 120.00”** under “Pusher Coin top-up — 3 coin(s) @ 40.00 UAH”. Kopiykas from the `DECIMAL(12,2)` string is the right unit, as decided.
  - **Pin `Stripe-Version: 2026-06-24.dahlia`** — the account default, and the version every event observed here carried (`stripe listen` reported the same). The latest dated version is `2026-08-26.dahlia`; it is **not** pinned, because no evidence in this entry was gathered on it. Upgrading is a deliberate later change, not a default.
  - **Send `adaptive_pricing[enabled]=false` when creating the session.** Stripe enabled Adaptive Pricing on its own (`"adaptive_pricing": {"enabled": true}` in the response, never requested). With a Ukrainian billing address it stayed inert — `currency_conversion` was `null` and `amount_total` came back exactly as sent — but a buyer in another country can have the price converted, which populates `currency_conversion`, and Step 4 compares `amount_total` and `currency` against the pending row. Left on, that is a silent `amount_mismatch` for a legitimate payment and a breach of invariant 1 in spirit.
  - **Verify the `v1` entry by name.** The header arrived as **three** parts — `t=…,v1=…,v0=…`. A verifier that splits on `,` and takes the second field happens to work; one that takes the last field does not. Parse keys, collect every `v1`, and compare each.
  - **Do not assume event ordering.** The six events of one payment did not arrive in creation order: `payment_intent.created` (created `…081`) landed **after** `checkout.session.completed` (created `…082`). Step 4's idempotency must rest on state, not on arrival sequence.
- **Alternatives rejected:** Pinning `2026-08-26.dahlia` — newer, but nothing here was observed on it. Reading the account country from the Dashboard — `GET /v1/account` answers it, and the response's `Stripe-Version` header gives the account default without a login. Waiting for `expires_at` to lapse — its minimum is 30 minutes, so the spike used `POST /v1/checkout/sessions/{id}/expire`, which expired the session instantly. Leaving Adaptive Pricing on and widening Step 4's amount check — that would weaken the one check that protects the ledger.
- **Consequences:**
  - **The `uah` answer is provisional and must be re-verified on the client's real account** before go-live; it binds to a US sandbox. Note also that *presentment* is not *settlement*: a US account presents UAH but settles in USD. The product's invariant is only about what the player is charged and what the wallet records, both of which stayed `uah`/`12000`.
  - Proved end to end, against a real delivery: `HMAC-SHA256` over `{t}.{raw body}` with the CLI's `whsec_` **matches the `v1` entry**; a body altered by one byte and a wrong secret were both rejected. A 300 s tolerance is comfortable — the signature was 52 s old when checked, and delivery took 0.88 s.
  - Measured delivery, `stripe listen` → DDEV: `checkout.session.completed` **0.88 s** after the event, `checkout.session.expired` **0.48 s**. One card payment produced six events: `payment_intent.created`, `charge.succeeded`, `payment_intent.succeeded`, `checkout.session.completed`, `charge.updated`, and later `checkout.session.expired` for the second session. Step 4's "any other type → 200 `note: ignored`" branch will carry most of this traffic.
  - `checkout.session.expired` carries `status: expired`, `payment_status: unpaid`, and the original `amount_total`/`currency`/`client_reference_id` — enough for Step 4 to mark the row `failed` without a second API call.
  - **Idempotency replays the original response, not current state.** Re-sending the create with the same `Idempotency-Key` returned the same `cs_test_…` id but the *cached* body, still `payment_status: unpaid, status: open`, although the session had been paid. `pc-topup-{txn_id}` is therefore safe against creating two sessions for one transaction, but its response must never be read as live state.
  - `stripe listen --forward-to https://pusher-coin.ddev.site/…` accepted DDEV's locally-trusted certificate with **no `--skip-verify`**, and `charges_enabled: false` did not block test-mode payment. The session offered `payment_method_types: ["card", "link"]`.
  - The CLI's `whsec_` is per-`listen`-session and is not the Dashboard's; production registers its own. The sandbox's `sk_test_` expires **2026-10-13**, before which Step 3's manual verification needs a fresh key or the client's account.

---
## 2026-09-17 — A settlement that cannot be written answers 500, so Stripe retries
- **Context:** The 2026-09-17 entry "The Stripe webhook settles top-ups the way the LiqPay callback did" ends "A non-2xx leaves the handler only for a missing or invalid signature", and Sprint 1 Step 4's text repeats it. Building the handler surfaced a case neither had considered: `Wallet_Service::settle_topup` returns `false` when a write fails — the transaction rolls back and the row stays `pending` (proved by `wallet-rollback.php`). Under the rule as written the webhook would answer 200, Stripe would stop delivering, and a player who had paid would keep a `pending` row and no coins until somebody read the audit log. Raised as Question 1 of the Sprint 1 Step 4 plan rather than decided in passing.
- **Decision:** A rolled-back settlement answers **500 `wallet_write_failed`**, so Stripe retries it — up to three days, at least once. Everything else the earlier entry fixed stands unchanged: an unknown session, an already-settled one, an amount or currency mismatch, and any unhandled event type all answer 200 with a `note`, and a missing or invalid signature answers 400 / 401.
- **Alternatives rejected:** 200 with `note: settle_failed` and an audit entry — keeps the earlier wording exactly, at the price of converting a transient database fault into a silently unpaid top-up, which is the one outcome the money zone exists to prevent. Retrying inside the request — a second attempt against the same unhealthy database, holding the connection open, with no more chance of success.
- **Consequences:** The webhook now has three non-2xx answers, not one; `docs/CONTRACTS.md`, root `CLAUDE.md` invariant 5 and `docs/features/stripe/FEATURE.md` invariant 3 all say so. The distinction that matters is **permanent versus transient**: retrying an unknown or already-settled session can never help, so it answers 200; a failed write is exactly what a retry fixes. `tests/stripe-webhook.php` forces the failure, asserts the 500 and the untouched row, and then asserts the retry settles normally.

---

## 2026-09-18 — Machine power is a manual operation; `realtime` assumes the machine on
- **Context:** `PUSHER-COIN-COMMANDS.txt` documents `switch/turn_on` and `switch/turn_off`, and the 2026-09-16 ad-hoc repointed `Machine_Service::power_on()` / `power_off()` at the renamed entity `switch.s60tpf`. The user states that today these calls do not work: the machine at the venue is switched on and off by hand, and it will be left on throughout the `realtime` sprints.
- **Decision:** For the whole of `realtime`, machine power is a physical, manual operation outside the software. Every sprint, step, verification and transport assumes the machine is already on; nothing may depend on `turn_on` / `turn_off` answering, and no step is allowed to require switching the machine from the admin **Machine** screen.
- **Alternatives rejected:** Making the power commands work first, as a prerequisite — the switch is the venue's hardware and the venue's Home Assistant, outside this feature's control, and no `realtime` step needs remote power to do its job. Folding "why does power not work" into the transport spike — the spike exists to choose how events leave Home Assistant, not to debug a wall switch.
- **Consequences:** Sprint 3's "machine offline" detection can no longer treat power-off as a fault by itself: the machine goes off by hand every day, so *offline during a scheduled broadcast window* is the incident and `wp_pc_room_schedules` is what tells the two apart. The admin **Machine** screen's On/Off control and ROADMAP Phase 5 §2 are now unverified against a machine nobody switches from software — whether to retire or keep them is a product question for `core`, recorded here as open, not a `realtime` task. `PUSHER-COIN-COMMANDS.txt` keeps documenting both calls until the machine's owner says otherwise. The `realtime` sprint files are rewritten to drop every "power it on/off from the admin screen" precondition.

---

## 2026-09-18 — What the abandoned spike established is kept as fact; Step 2 does not repeat it
- **Context:** The first run of `realtime` Sprint 1 lives on the unmerged branches `realtime/sprint-1` and `realtime/sprint-1-spike-transport` (Step 1 closed, Step 2 half done on 2026-09-16). Those branches are being dropped (next entry), but the spike's desk session produced facts about the venue's Home Assistant that nothing else records.
- **Decision:** Treat the following, observed 2026-09-16 against the live instance, as established; the rewritten Sprint 1 Step 2 starts from them and does not re-run them. (1) The long-lived token authenticates on both the REST API and `wss://developer-it.com/api/websocket` (`auth_ok`); Home Assistant 2026.9.2. (2) The token is **owner-level admin** (`auth/current_user` → `is_admin=true is_owner=true`), so an HA automation is available as a transport. (3) `subscribe_events` with `state_changed` is accepted and delivers; socket delivery lags HA's own `last_updated` by ~40 ms. (4) Every state carries `last_changed`, `last_updated`, `last_reported` and a `context.id` — the last two are `event_key` candidates. (5) Idle values: `sensor.coin=0`, `sensor.lc01_12=0`, `sensor.light_b_t=0`, `sensor.relay_on=1`; after power-on the sensors come alive ~30 s later in one batch. (6) Ten days of `/api/history/period` (which silently returns one day unless `end_time` is passed) show `sensor.coin` cycling `0…8` and **returning to 0 — a per-payout counter, not a cumulative total** — and `sensor.lc01_12` at `0` throughout: **no bonus has ever been recorded**. (7) A `toss_a_coin` press accepted with HTTP 200 moved no sensor within 25 s. (8) An undocumented `sensor.sw_b_t_relay` exists next to `sensor.relay_on`; which one is the contact sensor is unknown. (9) `sensor.relay_on` idling at `1` is read by `normalise_truthy()` as "closed", on which `POST /rooms/{id}/play` would refuse every toss — unconfirmed against a real payout. (10) 2.5 idle minutes produced zero `state_changed` events; whether the integration *re-reports* unchanged values (visible only through `last_reported`, never through `state_changed`) was not tested.
- **Alternatives rejected:** Re-running the desk session — it would reproduce the same facts at the cost of a day. Keeping the branch alive to hold them — a branch is not a record; the next entry explains why it goes.
- **Consequences:** Points 6, 8 and 9 are contradictions between the code and the machine, not open questions: `get_coin_count()` and `ingest_coins_dropped` assume a cumulative counter that does not exist; the relay polarity and the relay entity are unsettled. Sprint 1 Step 2 resolves them by observation and Step 4 builds on the resolved model, not on the current code's assumptions. Point 10 is the first thing Step 2 checks, from the desk.

---

## 2026-09-18 — `realtime/sprint-1` is dropped; Sprint 1 is re-planned from scratch
- **Context:** While the first Sprint 1 sat unmerged on its branch, the `stripe` feature ran two sprints into `main`; its Sprint 1 Step 1 created `backend/bin/check` and `frontend/bin/check` and documented them in `docs/TECH-STACK.md`. The branch's closed Step 1 had created the same command differently, so merging it now would collide on the one file every commit must pass through. Separately, the 2026-09-18 decision that machine power is manual invalidates preconditions in every `realtime` sprint file, and the files themselves predate playbook v1.21, whose sprint template has no Definition of Done, Out of scope or Risks section.
- **Decision:** Abandon `realtime/sprint-1` and `realtime/sprint-1-spike-transport` (the user deletes them); treat Sprint 1 as never started, and rewrite all three `realtime` sprint files from the v1.21 template. Step 1 becomes a delta-audit only — the check command exists on `main` — and Step 2 starts from the facts kept in the previous entry.
- **Alternatives rejected:** Keeping the branch as the truth and appending to Sprint 1 — leaves two competing check commands to reconcile at merge, and a closed Step 1 whose main deliverable another feature has since shipped. Rebasing the branch onto `main` — it is docs plus a throwaway spike; there is less to rebase than to rewrite.
- **Consequences:** `SPRINT-1.md` shows no closed step and may be rewritten freely until `/close-step` ticks one. The delta-audit now reads `stripe`'s `FEATURE.md` as a sibling and follows its `app/stripe/bootstrap.php` registration pattern for `app/realtime/`. What the old files held under Definition of Done moves into each sprint's Goal and each step's Verification (manual); what they held under Out of scope moves to `FEATURE.md` → Roadmap.

---

## 2026-09-18 — Spike: machine events reach WordPress by polling Home Assistant's history; the coin counter is the only payout signal
- **Context:** `realtime` Sprint 1 Step 2, the timeboxed spike (the transport was left open on 2026-09-15). **Evidence base.** A live recording ran 11:24:30–13:19:46Z on 2026-09-18: a WebSocket `state_changed` subscription and a 10-s REST sample of `sensor.coin`, `sensor.lc01_12`, `sensor.relay_on`, `sensor.sw_b_t_relay`, `input_button.toss_a_coin` and `switch.s60tpf`, 4,514 records. Home Assistant's stored history covered 2026-09-08 → 2026-09-18 (`/api/history/period` with an explicit `end_time`, `significant_changes_only=0`), including the relay buttons. There was one announced toss press, at 13:16:41Z. The step asked for a full venue day of recording. The user cut it short ("I can't wait that long") and chose stored history plus up to five presses. One press was enough. HA 2026.9.2, `config_source: storage`. The machine's sensors arrive through the `modbus` integration.
- **Decision:** **WordPress polls Home Assistant's history.** On a schedule, WordPress asks `GET /api/history/period/{cursor}?end_time={now}&filter_entity_id=sensor.coin&significant_changes_only=0` through `Machine_Service`, the only code that talks to Home Assistant. It credits what changed since the cursor. The user chose this among the three options; it was the recommended one.
  - **`event_key`** = `ha:{entity_id}:{last_updated}`. `last_updated` is taken verbatim from the history row, e.g. `ha:sensor.coin:2026-09-10T11:36:14.233147+00:00`. Every history row carries it, a given change always carries the same value, and no two changes of one entity share it. `context.id` is **not** usable: all sensors written in one batch share one context id (the four sensors at power-on had one id).
  - **Coins credited per row:** take the consecutive numeric `sensor.coin` rows, skipping `unavailable` / `unknown`. When the value rose, credit `new − previous`. When it fell, the counter was reset by a toss, so credit `new` (normally 0, which is no event).
  - **Latency: 65 s** from coins landing to the credit, with a 60-s schedule. That is ≤ 2 s for the machine's Modbus cycle (every `sensor.coin` change in history sits on a ~2-s grid), plus ≤ 60 s waiting for the next poll, plus ≤ 1.6 s for the history call (five calls took 0.13–1.55 s). The schedule interval is a `pc_realtime_*` option, so a shorter interval lowers the number.
- **Observed sensor model** (log lines in the spike's notes; history rows cited by time):
  - **`sensor.coin` counts coins paid out since the last toss, and a toss resets it to 0.** 13 of the 14 resets in history came 0.09–1.98 s after a `toss_a_coin` press, e.g. 11:36:35.64 press → 11:36:36.24 reset from 5. The fourteenth (2026-09-10 11:47:40) had no press in the 74 s before it; its cause is not observable here. The value rises in steps on the ~2-s cycle, sometimes several at once (0 → 4, 3 → 6). It is 0 at every power-on (17 power-ons). It is **not** cumulative, so `get_coin_count()`'s "Cumulative" and `ingest_coins_dropped`'s assumed "delta of a cumulative reading" are wrong. The first coin of a payout came 1–12 s after a press, and a payout kept counting for up to ~20 s.
  - **Neither relay sensor signals a payout.** `sensor.relay_on` follows the relay the system switches with the buttons, about 1.1 s after the press. On 2026-09-10: `input_button.relay_off` 11:51:21.49 → `relay_on` 0 at 11:51:22.73; `input_button.relay_on` 11:51:41.77 → 1 at 11:51:42.74; `relay_off` 11:51:57.65 → 0 at 11:51:58.76. It also went back to 1 at 11:57:06.85, in the same poll as a toss-triggered reset. Its normal state is **1**, which by the documented button names ("замкнуть" = close, `relay_on` → 1) means **closed**. It stayed 1 through all 28 rises of the coin counter in history. `sensor.sw_b_t_relay` was 0 in every row for ten days, including while the relay was switched.
  - **A repeated value leaves no trace.** In all 2,768 live sensor samples, `last_reported` equalled `last_updated`: the integration writes only changed values. Nothing re-reported during 104 s of idle at the desk, nor after two real toss presses (13:16:41Z and 13:17:04Z), which would have re-written the counter's 0. `state_reported` cannot be subscribed to over the WebSocket without a filter; HA answered "Event filter is required for event state_reported". A repeat is invisible to every transport.
  - **No bonus was ever observed.** `sensor.lc01_12` was 0 in every row of ten days and today. This is a **recorded unknown**: nobody knows what a bonus looks like, whether its coins also pass through `sensor.coin`, or whether the number returns to 0 between bonuses.
- **Alternatives rejected:**
  - **HA automation → webhook.** This Home Assistant has no outbound-HTTP service: no `rest_command` or `shell_command` domain; `notify` only reaches the mobile app. An automation cannot call WordPress until someone with configuration-file access adds one. API admin rights do not reach that, so it depends on the machine owner.
  - **An always-on WebSocket worker.** It is the fastest: delivery measured −46…263 ms from HA's `time_fired`, with a ping round trip of about 55 ms and the laptop within 6 ms of `time.apple.com`. But it adds a fourth deploy target, and it would be a second Home Assistant client outside WordPress, against root `CLAUDE.md` invariant 8.
  - **Polling current states** instead of history. A 60-s poll misses whole payouts, because the counter can rise and reset between two reads. History holds every change.
- **Consequences:**
  - **Crediting.** The ingest **may credit only from `sensor.coin` increments**. It may not credit from `sensor.lc01_12`: never observed, and a repeated number is invisible. It may not credit from the relay either (`ingest_relay_closed` / `pc_machine_relay_coin_count`): no payout event lies behind it. If both a bonus's mapped coins and its physically dropped coins were credited, one bonus could pay twice. Step 4 (its bonus test and its coins-dropped wording) and Step 5 (its "cron poller" line names a `last_reported`-derived key, which is `last_updated` here, and the two are equal for these sensors) plan against this entry.
  - **Retention.** HA keeps ten days of history (the oldest row seen), so a poller outage longer than that loses events. A lost cursor loses nothing, because the idempotency key catches a re-read.
  - **Scheduling.** Whether the production host has a real cron or only traffic-driven WP-Cron is not known yet; that is Step 5's to establish.
  - **The toss lock is wrong today.** `Machine_Service::get_relay_closed()` reads the normal `1` as "closed", so `POST /rooms/{id}/play` answers 423 `relay_closed` to **every toss while the machine is on**. That is a defect in frozen `core`, for `/adhoc`. Sprint 2 Step 3's "relay lock" has no payout signal to follow.
  - **The last-coin handover is real.** A payout keeps counting up to ~20 s after a toss, so after a player's last declared coin the later coins go to the next player (`FEATURE.md` → Conflicts).
  - **Someone else holds the token.** A second `toss_a_coin` press at 13:17:04Z came from the same HA user as the spike's token, but not from the spike, so the token is also in use elsewhere.
  - **Machine power:** off → sensors `unavailable` about 3 min 22 s later; on → sensors live 8.6–45 s later, all in one batch.

---

## 2026-09-21 — The machine ingest: a shared secret, one rate-limit bucket, and 200 for everything a retry cannot fix
- **Context:** `realtime` Sprint 1 Step 4 built `POST /pc/v1/machine/events`. Its caller is a machine, not a signed-in player, so there is no JWT to gate on; and whatever transport calls it (Step 5 builds one; others may follow) will redeliver whenever it gets an error. Two shapes had to be fixed before anything is built on top of it.
- **Decision:**
  - **The credential is a shared secret** in `X-PC-Machine-Secret`, compared to `PC_MACHINE_INGEST_SECRET` with `hash_equals()`. A wrong secret, a missing header and a server with no secret configured all answer one 401 that says nothing about which; the audit log distinguishes them.
  - **One rate-limit bucket for the whole endpoint, checked before the secret.** Not per IP: `Rate_Limiter::client_ip()` reads a spoofable `X-Forwarded-For` (review item 5), so an IP-keyed ceiling is none. Checking it before the credential caps what an unauthenticated flood can write into `wp_pc_auth_audit_log`. The ceiling is `pc_realtime_ingest_rate_max` per `pc_realtime_ingest_rate_window_seconds` (120 / 60).
  - **Everything a retry cannot fix answers 200** with a `status`: `credited`, `recorded` (a bonus the map pays 0 for), `unattributed`, `already_recorded`, and `failed` (the row is written, the wallet move was not). **The one non-2xx besides validation and the 401 is `machine_event_write_failed` 500** — the event row itself could not be written, so nothing was decided and a retry is exactly right.
  - **A duplicate and a failed write are different answers.** `Machine_Event_Log::record_result()` reports `inserted` / `duplicate` / `failed`; the crediting path turns `failed` into the 500. The audit-only path (`log_event()`) reports it as a flag instead, because its caller in the toss endpoint has already debited a coin and tossed it for real — an error object there would cost a player a coin.
- **Alternatives rejected:** An HMAC over the payload, like Stripe's — the transport is WordPress itself polling Home Assistant, and Home Assistant cannot sign anything on the way out (it has no outbound-HTTP service at all, 2026-09-18), so a signature would only be WordPress signing its own request. A per-IP rate limit — bypassable by the very header the limiter trusts. 500 on a wallet write that failed — the row already holds that `event_key`, so every retry would come back `already_recorded`; the row is the forensic record and an operator settles it with `wp pc machine-ingest --player=…`. 200 for an unwritten row — it retires a payout unpaid, unlogged and unretryable, which is the defect this step fixed.
- **Consequences:** Rotating the ingest secret is a wp-config edit on the server and nothing else; while it is unset every call answers 401, so a real event delivered before an operator sets it is lost (Step 5's verification depends on it existing in production). A transport must treat 200 as final and 500 as "send it again", and must supply an `event_key` — a body without one is refused outright. A flood can spend the window on the legitimate transport; that loses nothing, because the chosen transport re-reads history from its own cursor. `status: failed` rows are a queue for a human, and nothing yet surfaces them — Sprint 3's alerting is where that belongs.

---
