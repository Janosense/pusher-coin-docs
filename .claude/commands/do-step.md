---
description: Execute the plan of the current step. Running this command is the approval of the written plan. Scope frozen.
---

**Running `/do-step` is the approval.** The user is not asked to say
"approved" anywhere — `/plan-step` ends with the plan and `/do-step` runs it.
Never ask "do you approve?" and never wait for a confirmation before starting.

**Which step.** The step is the one whose plan `/plan-step` printed in this
session — feature, N and M come from that plan. If this session holds no such
plan: take the feature from the latest `docs/WORKLOG.md` entry, open
`docs/features/{feature}/sprints/SPRINT-{N}-PLAN.md` and look for sections in
state `awaiting approval` — exactly one: that is the step, run it; none or
several: STOP and ask which step to run. Never pick a step yourself.

**Preconditions — refuse to proceed if any fails:**
- The plan section has status `awaiting approval`.
- The section has no open Questions / ambiguities — or every open question
  carries a recommendation (the plan was printed with the
  "Open questions carry recommendations" line). Running `/do-step` means
  "as recommended": before
  executing, write that resolution into each question in the plan file
  (`Resolved: approved as recommended — {answer}`). A question the user
  answered differently in this session is resolved with the user's words.
  A question without a recommendation and without an answer makes the plan
  not runnable: print the question, STOP, and wait for the answer — then it
  belongs to `/plan-step`'s revision loop, not to this command.
- The user's last message before `/do-step` was not a pending change request
  to this plan. If the user wrote objections or answers that were never
  folded into the plan file (the file still shows the old text), STOP: fold
  them in first (as `/plan-step` step 7 does), print the plan again and end
  with `/plan-step`'s closing line — the user re-runs `/do-step` against the
  revised plan.

**Not this command.** If the user reports that a step's manual verification
failed and its plan section is `closed`, answer with
`/fix-step <what failed>` — never patch or re-run the step here. If the section is
`implemented, awaiting close`, the step is not closed yet: answer with
`/close-step` first (it writes the verification guide; the failure is then
reported against that guide via `/fix-step`).
A section in state `in progress — reopened: …` belongs to `/fix-step` too:
continue it there, not here.

Mark the plan section `(status: approved, in progress)` and execute:

1. **Branch.** Check out the task branch named in the plan's `### Branch`:
   if it does not exist, update the base and create the branch from it (if
   the base itself — the sprint branch — does not exist yet, create it first
   per the git model in root `CLAUDE.md`); if it exists, continue on it.
   Every commit of this step goes to the task branch — never to the base or
   to `main` directly.
2. **Scope is frozen.** Execute only the approved tasks. If mid-step you discover
   a needed dependency, a design flaw, a doc/reality mismatch, or scope growth:
   **stop immediately**, describe the problem and options, and wait. Do not
   silently expand or "fix along the way". A Claude Design handoff received
   in this session is input for the approved tasks only — it never adds
   tasks or screens; design files in `docs/features/{feature}/design/` are
   references, never copied into the code area.
3. **One task → working state → commit.** Build, lint and tests must be green
   before every commit. Conventional commit messages. Never leave the branch
   broken between tasks. The gate is the project's check command named in
   `docs/TECH-STACK.md` → Check command — one committed script that runs
   tests, lint, static analysis and build and exits non-zero on the first
   failure. Run it and commit only on exit code 0; never judge grepped
   output, and never replace it with an inline shell chain or function. No
   check command in the project yet → the plan's first task creates it; if
   the plan does not list that task, stop (rule 2).
4. **Tests are part of the task**, written in the same task as the code —
   especially for the profile's test-critical zones.
5. **Docs that describe the changed code are part of the task.** The entries
   of the plan's "Docs to update" that document what a task changes
   (`DATA-MODEL.md` for a schema change, `CONTRACTS.md` for an endpoint or
   payload shape, `ARCHITECTURE.md` for a new module/endpoint/flow,
   `TECH-STACK.md` for an approved dependency, `DESIGN.md` for a new or
   changed token or shared UI component) are updated in the same commit as
   the change (core rule 5).
6. **Tick the checkboxes** in `SPRINT-{N}-PLAN.md` as tasks complete.
7. Do **not** write the WORKLOG entry or the verification guide, tick the
   step in `SPRINT-{N}.md`, or merge the branch — that is `/close-step`'s job.

When all tasks are done: run the check command once more (exit 0), mark the
plan section `(status: implemented, awaiting close)`, and finish with:

> Step {M} implemented. Run `/close-step` to produce the report, verification guide and docs updates.
