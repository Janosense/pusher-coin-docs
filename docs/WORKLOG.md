# Worklog — Pusher Coin

<!-- Add-only project memory for the agent: entries are never edited or
     removed. Written ONLY by /close-step and /adhoc (off-cycle tasks),
     newest entry at the TOP, directly under the entry format. A fresh Claude
     Code session reads the latest 5 entries at start (CLAUDE.md core rule 6).
     Keep entries 3–6 lines; this is a memory index, not a diary — details live
     in commits and verification guides. -->

Entry format:

## {{YYYY-MM-DD}} — [{{feature}}] Sprint {{N}} Step {{M}} — {{title}}
(ad-hoc tasks: `## {{YYYY-MM-DD}} — [adhoc] [{{feature}}] — {{title}}`;
sprint closed: `## {{YYYY-MM-DD}} — [{{feature}}] Sprint {{N}} closed`)
- Changed: {{what, at module/feature level}}
- Decisions: {{key ones made or DECISIONS.md entries added, or "—"}}
- Open: {{unresolved questions carried forward, or "—"}}

---

## 2026-09-18 — [stripe] Sprint 1 closed
- Merged: `stripe/sprint-1` → `main` `--no-ff` in all four repositories — docs `6da0126`, backend `75450469`, frontend `7210c59`, admin `c40773a`; zero conflicts, every gate exit 0 on `main` (146 checks). Sprint branches kept. **Stripe replaced LiqPay end to end**: hosted Checkout hand-off, the settlement webhook as the only path to `completed`, and LiqPay gone from all three repositories. `SPRINT-1-CLOSE.md` is the handoff
- Deployed: **claimed, not corroborated.** The three boundary items are ticked `confirmed by user 2026-09-18`. At close time `main` was 17 commits ahead of `origin/main` in `backend` (its `origin/main` still `b23e5f2a`, 2026-07-28), 8 in `frontend`, and `https://pusher-coin.envstage.link` answered HTTP 500 "Error establishing a database connection" on every URL. `SPRINT-1-CLOSE.md` records this in full — **re-check the deploy before trusting anything that assumes a live release**
- Carried: nothing carried into Sprint 2 as a Definition of Done item. Open elsewhere: three `pending` LEARNINGS entries awaiting a retro (the repo-root `grep` blind spot, the test-card prohibition collision, FEATURE.md's 80-line guideline); five contradictions listed unresolved in `SPRINT-1-CLOSE.md`; `uah` still provisional on a US sandbox, to be re-verified on the client's account; the sandbox key expires 2026-10-13

---

## 2026-09-17 — [stripe] Sprint 1 Step 5 — The SPA hand-off, and LiqPay leaves
- Changed: **the player's top-up button works again** — `ReplenishmentBalance.vue` navigates to the Checkout Session URL instead of building LiqPay's form, closing the gap Steps 3-4 opened by design. **LiqPay is removed from all three repositories**: `PaymentController.php` and `liqpay-client.php` deleted with their loaders (`POST /payments/liqpay/callback` answers 404), `liqpayCheckout.js` deleted, the admin Settings section rewritten to name the two wp-config constants and the webhook URL to register. **Touches shared code in all three apps** — `WalletController`, `app/rest-api.php`, `app/utils.php`, `captcha-verifier.php`, `stores/wallet.js` (docblock only, `realtime` Sprint 2 also reads it), `walletService.js`, `AccountView.vue`, `SettingsView.vue`. Both config templates gained **commented** Stripe placeholders — the delta-audit was right that this adds rather than replaces. Frontend `294ad91`, backend `356161ae`, admin `5e638d8`, docs `3692364` `fc8e1c5` `3915281` `d120db4`
- Decisions: none new — the step executed `DECISIONS.md` 2026-09-16 "LiqPay is removed, not parked". Two inherited doc corrections landed here: `DATA-MODEL.md` invariant 6 still named the LiqPay callback (Step 3 assigned the rewording to Step 4; Step 4's doc list did not carry it) and its audit-event table still listed the seven `liqpay_*` types instead of the ten `stripe_webhook_*` ones
- Open: **LEARNINGS 2026-09-17 — a repo-root `grep` silently skips `backend/`, `frontend/` and `admin/`** (the root `.gitignore` lists them; the session's grep honours ignore files), so this sprint's own "grep across the three repositories" evidence reads as a clean pass while searching nothing. Verified per repository with `git grep` instead: frontend and admin clean, backend keeps exactly 7 deliberate lines (the option-deleting migration, the "LiqPay-era rows still render" note, one decision citation). `docs/BACKEND-REVIEW.md` items 2 and 11 describe defects in code now deleted — left for `/close-sprint` or `/adhoc`. The **browser** hand-off is the one thing not agent-verified (signing in as a player needs a password); the bundle ships `location.assign(checkoutUrl)` and no LiqPay

---

## 2026-09-17 — [stripe] Sprint 1 Step 4 — The webhook and settlement
- Changed: `POST /pc/v1/payments/stripe/webhook` — **a paid session now credits the wallet end to end.** New `app/stripe/StripeWebhookController.php`, registered from the feature's own `bootstrap.php`, so **no shared code was modified at all** this step (`Wallet_Service` and `Audit_Log` are called, not changed). The signature over the raw body is the credential. Idempotency rests on the row's status, never on delivery order; the coins credited come from the ledger row, never from the event. `tests/stripe-webhook.php` adds **54 checks** (146 across the three scripts). Backend `0e051977`, docs `3a27739` `9cba287`
- Decisions: DECISIONS 2026-09-17 "A settlement that cannot be written answers 500, so Stripe retries" — raised as Question 1 of the step plan and approved. It **extends** the earlier "a non-2xx leaves the handler only for a bad signature": permanent conditions (unknown session, already settled, amount mismatch) answer 200 with a `note`, but a rolled-back write is transient and exactly what a retry fixes, so it answers 500 rather than becoming a silently unpaid top-up. `FEATURE.md` invariant 3 and root `CLAUDE.md` invariant 5 corrected to match
- Open: **verified live, not only by script** — a real test payment took the player 0 → 2 coins with the row `completed`; resending the same event left the balance at 2 and logged `already_settled`; a dispute changed nothing. The player SPA's top-up button is still broken until Step 5. **The production webhook URL is not registered in the Stripe Dashboard** and its `whsec_` differs from the CLI's — a Definition-of-Done item before the first release. `docs/TECH-STACK.md`'s ANTI-PATTERNS line and the LiqPay Integrations row are Step 5's to remove

---

## 2026-09-17 — [stripe] Sprint 1 Step 3 — The Stripe client and the Checkout Session
- Changed: `POST /wallet/topup` now opens a real Stripe Checkout Session. New `app/stripe/` (`bootstrap.php` + `stripe-client.php`, one `require_once` in `functions.php`), carrying both spike findings: `adaptive_pricing[enabled]=false` injected by the client so no caller can forget it, and `Stripe-Version` pinned to `2026-06-24.dahlia`. **Touches shared code:** `Wallet_Service` (new `set_external_ref()`), `WalletController::topup` (rewritten in place), `Install_Schema` (`DB_VERSION` 1.8.0 → **1.9.0**, `pc_liqpay_public_key` deleted through a new `remove_retired_options()` — the installer had no upgrade path at all). `set_external_ref()` removes the **last raw `$wpdb->update` of a money column** in the codebase. Backend `433b19a8` `d6aa851f` `4603f09e`, docs `5ab61a1` `6afde91` `e683895`
- Decisions: none new — Adaptive Pricing and the API version were settled by the Step 2 spike entry. Two narrowing deviations recorded in the plan: `stripe_call_failed` → 502 set directly rather than through a one-row lookup table, and a failed `set_external_ref` answers `wallet_write_failed` 500 with the row parked `failed` (the plan said the write is checked but not what the check does on failure). `Audit_Log` on those paths was **left out** as Step 4's scope
- Open: **the player SPA's top-up button is broken until Step 5** by design — the server returns `checkout_url` while `services/walletService.js:22-31` and `ReplenishmentBalance.vue:76` still expect the LiqPay envelope; nothing is deployed before the sprint boundary, and this step was verified by calling the endpoint directly. Nothing can be paid successfully until Step 4's webhook. `docs/DATA-MODEL.md` invariant 6 still says "the LiqPay callback" because it still is — Step 4 owns that rewording. The sandbox `sk_test_` key expires **2026-10-13**

---

## 2026-09-17 — [stripe] Sprint 1 Step 2 — Spike: Stripe on the test keys
- Changed: documentation only — no application code, no shared code, nothing deployed. **The decisive question is answered: Stripe accepts `uah`** (HTTP 200, `currency: uah`, `amount_total: 12000`, the hosted page rendered "UAH 120.00"), so the sprint does not stop and Step 3 may proceed. Evidence came from `acct_1TtSrOElMyJqvLDl` — a **US sandbox** the Stripe CLI was already logged into, approved as provisional; the entry says so and requires re-verification on the client's real account before go-live. The throwaway recorder and its log are deleted (`POST /pc/v1/payments/stripe/webhook` answers 404). Docs `0c7a533`
- Decisions: DECISIONS 2026-09-17 "Spike: Stripe accepts `uah`, on a provisional US sandbox; pin `2026-06-24.dahlia` and turn Adaptive Pricing off". Four findings the sprint did not anticipate, each binding on later steps: **Adaptive Pricing is enabled by Stripe unasked** and must be sent `false` in Step 3, or a legitimate payment from outside Ukraine reads as `amount_mismatch` in Step 4; the signature header carries **three** parts (`t`, `v1`, `v0`) so the verifier parses keys, not positions; the six events of one payment **do not arrive in creation order**; an idempotent replay returns the **cached original body**, not live state. Signature scheme proved against a real delivery with tampered-body and wrong-secret controls; delivery measured at 0.88 s / 0.48 s
- Open: the sandbox `sk_test_` **expires 2026-10-13** — Step 3's manual verification needs a working key before then. The `uah` answer is presentment, not settlement (a US account settles USD); the product invariant covers only what the player is charged and what the wallet records, both `uah`. Stripe CLI 1.43.8 now recorded in TECH-STACK per the 2026-09-17 decision. LEARNINGS 2026-09-17: "pay with the test card" collides with the standing prohibition on entering card numbers — resolved for a published test number in a `livemode: false` sandbox, flagged so Step 4 need not re-argue it

---

## 2026-09-17 — [stripe] Sprint 1 Step 1 — Delta-audit and the check command
- Changed: `stripe/sprint-1` now carries the check command, copied byte-for-byte from `realtime/sprint-1` (blob-hash verified, not eyeballed) per DECISIONS 2026-09-17 — `backend/bin/check`, `frontend/bin/check`, and the `lint` / `lint:fix` split. **Touches shared code:** `npm run lint` is report-only in `frontend/` and `admin/` (both branches now carry the identical change; frontend CI reports fixable errors instead of repairing them). Backend `40fbfc8c`, frontend `9167691`, admin `c935d4d`, docs `01c82fe` `636af4b`
- Decisions: none new — the copy was already settled by DECISIONS 2026-09-17. That entry's three doc targets were treated as exhaustive, so the realtime-only DECISIONS entry "The check command calls the WP-CLI eval scripts…" was deliberately **not** copied; the copied TECH-STACK text's `DECISIONS.md` 2026-09-15 citation resolves on this branch to "Interim money checks are WP-CLI eval scripts"
- Open: the delta-audit corrected five things now recorded in `docs/features/stripe/FEATURE.md` with file:line — most importantly `frontend/src/services/walletService.js:22-31` was **missing** from the touchpoint list (Steps 3 and 5 must change it), and Step 5's "replace the LiqPay constant" in `wp-config-*` is an **addition**, not a replacement, because neither tracked config defines any `PC_*` constant. `SPRINT-1.md` Step 5's task text still reads "replace" — left as written; FEATURE.md is the corrected record. FEATURE.md is 96 lines over its ≤80-line guideline (LEARNINGS 2026-09-17). Nothing pushed; `stripe/sprint-1` exists in all four repositories

---

## 2026-09-16 — [adhoc] [core] — Machine power switch points at a renamed entity
- Changed: `Machine_Service::power_on()` / `power_off()` / `get_power_on()` now default to `switch.s60tpf`; `switch.sonoff_10024fb618` was renamed in the venue's Home Assistant and 404s, so the admin **Machine** screen could neither read nor change the machine's power. No migration — `Install_Schema` seeds no `pc_machine_*` option, so the code default is what every environment uses. Found by the `realtime` Sprint 1 Step 2 spike. Backend `6d2cdf5e`, docs `c65430e`
- Decisions: — (the relay-sensor and coin-counter findings from the same spike are NOT fixed here: unproven until a real payout is observed)
- Open: verified against HA by curl, but the admin screen itself is unverified until `backend` `main` is pushed (that push is the production deploy). `admin/src/views/MachineView.vue:122` still tells the operator it "Toggles `switch.sonoff_*` directly" — left alone, outside this ad-hoc's approved scope

---

## 2026-09-15 — [adhoc] [core] — Wallet writes roll back on database failure
- Changed: `Wallet_Service` checks every statement inside its transactions (`ensure_written()`, START/COMMIT included), so a failed write rolls the whole money operation back; review item 1 of `docs/BACKEND-REVIEW.md`. `POST /wallet/withdraw` answers `wallet_write_failed` with 500. New check `ddev wp eval-file wp-content/themes/pc/tests/wallet-rollback.php` (53 checks; 33 fail on the old code). Backend `35088818`, docs `975beb5`
- Decisions: DECISIONS 2026-09-15 "Interim money checks are WP-CLI eval scripts, not PHPUnit"
- Open: backend `main` not pushed (a push is a production deploy). Review items 2–15 open; items 2, 3 and 9 race conditions are next in the money zone. `ddev start` strips the JWT/Google constants from `wp-config-ddev.php` (LEARNINGS)

---

## 2026-09-15 — [adhoc] — Playbook v1.17
- Changed: `.claude/commands/` and `templates/` synced from the playbook (v1.17); root `CLAUDE.md` Core rules (9 → 6) and Step protocol replaced verbatim, `Origin:` line, Features definition and Rules block dropped, Git model environments sentence from the template; "core rule N" references renumbered in TECH-STACK, DATA-MODEL, WORKLOG (TECH-STACK's retired rule 2 now points at `/do-step` §3)
- Decisions: —
- Open: —
