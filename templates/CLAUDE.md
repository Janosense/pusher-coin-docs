# {{PROJECT_NAME}} — {{one-line description}}

<!-- playbook: v1.21 — Core rules and Step protocol are verbatim copies of
     templates/CLAUDE.md; never edit them here. -->

{{2–4 sentences: what the product does, who uses it, the one platform-level
truth if there is one (e.g. "HubSpot is the source of truth for CRM data").}}

## Project profile
- Verification: {{omit the line if the user reads code | user-verified — the
  user does not read code: plans, reports and verification guides in plain
  language, for a non-developer}}
- Deploy: {{manual — never assume push-to-deploy | describe pipeline}}
- Test-critical zones (code without tests here = unfinished task):
  {{e.g. payments/entitlements, access control, webhook idempotency, AI output validation}}
- Git model: {{chained sprint branches: {feature}/sprint-N ← main, task
  branches {feature}/sprint-N-short-name merged --no-ff | simple: task
  branch → main. Sprint numbers are per feature, so branch names carry the
  feature name}}. Environments track `main` (or the deployment branch named
  under Deploy) — never a task or sprint branch; a deploy is the user's own
  action from `main`, never a step task.

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
| `docs/WORKLOG.md` | At session start: latest 5 entries (top of file) = project memory |
| `docs/LEARNINGS.md` | When something went wrong before — check if it's a known failure mode |
| {{project-specific docs, e.g. spike notes, client-plans}} | {{when}} |

## Core rules
1. New dependencies only after explicit approval. Propose, explain why, wait for a "yes". This includes transitive tooling (linters, build plugins).
2. Do not "improve" without being asked. No speculative abstractions, caches, or extra layers. See an opportunity — propose it, don't do it silently.
3. Never hardcode business values. Prices, limits, intervals, texts that the business may change are configuration, not constants.
4. Secrets only via environment config. Never commit keys, never log secret values.
5. Documentation is part of the task. Docs that describe changed code are updated in the same commit as the change; a schema change without a `docs/DATA-MODEL.md` update is an unfinished task.
6. Project state is derived, never asked for: the newest `docs/WORKLOG.md` entry names the feature, sprint and step; that feature's `sprints/SPRINT-N.md` shows which steps are ticked, `SPRINT-N-PLAN.md` the step in flight (awaiting approval / in progress / implemented / closed). Derive it before any work on the project — not before answering a question. No WORKLOG entries = the project has not started.

## Step protocol
- Code is changed only inside a step (`/plan-step` → `/do-step`) or an
  `/adhoc`. The unit of work is **one Step** of the feature's `SPRINT-N.md` —
  never a whole sprint. If the user asks to "do the sprint" or "start the sprint":
  do not execute it; propose `/plan-step` for the first incomplete step.
- A step plan is produced only by `/plan-step` and executed only by
  `/do-step`. Running `/do-step` is the approval of the written plan — the
  user is never asked to say "approved"; any other message after the plan is
  a change request. Approval authorizes that step only — never the
  following steps.
- An implemented step is closed with `/close-step` before any other work begins.
- A closed step whose manual verification fails is re-opened only by
  `/fix-step <what failed>` and re-closed by `/close-step`. A failure report
  is never an instruction to patch the step directly.
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
{{Numbered, non-negotiable, project-specific rules — e.g. "server-side
entitlement checks on every endpoint", "all LLM responses validated with Zod
before touching DB", "persist message to DB first, then broadcast".}}

## Features
This table is the router: a feature's docs are `docs/features/{feature}/`,
its code is the Code path below — resolve through the table, never through
the file hierarchy.

| Feature | Docs (FEATURE.md + sprints) | Code |
|---|---|---|
| {{core}} | `docs/features/{{core}}/` | `{{src/ or wp-content/...}}` |

## Commands
```bash
{{check command from docs/TECH-STACK.md first (the commit gate), then
  dev / lint / test / build / migrate — exact, copy-pasteable}}
```
