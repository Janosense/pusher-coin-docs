---
description: Fix a failed item of a step's verification guide — diagnose, mini-plan, approval, fix. Before the step's close the fix lands on its task branch; after the close, on a fix branch this command merges. Scope = the reported failure only.
argument-hint: <what failed>
disable-model-invocation: true
---

**Enter plan mode.** Call the `EnterPlanMode` tool before anything else.
Everything up to the approved mini-plan is read-only: no write to the
working tree, no git change, no install. Reading files, running the guide's
item, the tests or the failing command is allowed. A reproduction that would
write to the working tree waits for the regression test of step 4. Tool
unavailable (non-interactive session): keep the same discipline and print
the mini-plan in the chat.

The failure: $ARGUMENTS — or the user's chat message, when the failure was
reported without the command. A fix makes a step do what its verification
guide or Verification (manual) promised; it never plans new work or runs a
step plan (`/plan-step`, `/do-step`). Two modes, by the state of the step's
plan section:
- **Before the close** (`implemented, awaiting verification`): the fix is
  committed on the step's task branch; the user verifies again and runs
  `/close-step`.
- **After the close** (`closed`): the fix goes on a fix branch, which this
  command merges.

**Which step.** A section in state `in progress — verification fix: …`, or
a `fix/…` branch still unmerged, is an **Interrupted fix** (below): that
step, no new report needed. A step `implemented, awaiting verification` —
implemented in this session, the one whose task branch (its `### Branch`)
is checked out, or the one in that state in the `SPRINT-{N}-PLAN.md` files
(every N) of the latest `docs/WORKLOG.md` entry's feature — owns every
reported failure, even of behaviour an earlier step promised: its task
branch is what the user runs (step 1 checks that the cause is in its
changes). Otherwise the failure belongs to the closed step whose guide or
Verification (manual) promised the behaviour — named by the report (or by
the guide `docs/features/{feature}/verification/sprint-{N}-step-{M}.md` it
followed), else matched against the closed sections' guides of that
feature: exactly one candidate → name it and continue; none or several →
STOP and ask which step. Never pick a step yourself.

**Preconditions — refuse to proceed if any fails:**
- The report says what failed: what was done, what was expected, what
  happened instead (a guide item number is enough when it is precise, e.g.
  "guide item 3: `notes:add '   '` creates a note instead of exiting 1").
  Empty or "something is broken" → ask for the observation first; do not go
  looking for bugs. An interrupted fix needs no new report.
- The step's plan section is `implemented, awaiting verification`,
  `closed`, or `in progress — verification fix: …` (**Interrupted fix**,
  below). `awaiting approval` or `approved, in progress` → the step is in
  flight: not a fix — finish it with `/do-step`. No plan section at all →
  the step was never done → `/plan-step`.
- No other plan section of the latest `docs/WORKLOG.md` entry's feature is
  `in progress` — finish it first; steps are closed sequentially, never in
  parallel sessions.
- The failure is inside this step: the step's Verification (manual), its
  verification guide, or its tasks promise the behaviour — or, before the
  close, its changes broke anything that worked before (an earlier step,
  another feature, released behaviour). Behaviour no step promised (code
  from before the playbook, such as the `core` feature) → `/adhoc`; a
  missing capability → not a bug — say which sprint or step owns it (or
  that none does) and stop.

**Interrupted fix.** A section whose latest `### Verification fix` has
unticked tasks (before the close its status reads
`in progress — verification fix: …`), or whose fix branch still exists
after the close (a stopped merge), continues at step 4 or at the merge of
step 5 — no new diagnosis.

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
     expects) and the options, with a recommendation: keep the step as
     built (before the close: `/close-step`); the change in a step of this
     sprint that already owns the behaviour; or a new step or the next
     sprint, written with a DECISIONS entry by a re-planning chat in the
     Cowork Project (DISCOVERY → Feature mode, Re-planning). Wait. Never
     rewrite `SPRINT-{N}.md`, the plan section's tasks, or a DECISIONS
     entry yourself.
   - **The guide is wrong, the code is right** → say so; the mini-plan has
     one task: correct the guide.
   - **Cause already on the base** — before the close, an earlier step's
     behaviour fails and this step's changes did not cause it → STOP: this
     step is verified and closed first; the failure is then a fix after the
     close of the step that promised it.
   - **Outside the step** (see preconditions) → name the right path and stop.

