# {{PROJECT_NAME}} — {{one-line description}}

<!-- playbook: v1.16 — sections marked PLAYBOOK CORE are copied verbatim from
     templates/CLAUDE.md in the playbook repo. Never edit them inside a
     project: fix them in the playbook, bump the version, propagate. -->

{{2–4 sentences: what the product does, who uses it, the one platform-level
truth if there is one (e.g. "HubSpot is the source of truth for CRM data").}}

## Project profile
- Origin: {{initialized from playbook | playbook adopted into existing codebase}}
- Verification: {{developer-reviewed — the user reads code and reviews diffs
  | user-verified — the user does not read code; explain all changes in plain
  language and write verification guides for a non-developer}}
- Deploy: {{manual — never assume push-to-deploy | describe pipeline}}
- Test-critical zones (code without tests here = unfinished task):
  {{e.g. payments/entitlements, access control, webhook idempotency, AI output validation}}
- Git model: {{chained sprint branches: {feature}/sprint-N ← main, task
  branches {feature}/sprint-N-short-name merged --no-ff | simple: task
  branch → main. Sprint numbers are per feature, so branch names carry the
  feature name}}. Environments (staging, production) track `main` — or the
  deployment branch named under Deploy — never a task or sprint branch. A
  step may ship deploy tooling; its first run is the sprint boundary's
  deploy from `main`, after the sprint is merged.

## Documentation (read before the relevant task)
| File | When to read |
|---|---|
| `docs/ARCHITECTURE.md` | Before structural work: new modules, endpoints, integrations, deploy questions |
| `docs/TECH-STACK.md` | Before adding dependencies or choosing an approach. Contains the ANTI-PATTERNS section — mandatory |
| `docs/DATA-MODEL.md` | Before any schema change, migration, query, or API response shape |
| `docs/DOMAIN.md` | Before implementing or changing any domain logic: the customer's terms, rules, invariants |
| `docs/CONTRACTS.md` (if present) | Before touching any endpoint, event, or payload shape |
| `docs/TESTING.md` (if present) | Before writing or changing tests |
| `docs/DESIGN.md` (if present) | Before any UI work: tokens, components, screen names. Design files in `docs/features/{feature}/design/` are references, never code to copy |
| `docs/DECISIONS.md` | Before proposing an architecture/tooling change — it may already be decided |
| `docs/features/{feature}/FEATURE.md` | Before any work in a feature: its scope, data ownership, invariants, interfaces |
| `docs/features/{feature}/sprints/SPRINT-N.md` | Current sprint scope and steps |
| `docs/features/{feature}/sprints/SPRINT-N-CLOSE.md` | At the first step of Sprint N+1: what Sprint N left behind — built, deferred, contradictions between docs |
| `docs/WORKLOG.md` | At session start: latest 5 entries (top of file) = project memory |
| `docs/LEARNINGS.md` | When something went wrong before — check if it's a known failure mode |
| {{project-specific docs, e.g. spike notes, client-plans}} | {{when}} |

## Core rules (PLAYBOOK CORE)
1. Plan before code. No code changes without an approved step plan (see Step protocol).
2. Small increments. One task → working state (the check command from `docs/TECH-STACK.md` exits 0) → conventional commit. Never leave the branch broken between tasks.
3. Code without tests = unfinished task. Mandatory for the test-critical zones in the profile. Tests are written in the same task as the code, by you, in the same session.
4. New dependencies only after explicit approval. Propose, explain why, wait for a "yes". This includes transitive tooling (linters, build plugins).
5. Do not "improve" without being asked. No speculative abstractions, caches, or extra layers. See an opportunity — propose it, don't do it silently.
6. Never hardcode business values. Prices, limits, intervals, texts that the business may change are configuration, not constants.
7. Secrets only via environment config. Never commit keys, never log secret values.
8. Documentation is part of the task. The /close-step docs self-check is not optional; a schema change without a DATA-MODEL.md update in the same commit is an unfinished task.
9. Session start ritual. Before anything else read the latest 5 `docs/WORKLOG.md` entries (top of file); the newest names the feature, sprint and step last worked on. Then, through the Features table, open that feature's `sprints/SPRINT-N.md` (which steps are ticked) and `SPRINT-N-PLAN.md` (the state of the step in flight: awaiting approval / in progress / implemented / closed). That is the project state — there is no separate status field. No WORKLOG entries = the project has not started.

