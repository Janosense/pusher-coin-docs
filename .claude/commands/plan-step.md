---
description: Plan one sprint step. Never writes code.
argument-hint: [feature] <sprint> <step>
disable-model-invocation: true
---

You are planning exactly one step. This command never writes or edits
application code, runs migrations, or installs anything; the only file it
writes is the plan file.

1. **Parse arguments:** `$ARGUMENTS` → optional feature name, sprint number
   N, step number M (e.g. `1 3` or `event-calendar 1 3`).
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
   which writes one file per sprint of the plan. Do not offer to plan the
   sprint here; tell the user what to do:
   - the feature has no `sprints/` at all (e.g. `core` registered by Adoption
     — it has no sprints by design) → a feature with sprints must be created
     first via discovery (DISCOVERY → Feature mode);
   - the feature has earlier sprints and `FEATURE.md` → Roadmap lists
     Sprint {N} → the file is missing: a re-planning chat in the Cowork
     Project (DISCOVERY → Feature mode, Re-planning) writes it;
   - the Roadmap does not list Sprint {N} → the feature has no Sprint {N}:
     say so.

2. **Read, in this order:**
   - the code area's `CLAUDE.md` (the feature's host directory), if the feature's Code path has one
   - `docs/features/{feature}/FEATURE.md` and, if the code area hosts OTHER
     features, their FEATURE.md files — the plan must state whether shared
     code is touched; such tasks are explicitly marked "touches shared code —
     may affect other features"
   - `docs/features/{feature}/sprints/SPRINT-{N}.md` — the whole file first
     (goal, fixed decisions, dependencies), then focus Step {M}
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
   - `docs/TECH-STACK.md` → **ANTI-PATTERNS** and **CONVENTIONS** — always,
     regardless of the step: check every planned task against both and state
     in the plan that none is violated (or flag the conflict explicitly)
   - the latest 5 entries of `docs/WORKLOG.md` (top of file)
   - for M = 1 and N > 1: the WORKLOG entry of Sprint {N-1}'s last step
     (it ends with `Sprint {N-1} complete`) — the open questions it carries
     are Docs vs reality input for this plan

3. **Inspect the actual codebase state** relevant to the step. Do not trust the
   docs blindly — if reality and docs disagree, say so explicitly in the plan
   (Checks → Docs vs reality).

   Always:
   - The check command named in `docs/TECH-STACK.md` → Check command exists
     and runs. If the project has none yet, the first task of this plan
     creates it: one committed script that runs tests, lint, static analysis
     and build and exits non-zero on the first failure.

   Then each of these whose condition holds for this step:
   - **A task bootstraps a framework or skeleton into a non-empty
     repository** → list the skeleton's top-level files in the plan and, for
     each name that already exists (`CLAUDE.md`, `README.md`, `.gitignore`,
     agent instruction files such as `AGENTS.md`), state what happens to it.
     Never adopt the skeleton's agent file or bootstrap notes; a dependency
     they ask for needs explicit approval like any other (root `CLAUDE.md` →
     Core rules).
   - **`docs/TECH-STACK.md` has versions marked `(unverified — pinned at
     bootstrap)`** → verify each against the package registry and list in
     the plan the locked version that TECH-STACK will record.
   - **The step builds or changes a screen** →
     1. extract the text of that screen's artboards from the design export
        before listing strings;
     2. strings: the brief's wording wherever the brief spells it out, the
        artboard of the screen state for every other string; record each
        difference between the two under Checks → Docs vs reality — never a
        silent choice;
     3. scope: What the artboard of the step's screen state draws is the
        step's scope. Plan every control or field it draws, including one
        the step text does not list — do not defer it or ask about it —
        unless another step or `FEATURE.md` → Roadmap names it.

4. **Verify preconditions** — stop and report if any fails:
   - No step is in flight: neither this feature's `SPRINT-{N}-PLAN.md` nor
     the plan file named by the latest `docs/WORKLOG.md` entry (its feature
     and sprint) has a section in state `in progress` or
     `implemented, awaiting verification`, and no task branch named in a
     plan section's `### Branch` is left unmerged into its base
     (`git branch --no-merged`) — the plan file on the base may not show a
     step that awaits verification on its branch. An implemented step is
     verified by the user and closed with `/close-step` before another step
     is planned; steps are closed sequentially, never in parallel sessions.
   - All steps listed in Step {M} → "Depends on" are closed: their checkboxes
     in `SPRINT-{N}.md` are ticked (`### [x] Step … `).
   - Sprints run in order: for N > 1, every step of `SPRINT-{N-1}.md` is
     ticked. If not — say which steps of Sprint {N-1} are still open and stop.
   - Sprint {N-1} is on `main` (M = 1, N > 1): with chained sprint
     branches, the branch of `SPRINT-{N-1}.md` is merged into `main`
     (`git branch --merged main` lists it) — `/close-step` of its last step
     does that. If not — stop: "Sprint {N-1} is not merged into `main` —
     run `/close-step`: it resumes the unfinished merge". Never merge it
     here.
   - Step {M} itself is not already closed. If it is — say so; rework of a
     closed step goes through `/fix-step <what failed>`, not a new plan.

