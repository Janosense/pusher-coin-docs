---
description: Execute the plan of the current step and write its verification guide. Running this command is the approval of the written plan. Scope frozen.
disable-model-invocation: true
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
  "as recommended": before executing, write that resolution into each
  question in the plan file (`Resolved: approved as recommended — {answer}`).
  A question the user answered differently in this session is resolved with
  the user's words.
  A question without a recommendation and without an answer makes the plan
  not runnable: print the question and STOP without changing any file. The
  user's answer that follows is a change request — handle it as in the next
  bullet.
- The user's last message before `/do-step` was not a pending change request
  to this plan. If the user wrote objections or answers that were never
  folded into the plan file (the file still shows the old text), do not
  execute. Fold them in: edit this step's section of `SPRINT-{N}-PLAN.md`
  only (an answer goes under its question as `Resolved: {answer}`, and the
  tasks it affects change with it; the status stays `awaiting approval`),
  print the plan again and STOP with exactly this line — the user re-runs
  `/do-step` against the revised plan:

  > Plan for **Step {M}** written. `/do-step` runs it as written — running it is the approval, and it authorizes Step {M} only. Anything else you write is a change request to the plan.

**Not this command.** A report that an item of a step's verification guide
failed — before or after the step's close, in the chat or with
`/fix-step <what failed>` — is never patched or re-run here: Read
`.claude/commands/fix-step.md` and follow it, the report being its
argument. A section in state `in progress — verification fix: …` belongs to
`/fix-step` too.

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
   silently expand or "fix along the way". The user's reply decides; before
   going on, record it in a `### Mid-step decisions` subsection at the end of
   the plan section (`- {problem} → {decision}`):
   - the work stays out of this step (another step, the next sprint,
     `/adhoc`) → continue with the approved tasks;
   - the user explicitly approves a change to this step's tasks → edit the
     tasks in the plan section, mark each added or changed task
     `(mid-step)`, and continue; a new dependency needs its own explicit
     "yes" (root `CLAUDE.md` → Core rules);
   - the reply approves nothing specific → ask again; never infer.

   A Claude Design handoff received in this session is input for the
   approved tasks only — it never adds tasks or screens; design files in
   `docs/features/{feature}/design/` are references, never copied into the
   code area.
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
   especially in the Test-critical zones of root `CLAUDE.md` → Project
   profile.
5. **Docs that describe the changed code are part of the task.** The entries
   of the plan's "Docs to update" that document what a task changes
   (`DATA-MODEL.md` for a schema change, `CONTRACTS.md` for an endpoint or
   payload shape, `ARCHITECTURE.md` for a new module/endpoint/flow,
   `TECH-STACK.md` for an approved dependency, `DESIGN.md` for a new or
   changed token or shared UI component) are updated in the same commit as
   the change.
6. **Tick the checkboxes** in `SPRINT-{N}-PLAN.md` as tasks complete.
7. Do **not** write the WORKLOG entry, tick the step in `SPRINT-{N}.md`, or
   merge the branch — that is `/close-step`'s job, after the user has
   verified the step.

When all tasks are done:

8. **Verification guide.** Write
   `docs/features/{feature}/verification/sprint-{N}-step-{M}.md`: a numbered
   manual guide derived from the step's Verification (manual) and what the
   tasks built — what to open/run, what to click, what must happen,
   including at least one negative check (what must NOT be possible), when
   relevant. The user follows it on the task branch, before any merge: its
   first item says how to start what it checks from the checked-out task
   branch. If root `CLAUDE.md` → Project profile says `user-verified`:
   assume the reader never reads code; include exact URLs, commands, and
   expected screen states. An artefact under the plan's Not locally
   verifiable gets no item — the guide names the run that verifies it.
9. **Hand over.** With the guide written, run the check command once more
   (exit 0), mark the plan section
   `(status: implemented, awaiting verification)` and commit the guide with
   the plan file on the task branch:
   `docs(step): verification guide for sprint {N} step {M}`. Finish with:

> Step {M} implemented on `{task branch}`. Verify it on this branch by `docs/features/{feature}/verification/sprint-{N}-step-{M}.md`: every item holds → `/close-step` (running it means verified); an item fails → describe what you did, expected and got (or `/fix-step <what failed>`) — the fix lands on this branch.
