---
description: Plan ONE sprint step. Never writes code.
argument-hint: [feature] <sprint> <step>   # e.g. "1 3" or "event-calendar 1 3"
---

You are planning exactly one step. In this command it is FORBIDDEN to write or
edit application code, run migrations, or install anything. The only file you
may write is the plan file.

1. **Parse arguments:** `$ARGUMENTS` → optional feature name, sprint number N, step number M.
   Resolve the feature through the Features table in root `CLAUDE.md` — it is
   the router; never guess from the file hierarchy. If no feature name was
   given: with exactly one row in the table, use it; with several, STOP and
   ask which one — never pick one yourself. The row gives the feature's
   **Code** path; the feature's docs always live in `docs/features/{feature}/`:
   - `docs/features/{feature}/FEATURE.md` — the feature's architecture
   - `docs/features/{feature}/sprints/SPRINT-{N}.md` — the sprint
   - `docs/features/{feature}/sprints/SPRINT-{N}-PLAN.md` — the plan file this command writes

   There is no root `docs/sprints/` — a sprint file found there is a defect:
   report it, do not use it.

   If `docs/features/{feature}/sprints/SPRINT-{N}.md` does not exist — STOP.
   Never write a sprint file yourself; sprint files come only from discovery,
   which writes one file per sprint of the plan (since playbook v1.10):
   - the feature has no `sprints/` at all (e.g. `core` registered by Adoption
     — it has no sprints by design) → a feature with sprints must be created
     first via discovery (DISCOVERY → Feature mode);
   - the feature has earlier sprints but no Sprint {N}: if `FEATURE.md` →
     Roadmap lists it, the file is missing (project started before v1.10, or
     a Phase C defect) → a re-planning chat in the Cowork Project (DISCOVERY
     → Feature mode, Re-planning) writes it; if the Roadmap does not list it,
     the feature has no Sprint {N} → say so and stop.
   Tell the user which chat to open; do not offer to plan the sprint here.

2. **Read, in this order:**
   - the code area's `CLAUDE.md` (the feature's host directory), if the feature's Code path has one
   - `docs/features/{feature}/FEATURE.md` and, if the code area hosts OTHER
     features, their FEATURE.md files — the plan must state whether shared
     code is touched; such tasks are explicitly marked "touches shared code —
     may affect other features"
   - `docs/features/{feature}/sprints/SPRINT-{N}.md` — the whole file first
     (goal, fixed decisions, out of scope, dependencies), then focus Step {M}
   - the `docs/DECISIONS.md` entries referenced by the sprint's Fixed
     decisions — the plan must not reopen them; if a task would contradict
     one, flag it instead of deviating
   - the docs listed in the step's "Docs to update" / relevant to this step
     (ARCHITECTURE, DATA-MODEL, DOMAIN, TECH-STACK, CONTRACTS, TESTING,
     DESIGN as needed)
   - for a step that builds or changes a screen: `docs/DESIGN.md`, the
     `## UI` section of `docs/features/{feature}/FEATURE.md`, and the design
     files of that screen in `docs/features/{feature}/design/` — the plan
     names the screen as the design record does; design files are
     references, never files to copy into the code area
   - `docs/TECH-STACK.md` → **ANTI-PATTERNS** — always, regardless of the step:
     check every planned task against it and state in the plan that none is violated
     (or flag the conflict explicitly)
   - the latest 5 entries of `docs/WORKLOG.md` (top of file)
   - for M = 1 and N > 1: `docs/features/{feature}/sprints/SPRINT-{N-1}-CLOSE.md`
     (written by `/close-sprint` when Sprint {N-1} was closed) — what the
     previous sprint left behind; its Contradictions and Deferred items are
     Docs vs reality input for this plan
   - `docs/LEARNINGS.md` — check for known failure modes relevant to this step

