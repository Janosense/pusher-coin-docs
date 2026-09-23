---
description: Execute a small task outside the current sprint scope (hotfix, tweak, config). Isolated from the step cycle.
argument-hint: <short description of the task>
disable-model-invocation: true
---

The task: $ARGUMENTS

An ad-hoc task is a change that does not belong to the current sprint's steps.
It must never contaminate an open step's plan, branch or commits.

0. **Enter plan mode.** Call the `EnterPlanMode` tool before anything else.
   Steps 1–2 are read-only analysis and planning: no file, git or shell
   change until the mini-plan is approved. Tool unavailable (non-interactive
   session): keep the same discipline and print the mini-plan in the chat.
1. **Scope check first.** No task given above → ask what to change and
   stop. Otherwise read the latest `docs/WORKLOG.md` entry (it names
   the feature, sprint and step last worked on), then that feature's
   `docs/features/{feature}/sprints/SPRINT-{N}.md` and `SPRINT-{N}-PLAN.md`
   (the step in flight and its state). If the request actually belongs to
   the current sprint's scope, or is too large for an ad-hoc (touches schema,
   shared code across features, or needs more than ~3 commits): STOP and
   propose adding it to the sprint plan or the next sprint instead. Do not
   execute. A failed item of a step's verification guide — before or after
   the step's close — is not an ad-hoc either → `/fix-step <what failed>`.
2. **Mini-plan** (plain language if root `CLAUDE.md` → Project profile
   says `user-verified`): what will change, files, branch
   `adhoc/{short-name}` ← base, whether tests are needed (yes for anything
   in the profile's Test-critical zones), risks.
   Nothing goes into `SPRINT-{N}-PLAN.md`. Present it with `ExitPlanMode`
   and **wait for explicit approval** — the approval of the plan is the
   approval; do not ask again in the chat.
3. **State isolation.** If a step is `in progress` or
   `implemented, awaiting verification`, its task branch must be left clean:
   commit finished tasks, stash unfinished work; when the ad-hoc is done,
   check the task branch out again and restore the stash. Ad-hoc work
   happens on its own branch `adhoc/{short-name}` created from the branch
   the fix targets (per the git model in root `CLAUDE.md`: `main` for a
   hotfix of what is released, the sprint branch for a fix of sprint work)
   — never on a step's task branch.
4. **Execute:** small increments, tests where required, conventional commits
   prefixed `fix:` / `chore:`. Before every commit run the check command
   (`docs/TECH-STACK.md` → Check command) and commit only on exit code 0 —
   judge the exit code, never grepped output.
5. **Close:** docs self-check — Read `.claude/commands/close-step.md`,
   step 2 (**Docs self-check**), and apply its checklist to this change; a
   doc it names is updated in the same commit as the change it describes.
   Add a WORKLOG entry at the TOP of `docs/WORKLOG.md` tagged `[adhoc]`
   (+ feature if applicable); merge the ad-hoc branch into its base with
   `--no-ff` and delete it (a conflict limited to entries added at the TOP
   of `docs/WORKLOG.md` → keep both, newest on top; any other conflict →
   `git merge --abort`, report the conflicting files and stop). Do not
   touch `SPRINT-{N}.md` or `SPRINT-{N}-PLAN.md` — the step/sprint state is
   unchanged.
6. **Report** (plain language if the profile says `user-verified`): what
   changed, commits, merged into which branch, how to verify manually, then
   remind:

   > Ad-hoc done. Sprint state unchanged: [feature] Sprint {N}, Step {M} ({state}) — or "no step in flight".
