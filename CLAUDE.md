# Pusher Coin — real-time coin-pusher gambling platform

<!-- playbook: v1.25 — Core rules and Step protocol are verbatim copies of
     templates/CLAUDE.md; never edit them here. -->

Players watch a live stream of a physical coin-pusher machine, buy coins in UAH, queue for
their turn, and toss a real coin into the real machine from the browser; whatever the machine
pays out is credited back to their wallet. Three decoupled apps: a player SPA (`frontend/`), an
operator SPA (`admin/`), and a WordPress install (`backend/`) whose custom `pc` theme *is* the
whole API — and the only party that ever holds machine or payment credentials.

## Project profile
- Verification: user-verified — the user does not read code; explain all changes in plain language and write verification guides for a non-developer
- Deploy: **manual per repository, and a merge is a deploy.** `backend/` — GitHub Actions FTP-syncs the whole tree on every push to `main` (pushing `main` *is* a production release); `frontend/` — Vercel builds from `main` with `.env.production`; `admin/` — no deploy target and no CI at all, local-only today.
- Test-critical zones (code without tests here = unfinished task): money — `Wallet_Service`, coin lots, transactions, withdrawal approve/reject; the Stripe webhook — signature verification and settle-once idempotency; refresh-token rotation and reuse detection; every `Permissions::*` callback; machine-event idempotency and crediting. **There is no automated test suite in any of the three repositories today** — see `docs/TECH-STACK.md` → Check command.
- Git model: chained sprint branches: `core/sprint-N` ← `main`, task branches `core/sprint-N-short-name` merged `--no-ff`. Branch names carry the feature name. The three apps are separate git repositories, so a step that touches more than one carries the same branch name in each and they are merged together at the sprint boundary. Environments track `main` (or the deployment branch named under Deploy) — never a task or sprint branch; a deploy is the user's own action from `main`, never a step task.

## Documentation (read before the relevant task)
| File | When to read |
|---|---|
| `docs/ARCHITECTURE.md` | Before structural work: new modules, endpoints, integrations, deploy questions. Same commit: update it when the auth flow, deploy targets, trust boundaries or state stores change |
| `docs/TECH-STACK.md` | Before adding dependencies or choosing an approach. Contains the ANTI-PATTERNS and CONVENTIONS sections — mandatory |
| `docs/DATA-MODEL.md` | Before any schema change, migration, query, or API response shape. A new table / CPT / user meta key / WP option is unfinished until it is here *and* in `Install_Schema` / `User_Meta_Keys` / `Post_Meta_Keys` |
| `docs/DOMAIN.md` | Before implementing or changing any domain logic: the customer's terms, rules, invariants |
| `docs/CONTRACTS.md` | Before touching any endpoint, route, request/response shape or error code. Same commit: a shipped endpoint moves from "planned" to "current", a new error code goes into the registry |
| `docs/TESTING.md` (if present) | Before writing or changing tests |
| `docs/DESIGN.md` (if present) | Before any UI work: tokens, components, screen names. Design files in `docs/features/{feature}/design/` are references, never code to copy |
| `docs/DECISIONS.md` | Before proposing an architecture/tooling change — it may already be decided |
| `docs/features/{feature}/FEATURE.md` | Before any work in a feature: its scope, data ownership, invariants, interfaces |
| `docs/features/{feature}/sprints/SPRINT-N.md` | Current sprint scope and steps |
| `docs/WORKLOG.md` | At session start: latest 5 entries (top of file) = project memory |
| `docs/ROADMAP.md` | The pre-playbook status board: phases 0–8 tagged `[done]` / `[partial]` / `[todo]`, the tracking matrix, and the still-open questions. Read it before planning anything new; update the tag and the matrix in the same change that completes the work |
| `docs/INVENTORY.md` | The frozen Phase 0 audit plus per-phase highlights: REST surface, `player` role, user meta keys, frontend stubs, resolved frontend/backend drift |
| `docs/PROJECT-TREE.md` | Before moving files or adding top-level ones — the directory map of all three apps; update it in the same change |
| `docs/PUSHER-COIN-COMMANDS.txt` | Before any Home Assistant work — the physical machine's own API, entity ids and service calls |

## Core rules
1. New dependencies only after explicit approval. Propose, explain why, wait for a "yes". This includes transitive tooling (linters, build plugins).
2. Do not "improve" without being asked. No speculative abstractions, caches, or extra layers. See an opportunity — propose it, don't do it silently.
3. Never hardcode business values. Prices, limits, intervals, texts that the business may change are configuration, not constants.
4. Secrets only via environment config. Never commit keys, never log secret values.
5. Documentation is part of the task. Docs that describe changed code are updated in the same commit as the change; a schema change without a `docs/DATA-MODEL.md` update is an unfinished task.
6. Project state is derived, never asked for: the newest `docs/WORKLOG.md` entry names the feature, sprint and step; that feature's `sprints/SPRINT-N.md` shows which steps are ticked, `SPRINT-N-PLAN.md` the step in flight (awaiting approval / in progress / awaiting verification / closed). Derive it before any work on the project — not before answering a question. No WORKLOG entries = the project has not started.

## Step protocol
- Code is changed only inside a step (`/plan-step` → `/do-step`, a
  `/fix-step`) or an `/adhoc`. The unit of work is **one Step** of the feature's `SPRINT-N.md` —
  never a whole sprint. If the user asks to "do the sprint" or "start the sprint":
  do not execute it; propose `/plan-step` for the first incomplete step.
