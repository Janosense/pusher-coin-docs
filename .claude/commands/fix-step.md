---
description: Re-open a closed step whose manual verification failed — diagnose, mini-plan, approval, fix. Scope = the reported failure only. Re-closed by /close-step.
argument-hint: <what failed>   # e.g. "guide item 3: notes:add '   ' creates a note instead of exiting 1"
---

A fix is rework of a step that was already closed: the user followed the
verification guide (or used the feature) and something the step promised
does not hold. This command never plans new work and never executes
an approved plan — `/plan-step` and `/do-step` do that.

**Enter plan mode.** Call the `EnterPlanMode` tool before anything else.
Everything up to the approved mini-plan is read-only: no write to the
working tree, no git change, no install. Reading files, running the guide's
item, the tests or the failing command is allowed. A reproduction that would
write to the working tree waits for the regression test of step 4. Tool
unavailable (non-interactive session): keep the same discipline and print
the mini-plan in the chat.

**Which step.** The failure belongs to the step whose verification guide or
whose Verification (manual) promised the behaviour. The user's report names
it (or the guide `docs/features/{feature}/verification/sprint-{N}-step-{M}.md`
it followed); if it does not: take the feature from the latest
`docs/WORKLOG.md` entry, open
`docs/features/{feature}/sprints/SPRINT-{N}-PLAN.md` and match the report
against the closed sections' verification guides — exactly one candidate:
name it and continue; none or several: STOP and ask which step. Never pick a
step yourself.

**Preconditions — refuse to proceed if any fails:**
- `$ARGUMENTS` describes what failed: what was done, what was expected, what
  happened instead (a guide item number is enough when it is precise). Empty
  or "something is broken" → ask for the observation first; do not go looking
  for bugs.
- The step's plan section is `closed`. `implemented, awaiting close` → the
  step is not closed yet: run `/close-step` first (it writes the guide the
  verification follows; its task branch is still unmerged, so a `-reopen`
  branch would lose it). `awaiting approval` or `in progress` → the step is
  in flight: not a fix — finish it with `/do-step` (its rule 2 handles
  surprises). No plan section at all → the step was never done →
  `/plan-step`.
- No step is in flight: the latest `docs/WORKLOG.md` entry's feature has no
  plan section in state `in progress` or `implemented, awaiting close` (in
  any of its `SPRINT-{N}-PLAN.md` files) — close it first. Steps are closed
  sequentially, never in parallel sessions.
- The failure is inside this step: the step's Verification (manual), its
  verification guide, or its tasks promise the behaviour. Behaviour the step
  never promised is not a fix: released behaviour or another feature →
  `/adhoc`; a missing capability → not a bug — say which sprint or step owns
  it (or that none does) and stop.

1. **Diagnose — no code changes.** Reproduce the failure (run the guide's
   item, the failing command, or a scratch check — nothing committed), read
   the code and tests the step touched, and find the **cause**, not the
   symptom: which file and line does what, and why the tests of the step did
   not catch it. Then classify:
   - **Code defect** — the sprint step, the plan, DECISIONS and the data
     model describe the behaviour the user expects; the code does not do it
     → continue to the mini-plan.
   - **The step or plan is wrong** — the code does what `SPRINT-{N}.md`, the
     plan section, a DECISIONS entry or `DATA-MODEL.md` says, and the user's
     report expects something else → STOP. This is a scope change, not a
     fix: quote the two sides (which file says what, what the report
     expects) and the options — re-planning of this step's scope in the
     Cowork Project (DISCOVERY → Feature mode, Re-planning) with a
     DECISIONS entry, a step of this sprint that already owns the
     behaviour, or the next sprint — with a recommendation, and wait.
     Never rewrite `SPRINT-{N}.md`, the plan section's tasks, or a
     DECISIONS entry yourself.
   - **The guide is wrong, the code is right** → say so; the mini-plan has
     one task (correct the guide) and `/close-step` rewrites it on re-close.
   - **Outside the step** (see preconditions) → name the right path and stop.

2. **Mini-plan** in the chat, not in the plan file (plain language if
   profile = user-verified):
   ```
   ## Fix — Sprint {N}, Step {M}: {title}
   - Failure: {the user's report, in one line}
   - Cause: {file:line — what is wrong and why the step's tests missed it}
   - Tasks (ordered, each → one commit):
     1. regression test that reproduces the failure (red) → `test: …`
     2. the fix (green) → `fix: …`
     (+ docs task if a doc describes the changed behaviour)
   - Files:
   - Branch: `{task branch}-reopen` ← `{base}`   (both from the plan's `### Branch`; `-2`, `-3`… if it exists)
   - Touches shared code: no | yes — {consuming features}
   - Also affected by the cause: none | {other places the same cause reaches — fixed here only if the step promised them; otherwise listed for `/adhoc` or the sprint}
   - Risks:
   ```
   The regression test comes first whenever the failure is testable — always
   in the profile's test-critical zones. A fix that needs more than ~3
   commits, a schema change, or shared code of another feature is said so
   explicitly: the user decides whether it is still a fix of this step or
   goes to the sprint. Present the mini-plan with `ExitPlanMode` and
   **wait for explicit approval** — the approval of the plan is the
   approval; ambiguous → ask, do not infer.

3. **Re-open.** Mark the plan section
   `(status: approved, in progress — reopened: {what failed})` and append to
   it a `### Reopen — {date}: {what failed}` subsection with the cause and
   the approved tasks as checkboxes (never edit the original tasks). If the
   section already has a Reopen subsection, add a new one below it.

4. **Execute as `/do-step` does — scope frozen to the approved tasks.**
   Update the base and create `{task branch}-reopen` from it (the original
   task branch was merged and deleted by `/close-step`); every commit goes
   there. One task → check command (`docs/TECH-STACK.md` → Check command)
   exit 0 → conventional commit (`test:` / `fix:` / `docs:`). Docs that
   describe the changed code are updated in the same commit (core rule 5).
   Tick the Reopen checkboxes as tasks complete. Anything the diagnosis
   missed — a second cause, a needed dependency, a mismatch — stop, describe
   it and wait; a fix never grows silently.

5. **Finish.** Run the check command once more, mark the section
   `(status: implemented, awaiting close — reopened: {what failed})` and end
   with:

   > Fix for Step {M} implemented. Run `/close-step` to re-close: `re-closed` WORKLOG entry, rewritten verification guide, merge of `{task branch}-reopen` into `{base}`.

   Do not write the WORKLOG entry, rewrite the guide, or merge — that is
   `/close-step`'s job; the guide's rewrite must make the failed item an
   explicit check.