## Step protocol (PLAYBOOK CORE)
The cycle is six commands — `/plan-step`, `/do-step`, `/close-step`,
`/fix-step`, `/adhoc` and, at the sprint boundary, `/close-sprint` — and each
defines its own procedure in `.claude/commands/`. This section only says
when they apply and what they do not authorize.
- The unit of work is **one Step** of the feature's `SPRINT-N.md` — never a
  whole sprint. If the user asks to "do the sprint" or "start the sprint":
  do not execute it; propose `/plan-step` for the first incomplete step.
- A step plan is produced only by `/plan-step` and executed only by
  `/do-step`. Approval of a step plan authorizes that step only — never the
  following steps.
- An implemented step is closed with `/close-step` before any other work begins.
- A closed step whose manual verification fails is re-opened only by
  `/fix-step <what failed>` (diagnosis → mini-plan → approval → fix on a
  `-reopen` branch; scope = the reported failure) and re-closed by
  `/close-step`. A failure report is never an instruction to patch the step
  directly.
- The next step begins only with a new `/plan-step` from the user.
- A sprint is closed only by `/close-sprint`, after its last `/close-step`:
  it writes `SPRINT-N-CLOSE.md`, ticks the Definition of Done with evidence
  (boundary items — deploy, observed outcomes — on the user's confirmation
  or with an explicit carry note), merges the sprint into `main` and names
  the next action. `/plan-step N+1 1` does not start before it; nobody
  ticks a Definition of Done box by hand.
- This protocol governs Claude Code coding sessions. Discovery chats (a Cowork
  Project on this directory whose Instructions are the playbook's
  `DISCOVERY.md`) see this file too, but follow those Instructions: they
  write docs only, never code, and are not bound by the step cycle.

Outside the step cycle (questions and ad-hoc tasks are normal, not violations):
- Questions (explain code, "why is X built this way", "what would it take
  to…", reading/analysis): answer anytime, freely, with no ceremony — but a
  question is never an instruction to change code. If the answer implies a
  change, say so and wait.
- Ad-hoc tasks (a change outside the current sprint's scope: hotfix, small
  tweak, config change): go through `/adhoc`. Never fold ad-hoc changes into
  an open step's commits or plan.

## Domain invariants
{{Numbered, non-negotiable, project-specific rules. Examples of the right kind:
"server-side entitlement checks on every endpoint", "all LLM responses validated
with Zod before touching DB", "persist message to DB first, then broadcast",
"webhook signature verification + idempotency". Delete this comment.}}

## Features
<!-- Always present, minimum one row — even a brand-new single-purpose project
     starts with one feature (name it after what it does, or "core"). Sprints
     never live in root docs/: always docs/features/{feature}/sprints/.
     This makes adding the second feature later a new row + a new folder,
     never a migration of existing files. -->
A feature is both an isolated module in the code (own subdirectory + bootstrap
inside a code area) and the unit of planning (own FEATURE.md, own sprint
numbering, own verification guides). Several features can share one code area
(e.g. two features inside one app or theme). This table is the router: /plan-step
resolves paths through it, not through the file hierarchy.

| Feature | Docs (FEATURE.md + sprints) | Code |
|---|---|---|
| {{core}} | `docs/features/{{core}}/` | `{{src/ or wp-content/...}}` |

Rules:
- Sprint numbering is independent per feature. WORKLOG/LEARNINGS entries are
  tagged with the feature name. WORKLOG and DECISIONS stay single/root.
- With exactly one feature, `/plan-step N M` resolves to it automatically;
  with several, the feature name is required — never guessed.
- Code areas that host several features (a shared app, package, or theme) get one
  `CLAUDE.md` of their own (per `templates/CLAUDE.area.md`) with isolation
  invariants: each feature in its own subdirectory with its own bootstrap;
  shared code (helpers, base styles, the area entry file beyond one
  registration line per feature) is changed only as an explicit plan task marked "touches shared code
  — may affect other features".
- Steps are closed sequentially, never in parallel sessions — WORKLOG is a
  shared resource.

## Commands
```bash
{{dev / lint / test / build / migrate commands — exact, copy-pasteable;
  the check command from docs/TECH-STACK.md first — it is the commit gate}}
```