- A step plan is produced only by `/plan-step` and executed only by
  `/do-step`. Running `/do-step` is the approval of the written plan — the
  user is never asked to say "approved"; any other message after the plan is
  a change request. Approval authorizes that step only — never the
  following steps.
- An implemented step is verified by the user on its task branch, by the
  verification guide `/do-step` wrote, and then closed with `/close-step` —
  running it means verified. No other step begins before the close.
- A failed item of a verification guide — reported in the chat or with
  `/fix-step <what failed>` — is handled only by the fix procedure: Read
  `.claude/commands/fix-step.md` and follow it, the report being its
  argument. Before the close the fix lands on the step's task branch; after
  it, on a fix branch. A failure report is never an instruction to patch
  the step directly.
- The next step begins only with a new `/plan-step` from the user.
- A sprint is complete when `/close-step` ticks its last step: that run
  merges the sprint into `main` and says so; `/plan-step N+1 1` does not
  start before the sprint is on `main`. Nothing in `SPRINT-N.md` is
  edited by hand.

Outside the step cycle:
- Questions (explain code, "why is X built this way", "what would it take
  to…", reading/analysis): answer anytime, with no ceremony — but a
  question is never an instruction to change code. If the answer implies a
  change, say so and wait.
- Ad-hoc tasks (a change outside the current sprint's scope: hotfix, small
  tweak, config change): go through `/adhoc`. Never fold ad-hoc changes into
  an open step's commits or plan.

## Domain invariants
1. Every REST route declares an explicit `permission_callback`. Whatever the SPA enforces — coin-quantity clamps, zero-balance routing, the nickname / terms / email gates — is re-checked server-side. The browser is untrusted; only the JWT identifies the caller.
2. Secrets are wp-config constants on the server: `JWT_AUTH_SECRET_KEY`, `PC_STRIPE_SECRET_KEY`, `PC_STRIPE_WEBHOOK_SECRET`, `PC_MACHINE_TOKEN`, `PC_MACHINE_INGEST_SECRET`, `PC_CAPTCHA_SECRET`, `GOOGLE_CLIENT_ID`, `APPLE_*`. Never in the database, never in the repository, never logged. The top-up provider has no public counterpart at all — hosted Checkout needs none; the captcha site key is a WP option.
3. Every wallet mutation goes through `Wallet_Service` under `SELECT … FOR UPDATE`. Coins are a FIFO stack of `(qty, unit_price)` lots, so a coin always pays back at the price it was bought at.
4. A coin is debited only after `Machine_Service::toss_coin()` answers HTTP 200. Any other answer re-credits the exact lot price that was taken.
5. `POST /payments/stripe/webhook` is the only place a transaction flips `pending → completed`. Idempotency rests on the row's own status, not on delivery order — Stripe delivers at least once and out of order. The route answers 200 with a `note` for everything except an unverifiable signature (400 / 401) and a settlement that rolled back (500, so Stripe retries).
6. Machine events are idempotent on `event_key`. Machine payouts credit coin lots directly and are audited in `wp_pc_machine_events` — they never pass through the transaction ledger, because the player's history shows money movements only.
7. Money is UAH, stored `DECIMAL(12,2)` and serialised as decimal strings end to end. Never a JavaScript float.
8. `Machine_Service` is the only code that talks to Home Assistant. Its typed `WP_Error`s map to gateway statuses (502 / 503) so a machine fault never reaches a SPA as a 401 and never trips the refresh interceptor.
9. Meta keys exist only as constants: user meta through `User_Meta_Keys`, `pc_room` meta through `Post_Meta_Keys`. A literal meta-key string in a controller is a defect.
10. Moderation hides, it does not delete: a chat message flips its `status` column, a retired support subject is trashed rather than removed — so authors, bodies, IPs and old tickets' subject labels survive.
11. A schema change bumps `pc_db_version` in `Install_Schema` and updates `docs/DATA-MODEL.md` in the same commit.

## Features
This table is the router: a feature's docs are `docs/features/{feature}/`,
its code is the Code path below — resolve through the table, never through
the file hierarchy.

| Feature | Docs (FEATURE.md + sprints) | Code |
|---|---|---|
| `core` | `docs/features/core/` | `backend/wp-content/themes/pc/`, `frontend/src/`, `admin/src/` |
| `realtime` | `docs/features/realtime/` | `backend/wp-content/themes/pc/app/realtime/`, `frontend/src/services/realtime.js`, `admin/src/services/realtime.js` |
| `stripe` | `docs/features/stripe/` | `backend/wp-content/themes/pc/app/stripe/`, `admin/src/views/TopupsView.vue`, `admin/src/services/adminTopupService.js` |

## Commands
```bash
# Check command (docs/TECH-STACK.md -> Check command): backend/bin/check and frontend/bin/check.
# admin/ has none yet — its gate is `npm run lint && npm run build`.

# backend/ — WordPress under DDEV
ddev start
ddev wp pc seed-rooms
ddev wp pc machine-ingest --help
bin/check                         # php -l over the theme, then tests/ when DDEV is running

# frontend/ — player SPA, dev server on :5173
npm ci
npm run dev
bin/check                         # npm run lint, then npm run build
npm run lint                      # reports only
npm run lint:fix                  # rewrites files
npm run build

# admin/ — operator SPA, dev server on :5174
npm ci
npm run dev
npm run lint                      # reports only
npm run lint:fix                  # rewrites files
npm run build
```