3. **Inspect the actual codebase state** relevant to the step. Do not trust the
   docs blindly — if reality and docs disagree, say so explicitly in the plan.
   - The check command named in `docs/TECH-STACK.md` → Check command exists
     and runs. If the project has none yet, the first task of this plan
     creates it (one committed script: tests, lint, static analysis, build —
     exit code non-zero on the first failure).
   - A task that bootstraps a framework or skeleton into a non-empty
     repository: list the skeleton's top-level files in the plan and state
     what happens to each name that already exists (`CLAUDE.md`,
     `README.md`, `.gitignore`, agent instruction files such as `AGENTS.md`).
     A skeleton's agent file or bootstrap notes are never adopted; the
     dependencies they ask for go through core rule 1 like any other.
   - Versions in `docs/TECH-STACK.md` marked `(unverified — pinned at
     bootstrap)`: verify each against the package registry and list in the
     plan the locked version that TECH-STACK will record.
   - A step that builds or changes a screen: extract the text of that
     screen's artboards from the design export before listing strings. The
     brief fixes what it spells out; every other string comes from the
     artboard of the screen state; a difference between the two is a Docs vs
     reality item, never a silent choice. What the artboard of the step's
     screen state draws is the step's scope: a control or field the step
     text does not list is planned, not deferred and not asked about —
     unless the sprint's Out of scope or another step names it.

4. **Verify preconditions** — stop and report if any fails:
   - No step is in flight: neither this feature's `SPRINT-{N}-PLAN.md` nor
     the plan file named by the latest `docs/WORKLOG.md` entry (its feature
     and sprint) has a section in state `in progress` or
     `implemented, awaiting close` — an implemented step is closed with `/close-step`
     before any other work begins; steps are closed sequentially, never in
     parallel sessions.
   - All steps listed in Step {M} → "Depends on" are closed: their checkboxes
     in `SPRINT-{N}.md` are ticked (`### [x] Step … `).
   - Sprints run in order: for N > 1, every step of `SPRINT-{N-1}.md` is
     ticked. If not — say which steps of Sprint {N-1} are still open and stop.
   - Sprint {N-1} is closed by `/close-sprint` (M = 1, N > 1):
     `docs/features/{feature}/sprints/SPRINT-{N-1}-CLOSE.md` exists and every
     item of `SPRINT-{N-1}.md` → Definition of Done is ticked or carries
     `— carried to Sprint {N}: …`. If not — say which items are open and
     stop: "run `/close-sprint` first". Never tick a Definition of Done item
     here. Carried items go into this plan's Checks → Docs vs reality.
   - Step {M} itself is not already closed. If it is — say so; rework of a
     closed step goes through `/fix-step <what failed>`, not a new plan.

