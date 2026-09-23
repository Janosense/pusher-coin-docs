---
description: Close a verified step — running it means the user verified the step by its guide. Docs self-check, worklog, merge; the last step of a sprint also merges the sprint into main.
disable-model-invocation: true
---

**Running `/close-step` means verified.** The user followed the step's
verification guide (written by `/do-step`) on the task branch, and every
item holds. Never ask whether the step was verified.

**Which step.** The step is the one implemented or fixed in this session —
feature, N and M come from its plan section. If this session did neither:
the checked-out branch is a task branch named in a plan section's
`### Branch` → that step. Otherwise take the feature from the latest
`docs/WORKLOG.md` entry, open its
`docs/features/{feature}/sprints/SPRINT-{N}-PLAN.md` files (every N) and
look for sections in state `implemented, awaiting verification` — exactly
one: that is the step; several: STOP and ask which step to close; none:
check **Resume** below, and if it does not apply either, STOP and ask.
Never pick a step yourself. A step whose section is already `closed` is
never closed again — only **Resume** applies to it.

**Resume.** A previous `/close-step` stopped at a merge (a conflict, or the
check command failing after it). Its step is already `closed` and
committed: do not repeat steps 1–4 and do not write a second WORKLOG entry.
- A `closed` section whose task branch (its `### Branch`) still exists →
  continue from step 5, then steps 6–7 as usual.
- Every step of `SPRINT-{N}.md` is ticked, the git model has chained sprint
  branches, and `git branch --merged main` does not list the sprint branch
  → do the merge of step 7, then the report of step 6 with the lines of
  step 7.

1. **Completeness check.** The plan section is
   `implemented, awaiting verification`, all its checkboxes — those of its
   `### Verification fix` subsections too — are ticked, and the check
   command (`docs/TECH-STACK.md` → Check command) exits 0 on the task
   branch. If not — list what's missing and stop. The verification guide
   `docs/features/{feature}/verification/sprint-{N}-step-{M}.md` missing →
   write it (Read `.claude/commands/do-step.md`, step 8), commit it on the
   task branch and STOP: the user verifies by it, then runs `/close-step`
   again.