5. **Derive the plan from Step {M} of `SPRINT-{N}.md` only.** The step's
   subsections are the source:
   - **Tasks** → the plan's tasks: the same work, made concrete against the
     real codebase — split into commit-sized ordered tasks with exact files
   - **Tests** (+ the Test-critical zones of root `CLAUDE.md` → Project
     profile) → the plan's tests
   - **Docs to update** → the plan's docs list
   - **Verification (manual)** → what the tasks must make possible
   Check the derived tasks and tests against the real code, not only against
   the step text.

   Always:
   - Every task leaves the branch green on its own. Tasks that build screens
     or modules linking to each other register the routes / contracts in the
     first of them, or the plan states how a link to a not-yet-existing
     target is written until it exists.

   Then each of these whose condition holds for this step:
   - **The step adds, narrows or conditions on a rule other code depends
     on** (permission / Policy, validation, invariant) → read the whole rule
     set, not the diff, and put in the plan:
     1. every existing test whose actor loses an ability, and what that test
        becomes;
     2. for every screen state conditioned on an ability: the ability that
        already admitted the reader to the screen, and how the two differ.
        If they cannot differ, the state is unreachable — no fallback task,
        no test.
   - **The step builds or changes UI** → the plan names the component whose
     DOM renders each pinned, conditional or interactive control, and at
     least one test asserts the state the user sees after the interaction
     (enabled / disabled, rendered / hidden), not only the markup at first
     paint.
   - **The step creates an artefact the check command cannot execute**
     (deploy script, cron line, server or hosting config) → no invented
     local test; name under Checks → Not locally verifiable the one real run
     that verifies it — for deploy tooling, the next deploy from `main`,
     done by the user.
   - **A step task says "deploy"** → Deploying an environment is never a
     task of a step: environments track `main` (git model in root
     `CLAUDE.md`). Put the task under Questions / ambiguities with the
     recommendation to drop it from the step.

   The plan never adds work the step does not name (what the artboard of the
   step's screen draws is named by the step — §3). Work that turns out
   necessary but is missing from the step → "Questions / ambiguities" (the
   user decides: this step, another step, or the next sprint). Work from
   other steps, or work that `FEATURE.md` → Roadmap places in a later
   sprint → never.

   **What is a question.** An item goes to Questions / ambiguities only when
   both hold: the plan's tasks differ depending on the answer, and no source
   settles it — the sprint (goal, Fixed decisions, this and the other
   steps), FEATURE.md, DECISIONS.md, DATA-MODEL, DESIGN.md / the
   artboard, the shipped code. Everything else is not a question:
   - a mismatch between two sources that the precedence rules settle (brief
     over artboard for strings; DECISIONS over everything; the data model
     over what a screen happens to draw; a string key already shipped for
     the same rule over a second wording of it) → resolve it and record it
     in Checks → Docs vs reality / Design with the resolution taken;
   - an observation outside this step whose recommendation is "leave it for
     another step or `/adhoc`" → one line in
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
   - ANTI-PATTERNS / CONVENTIONS: none violated | conflict: …
   - Docs vs reality: match | mismatch: …
   - Design: n/a | matches `{screen}` in DESIGN.md / FEATURE.md → UI | deviation: …
   - Check command: `{command}` (docs/TECH-STACK.md → Check command) | missing — created in task 1
   - Not locally verifiable: n/a | {artefact} — verified by {the one real run: the next deploy from `main`, cron tick, …}
   ### Questions / ambiguities
   none | (each question with a recommended answer where one exists; a question
   without a recommendation makes the plan NOT approvable until answered)
   ```

7. **Print the plan in the chat** (plain language if the profile says
   `user-verified`), then **STOP** with exactly this closing line:

   > Plan for **Step {M}** written. `/do-step` runs it as written — running it is the approval, and it authorizes Step {M} only. Anything else you write is a change request to the plan.

   If the plan has open questions and every one of them carries a
   recommendation, print this line directly above the closing line:

   > Open questions carry recommendations — `/do-step` accepts them as written; name any you want changed.

   If any question has no recommendation, the plan is not approvable: ask for
   the answers instead of printing the closing line (`/do-step` refuses such
   a plan).
   Do not ask for approval and do not wait for an "approved" — the user's
   next move is either `/do-step` or a message. Any message that is not
   `/do-step` (answers to the questions, objections, edits — even a short
   one) is a change request: fold it into the plan file (status stays
   `awaiting approval`), print the plan again and stop with the same closing
   line. Approval always refers to the written plan, never to the chat. A
   report that an item of a step's verification guide fails is not a change
   request: Read `.claude/commands/fix-step.md` and follow it.