5. **Derive the plan from Step {M} of `SPRINT-{N}.md` only.** The step's
   subsections are the source:
   - **Tasks** → the plan's tasks: the same work, made concrete against the
     real codebase — split into commit-sized ordered tasks with exact files
   - **Tests** (+ the profile's test-critical zones) → the plan's tests
   - **Docs to update** → the plan's docs list
   - **Verification (manual)** → what the tasks must make possible
   Check the derived tasks and tests against the real code, not only against
   the step text:
   - Every task leaves the branch green on its own. When tasks build screens
     or modules that link to each other, register the routes / contracts in
     the first of them, or state in the plan how a link to a not-yet-existing
     target is written until it exists.
   - A step that adds, narrows or conditions on a rule other code depends on
     (permission / Policy, validation, invariant): read the whole rule set,
     not the diff. List the existing tests whose actor loses an ability and
     what each becomes; for every screen state conditioned on an ability,
     name the ability that already admitted the reader to the screen and how
     the two differ — if they cannot differ, the state is unreachable: no
     fallback task, no test.
   - A UI step: the plan says which component's DOM renders each pinned,
     conditional or interactive control, and at least one test asserts the
     state the user sees after the interaction (enabled / disabled, rendered
     / hidden), not only the markup at first paint.
   - An artefact the check command cannot execute (deploy script, cron line,
     server or hosting config): no invented local test — name in Checks →
     Not locally verifiable the one real run that verifies it. Deploying an
     environment is never a task of a step: environments track `main` (git
     model in root `CLAUDE.md`), so the first run of deploy tooling is the
     sprint-boundary deploy; a step task that says "deploy" goes to
     Questions / ambiguities with the recommendation to move it to the
     sprint's Definition of Done.
   The plan never adds work the step does not name (what the artboard of the
   step's screen draws is named by the step — §3). Work that turns out
   necessary but is missing from the step → "Questions / ambiguities" (the
   user decides: this step, another step, or the next sprint). Work from
   other steps or from the sprint's Out of scope → never.

   **What is a question.** An item goes to Questions / ambiguities only when
   both hold: the plan's tasks differ depending on the answer, and no source
   settles it — the sprint (goal, Fixed decisions, Out of scope, this and
   the other steps), FEATURE.md, DECISIONS.md, DATA-MODEL, DESIGN.md / the
   artboard, the shipped code. Everything else is not a question:
   - a mismatch between two sources that the precedence rules settle (brief
     over artboard for strings; DECISIONS over everything; the data model
     over what a screen happens to draw; a string key already shipped for
     the same rule over a second wording of it) → resolve it and record it
     in Checks → Docs vs reality / Design with the resolution taken;
   - an observation outside this step whose recommendation is "leave it for
     another step, the Definition of Done or `/adhoc`" → one line in
     Checks → Docs vs reality (nothing, if it is not a mismatch) — it changes
     no task, so it is not a question;
   - a choice with one defensible answer → make it in the task and say so
     there.
   `none` is the expected content of the section for most steps; a plan is
   not better for having questions. A question that survives names the two
   (or more) answers, what changes in the tasks under each, and a
   recommendation where one exists.

6. **Write the plan** into `docs/features/{feature}/sprints/SPRINT-{N}-PLAN.md`
   (add or replace the section for this step; never touch other steps'
   sections), using this structure:

   ```
   ## Plan — Sprint {N}, Step {M}: {title}   (status: awaiting approval)
   ### Branch
   `{task branch}` ← `{base}`   (per the git model in root `CLAUDE.md`;
   base = the **Branch** of `SPRINT-{N}.md`, e.g. `{feature}/sprint-{N}-short-name` ← `{feature}/sprint-{N}`)
   ### Tasks (ordered)
   - [ ] task → expected commit
   (a task that changes shared code is marked "touches shared code — may
   affect other features" and names the consuming features)
   ### Files to create/change
   ### Tests to write
   ### Docs to update
   ### Checks
   - ANTI-PATTERNS: none violated | conflict: …
   - Docs vs reality: match | mismatch: …
   - Design: n/a | matches `{screen}` in DESIGN.md / FEATURE.md → UI | deviation: …
   - Check command: `{command}` (docs/TECH-STACK.md → Check command) | missing — created in task 1
   - Not locally verifiable: n/a | {artefact} — verified by {the one real run: sprint-boundary deploy from `main`, cron tick, …}
   ### Questions / ambiguities
   none | (each question with a recommended answer where one exists; a question
   without a recommendation makes the plan NOT approvable until answered)
   ```

7. **Print the plan in the chat** (plain language if profile = user-verified), then **STOP** with exactly this closing line:

   > Awaiting approval. Approval authorizes **Step {M} only**. After approval, run `/do-step`.

   If the plan has open questions and every one of them carries a
   recommendation, print this line directly above the closing line:

   > Open questions carry recommendations — "approved" accepts them as written; name any you want changed.

   If any question has no recommendation, the plan is not approvable: ask for
   the answers instead of printing the closing line.
   If the user answers the open questions, fold the answers into the plan
   file (status stays `awaiting approval`), print it again and stop with the
   same line — approval always refers to the written plan.