2. **Docs self-check.** Go through this checklist and update files where the
   answer is "yes" (in the same closing commit):
   - schema / table / field / index changed → `docs/DATA-MODEL.md` (and the feature's `FEATURE.md` data section)
   - new module, endpoint, flow, background job, integration → `docs/ARCHITECTURE.md`; feature-internal structure → its `FEATURE.md`
   - a decision was made, changed, or a fixed decision was challenged → new entry in `docs/DECISIONS.md` (a rule for code → the `docs/TECH-STACK.md` item below)
   - dependency or tool added → `docs/TECH-STACK.md`
   - the step set a rule for code that later steps must follow and would
     otherwise write differently → one line in `docs/TECH-STACK.md`:
     ANTI-PATTERNS when the stack's default leads the other way
     (`Do not X — Y`), otherwise CONVENTIONS; a rule of one code area only
     → that area's `CLAUDE.md` → Area conventions. The line is the record:
     it needs a DECISIONS entry only when alternatives were weighed. A line
     this step made untrue is edited or removed. A one-time choice, a
     detail of this step or screen, a data-model or domain rule is not such
     a rule — it has its own item in this list
   - new domain term or rule appeared or changed → `docs/DOMAIN.md`
   - endpoint / event / payload shape changed → `docs/CONTRACTS.md` (if the project keeps one)
   - design token or shared UI component added or changed, or a screen built → `docs/DESIGN.md` (if the project keeps one: Tokens / Components "In code" / Screens) and the feature's `FEATURE.md` UI section
   - a playbook rule was missing, ambiguous, contradicted another, or was
     broken in this step → entry in `docs/LEARNINGS.md` per its header. A
     playbook rule says which command, file, branch or approval does what
     and when (a section of `.claude/commands/`, a Core rule or Step protocol
     line of `CLAUDE.md`, a template); how to write code or tests is not one

3. **Worklog.** Add a new entry at the TOP of `docs/WORKLOG.md` (3–6 lines;
   existing entries are never edited): date, [feature] sprint/step, what
   changed, key decisions, open questions.
   If the step touched shared code, say so explicitly — other features read this.
   If the section has `### Verification fix` subsections: one line — which
   guide item failed and the cause.
   If this is the last step of the sprint, the entry ends with the
   line `Sprint {N} complete — all steps closed, {sprint branch} merged into
   main`.

4. **Status and commit.** Tick the step's checkbox in
   `docs/features/{feature}/sprints/SPRINT-{N}.md` (`### [x] Step {M} — …`),
   mark the plan section `(status: closed)`. Then commit everything from
   steps 2–4 on the task branch: `chore(step): close sprint {N} step {M}`.
   Nothing else is written to `SPRINT-{N}.md` — the tick is the only thing
   this command changes there.

5. **Merge.** Merge the task branch into its base (both named in the plan's
   `### Branch`) with `--no-ff`, message `merge: sprint {N} step {M} — {title}`;
   run the check command on the base (exit 0), then delete the task branch.
   A conflict limited to entries added at the TOP of `docs/WORKLOG.md` is
   resolved by keeping both, newest on top; any other conflict is never
   resolved silently: run `git merge --abort`, report the conflicting files
   and STOP. The check command failing after the
   merge: report it and STOP, keeping the task branch (the fix goes through
   `/adhoc` on the base). Either way the next `/close-step` resumes here
   (**Resume**).

6. **Final report** in the chat (plain language if root `CLAUDE.md` →
   Project profile says `user-verified`):

   ```
   ## Step {M} closed — {title}
   - What was done:
   - Files changed:
   - Commits:  (+ merged into {base})
   - Deviations from plan:  (incl. the section's `### Mid-step decisions` and `### Verification fix` subsections)
   - Verification guide: `docs/features/{feature}/verification/sprint-{N}-step-{M}.md` — verified by the user
   - Not locally verifiable: n/a | {artefact} — pending {the run named in the plan, e.g. the next deploy from `main`}
   - Docs updated: [list]
   - Design: n/a | unchanged | changed — `docs/DESIGN.md` (Tokens / Components "In code" / Screens) and `FEATURE.md` → UI updated in this close
   - Open questions:
   - Next: /plan-step [feature] {N} {M+1}   (feature name required when several exist)
     — or, if this was the last step of the sprint, the lines of **Last step of the sprint** below
   ```

7. **Last step of the sprint.** When every step of `SPRINT-{N}.md` is now
   ticked (this one included), the sprint is complete and you say so — the
   user must not have to check the sprint file to learn it.
   - **Merge the sprint into `main`.** Per the git model in root
     `CLAUDE.md`: with chained sprint branches, merge the sprint branch (the
     base of step 5) into `main` with `--no-ff`, message
     `merge: sprint {N} — {sprint name}`; the sprint branch stays. The check
     command exits 0 on `main` after the merge; if it fails, report it and
     STOP (the fix goes through `/adhoc` on `main`). A conflict is handled
     as in step 5 (WORKLOG entries only → keep both; otherwise
     `git merge --abort`, report the conflicting files and STOP — the next
     `/close-step` resumes here, **Resume**). With
     the simple model (task branch → `main`) step 5 already landed on
     `main` — nothing more to merge.
   - **Say it.** The report's Next line — and the last sentence of your
     reply — starts with exactly:

     > Sprint {N}: this was the last step — every step of `SPRINT-{N}.md` is closed. Sprint merged into `main` ({merge commit} | nothing to merge — simple git model).

     followed by what comes next, derived from the files: if
     `docs/features/{feature}/sprints/SPRINT-{N+1}.md` exists →
     `Next: /plan-step [feature] {N+1} 1`; if it does not but `FEATURE.md` →
     Roadmap lists Sprint {N+1} → the file is missing: name the re-planning
     chat in the Cowork Project (DISCOVERY → Feature mode, Re-planning) that
     writes it; if the Roadmap ends with Sprint {N} → the feature is done:
     the next feature comes through discovery. You never write or extend a
     sprint file yourself.
