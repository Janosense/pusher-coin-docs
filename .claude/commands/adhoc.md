---
description: Execute a small task OUTSIDE the current sprint scope (hotfix, tweak, config). Isolated from the step cycle.
argument-hint: <short description of the task>
---

An ad-hoc task is a change that does not belong to the current sprint's steps.
It must never contaminate an open step's plan, branch or commits.

0. **Enter plan mode.** Call the `EnterPlanMode` tool before anything else.
   Steps 1–2 are read-only analysis and planning: no file, git or shell
   change until the mini-plan is approved. Tool unavailable (non-interactive
   session): keep the same discipline and print the mini-plan in the chat.
1. **Scope check first.** Read the latest `docs/WORKLOG.md` entry (it names
   the feature, sprint and step last worked on), then that feature's
   `docs/features/{feature}/sprints/SPRINT-{N}.md` and `SPRINT-{N}-PLAN.md`
   (the step in flight and its state). If the request actually belongs to
   the current sprint's scope, or is too large for an ad-hoc (touches schema,
   shared code across features, or needs more than ~3 commits): STOP and
   propose adding it to the sprint plan or the next sprint instead. Do not
   execute. A failure of what a closed step promised (its verification
   guide does not hold) is not an ad-hoc either → `/fix-step <what failed>`.
2. **Mini-plan** (plain language if profile = user-verified): what will
   change, files, branch `adhoc/{short-name}` ← base, whether tests are
   needed (yes for anything in the profile's test-critical zones), risks.
   Nothing goes into `SPRINT-{N}-PLAN.md`. Present it with `ExitPlanMode`
   and **wait for explicit approval** — the approval of the plan is the
   approval; do not ask again in the chat.
3. **State isolation.** If a step is `in progress`, its task branch must be
   left clean: commit finished tasks, stash unfinished work (restore it when
   the ad-hoc is done). Ad-hoc work happens on its own branch
   `adhoc/{short-name}` created from the branch the fix targets (per the git
   model in root `CLAUDE.md`: `main` for a hotfix of what is released, the
   sprint branch for a fix of sprint work) — never on a step's task branch.
4. **Execute:** small increments, tests where required, conventional commits
   prefixed `fix:` / `chore:`; every commit gated by the check command
   (`docs/TECH-STACK.md` → Check command), as in `/do-step`.
5. **Close:** docs self-check (same checklist as /close-step §2 — an ad-hoc
   schema or architecture change still updates docs in the same commit);
   add a WORKLOG entry at the TOP of `docs/WORKLOG.md` tagged `[adhoc]`
   (+ feature if applicable); merge the ad-hoc branch into its base with
   `--no-ff` and delete it (conflict → stop and report). Do NOT touch
   `SPRINT-{N}.md` or `SPRINT-{N}-PLAN.md` — the step/sprint state is
   unchanged.
6. **Report** (plain language if profile = user-verified): what changed,
   commits, merged into which branch, how to verify manually, then remind:

   > Ad-hoc done. Sprint state unchanged: [feature] Sprint {N}, Step {M} ({state}) — or "no step in flight".
