---
description: Close the current step — report, docs self-check, verification guide, worklog, merge; the last step of a sprint also merges the sprint into main.
---

**Which step.** The step is the one implemented in this session — feature, N
and M come from its plan section. If this session implemented nothing: take
the feature from the latest `docs/WORKLOG.md` entry, open
`docs/features/{feature}/sprints/SPRINT-{N}-PLAN.md` and look for sections in
state `implemented, awaiting close` — exactly one: that is the step; none or
several: STOP and ask which step to close. Never pick a step yourself.

1. **Completeness check.** The plan section is `implemented, awaiting close`
   (on a re-close: `implemented, awaiting close — reopened: …`), all its
   checkboxes — on a re-close, those of the latest `### Reopen` subsection
   too — are ticked, and the check command (`docs/TECH-STACK.md`
   → Check command) exits 0 on the task branch. If not — list what's missing
   and stop.

2. **Docs self-check.** Go through this checklist and update files where the
   answer is "yes" (in the same closing commit):
   - schema / table / field / index changed → `docs/DATA-MODEL.md` (and the feature's `FEATURE.md` data section)
   - new module, endpoint, flow, background job, integration → `docs/ARCHITECTURE.md`; feature-internal structure → its `FEATURE.md`
   - a decision was made, changed, or a fixed decision was challenged → new entry in `docs/DECISIONS.md`
   - dependency or tool added → `docs/TECH-STACK.md`
   - new domain term or rule appeared or changed → `docs/DOMAIN.md`
   - endpoint / event / payload shape changed → `docs/CONTRACTS.md` (if the project keeps one)
   - design token or shared UI component added or changed, or a screen built → `docs/DESIGN.md` (if the project keeps one: Tokens / Components "In code" / Screens) and the feature's `FEATURE.md` UI section
   - something went wrong with the process itself this step (agent overreach,
     misread instruction, broken assumption) → entry in `docs/LEARNINGS.md`;
     on a re-close, why the step's tests did not catch the failure is such an
     entry when the cause is the process (a test-critical zone left untested,
     a guide that did not cover the step's promise), not a one-off slip

3. **Verification guide.** Write
   `docs/features/{feature}/verification/sprint-{N}-step-{M}.md`: a numbered
   manual guide — what to open/run, what to click, what must happen,
   including at least one negative check (what must NOT be possible), when
   relevant. If profile = user-verified: assume the reader never reads code;
   include exact URLs, commands, and expected screen states. On a re-close
   the guide is rewritten in place and the item that failed becomes an
   explicit check.

4. **Worklog.** Add a new entry at the TOP of `docs/WORKLOG.md` (3–6 lines;
   existing entries are never edited): date, [feature] sprint/step, what
   changed, key decisions, open questions.
   If the step touched shared code, say so explicitly — other features read this.
   On a re-close (the step was reopened by `/fix-step` after a failed
   verification — its section reads `implemented, awaiting close — reopened:
   …`): a NEW short entry titled `… — re-closed`, stating what failed and
   what was fixed (from the section's `### Reopen` subsection).
   If this is the last step of the sprint (step 8), the entry ends with the
   line `Sprint {N} complete — all steps closed, {sprint branch} merged into
   main`.

5. **Status and commit.** Tick the step's checkbox in
   `docs/features/{feature}/sprints/SPRINT-{N}.md` (`### [x] Step {M} — …`),
   mark the plan section `(status: closed)`. Then commit everything from
   steps 2–5 on the task branch: `chore(step): close sprint {N} step {M}`
   (`re-close` on a re-close). Nothing else is written to `SPRINT-{N}.md` —
   the tick is the only thing this command changes there.

6. **Merge.** Merge the task branch into its base (both named in the plan's
   `### Branch`) with `--no-ff`, message `merge: sprint {N} step {M} — {title}`,
   then delete the task branch. The check command exits 0 on the base after
   the merge; a conflict is stopped and reported, never resolved silently. On a
   re-close, the `-reopen` branch created by `/fix-step` is merged the same
   way and deleted.

7. **Final report** in the chat (plain language if profile = user-verified):

   ```
   ## Step {M} closed — {title}
   - What was done:
   - Files changed:
   - Commits:  (+ merged into {base})
   - Deviations from plan:
   - Verification guide: `docs/features/{feature}/verification/sprint-{N}-step-{M}.md`
   - Not locally verifiable: n/a | {artefact} — pending {the run named in the plan, e.g. the next deploy from `main`}
   - Docs updated: [list]
   - Design: n/a | unchanged | changed — `docs/DESIGN.md` (Tokens / Components "In code" / Screens) and `FEATURE.md` → UI updated in this close
   - Open questions:
   - Next: /plan-step [feature] {N} {M+1}   (feature name required when several exist)
     — or, if this was the last step of the sprint, the lines of step 8
   ```

8. **Last step of the sprint.** When every step of `SPRINT-{N}.md` is now
   ticked (this one included), the sprint is complete and you say so — the
   user must not have to check the sprint file to learn it.
   - **Merge the sprint into `main`.** Per the git model in root
     `CLAUDE.md`: with chained sprint branches, merge the sprint branch (the
     base of step 6) into `main` with `--no-ff`, message
     `merge: sprint {N} — {sprint name}`; the sprint branch stays. The check
     command exits 0 on `main` after the merge; a conflict is stopped and
     reported, never resolved silently. With the simple model (task branch
     → `main`) step 6 already landed on `main` — nothing more to merge.
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
