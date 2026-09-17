# Worklog — Pusher Coin

<!-- Add-only project memory for the agent: entries are never edited or
     removed. Written ONLY by /close-step, /close-sprint (one entry per closed
     sprint) and /adhoc (off-cycle tasks),
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
