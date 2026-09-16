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