2. **Mini-plan** in the chat, not in the plan file (plain language if root
   `CLAUDE.md` → Project profile says `user-verified`):
   ```
   ## Fix — Sprint {N}, Step {M}: {title}   (before the close | after the close)
   - Failure: {the user's report, in one line}
   - Cause: {file:line — what is wrong and why the step's tests missed it}
   - Tasks (ordered, each → one commit):
     1. regression test that reproduces the failure + the fix → `fix: …`
        (the test is seen red before the fix, committed green with it)
     2. this step's guide gains the failed item as an explicit check → `docs: …`
     (+ docs task if a doc describes the changed behaviour)
   - Files:
   - Branch: `{task branch}` | `fix/{feature}-sprint-{N}-step-{M}` ← `{base}`   (after the close: the base of the plan's `### Branch`, or `main` once the sprint branch is merged into it; `-2`, `-3`… if the name exists)
   - Touches shared code: no | yes — {consuming features}
   - Also affected by the cause: none | {other places the same cause reaches — fixed here only if the step promised them; otherwise listed for `/adhoc` or the sprint}
   - Risks:
   ```
   The regression test comes first whenever the failure is testable — always
   in the profile's Test-critical zones. A fix that needs more than ~3
   commits, a schema change, or shared code of another feature is said so
   explicitly: the user decides whether it is still a fix of this step or
   goes to the sprint. Present the mini-plan with `ExitPlanMode` and
   **wait for explicit approval** — the approval of the plan is the
   approval; ambiguous → ask, do not infer.

3. **Branch and record.** Before the close: check out the step's task
   branch. After the close: stash uncommitted work of other steps (e.g. a
   plan awaiting approval — it never goes into the fix's commits), update
   the base and create the fix branch from it. Append to the plan section a
   `### Verification fix — {date}: {what failed}` subsection with the cause
   and the approved tasks as checkboxes (never edit the original tasks; a
   later fix adds a new subsection below). Before the close, mark the
   section `(status: in progress — verification fix: {what failed})`; after
   the close its status stays `closed`.

4. **Execute — scope frozen to the approved tasks.** Every commit goes to
   the branch of step 3. One task → check command (`docs/TECH-STACK.md` →
   Check command) exit 0 → conventional commit (`fix:` / `docs:`). Docs
   that describe the changed code are updated in the same commit. Tick the
   subsection's checkboxes as tasks complete. Anything the diagnosis
   missed — a second cause, a needed dependency, a mismatch — stop, describe
   it and wait; a fix never grows silently. Record the user's decision in
   the subsection (`- Mid-step: {problem} → {decision}`): a change the user
   explicitly approves edits the subsection's tasks (each marked
   `(mid-step)`); work left out of the fix → continue with the approved
   tasks; a reply that approves nothing specific → ask again.

5. **Finish.** Run the check command once more (exit 0).
   - **Before the close:** mark the section
     `(status: implemented, awaiting verification)`, commit the plan file on
     the task branch, and end with the line below. The WORKLOG entry, the
     step's tick and the merge are `/close-step`'s job.

     > Fix for Step {M} committed on `{task branch}`. Verify again by `docs/features/{feature}/verification/sprint-{N}-step-{M}.md` — the failed item is now an explicit check. Every item holds → `/close-step`.

   - **After the close:** Read `.claude/commands/close-step.md`, step 2
     (**Docs self-check**), and apply it to this fix; add an entry at the
     TOP of `docs/WORKLOG.md` titled
     `## {date} — [{feature}] Sprint {N} Step {M} — fix: {what failed}`
     (cause, what changed, open questions); commit. Merge the fix branch
     into its base with `--no-ff` (`merge: fix sprint {N} step {M}`), run
     the check command on the base (exit 0), delete the fix branch, check
     out the branch that was checked out before and restore the stash. A
     conflict limited to entries added at the TOP of `docs/WORKLOG.md` is
     resolved by keeping both, newest on top; any other conflict →
     `git merge --abort`, report the conflicting files, STOP. The check
     command failing after the merge → report it and STOP, keeping the fix
     branch (the repair goes through `/adhoc` on the base). The next
     `/fix-step` resumes at the merge (**Interrupted fix**). End with the
     cause, the commits, the branch merged into, and:

     > Fix for Step {M} merged into `{base}`. Verify it by `docs/features/{feature}/verification/sprint-{N}-step-{M}.md` — the failed item is now an explicit check. Still failing → `/fix-step <what failed>` again.
