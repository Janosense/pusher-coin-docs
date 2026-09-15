---
description: Close a sprint — every step closed, Definition of Done ticked with evidence, SPRINT-N-CLOSE.md, merge into main, deploy by hand, retro agenda. Re-runnable until every item is settled.
argument-hint: [feature] [sprint]   # e.g. "2" or "event-calendar 2"; default = the sprint of the latest WORKLOG entry
---

The sprint boundary has one owner: this command. `/close-step` closes steps
and never touches the sprint; `/plan-step {N+1} 1` does not start until this
command has closed Sprint {N}. It is **re-runnable**: run it after the last
`/close-step`, again after the deploy, again until every Definition of Done
item of `SPRINT-{N}.md` is ticked or carries a carry note. Nothing here plans
or writes code.

**Which sprint.** `$ARGUMENTS` → optional feature name, sprint number N.
Resolve the feature through the Features table in root `CLAUDE.md` (with one
row it is implied; with several and no name → STOP and ask). No sprint number
→ the sprint of the latest `docs/WORKLOG.md` entry. Files:
`docs/features/{feature}/sprints/SPRINT-{N}.md`, `SPRINT-{N}-PLAN.md`,
`SPRINT-{N}-CLOSE.md` (this command writes it).

**Preconditions — refuse to proceed if any fails:**
- Every step of `SPRINT-{N}.md` is ticked (`### [x] Step …`) and every plan
  section of `SPRINT-{N}-PLAN.md` is `closed`. An unticked step → name it:
  `/plan-step` / `/do-step` / `/close-step` finish it first.
- No step of any feature is in flight (`in progress` or `implemented,
  awaiting close` in the plan file named by the latest WORKLOG entry).
- The working tree is clean.

## Phase 1 — close the sprint in the repo (first run)

Skip to Phase 2 if `SPRINT-{N}-CLOSE.md` already exists and Phase 1's commit
is on the sprint's base.

1. **Check command** (`docs/TECH-STACK.md` → Check command) exits 0 on the
   sprint branch (the **Branch** of `SPRINT-{N}.md`; with the simple git
   model, on `main`). Non-zero → stop and report; nothing is closed on red.
2. **Write `SPRINT-{N}-CLOSE.md`**
   (`docs/features/{feature}/sprints/SPRINT-{N}-CLOSE.md`) — the sprint's handoff to the retro, the
   re-planning chat and `/plan-step {N+1} 1`. It is a report, not a sprint
   file; write it from the files, the code and git, never from memory of a
   session:
   - **Definition of Done** — every item of `SPRINT-{N}.md` with its evidence
     (check command output, commit, verification guide, merge commit), or
     `open — sprint boundary` for an item only the deploy or the user can
     settle, or `open — {why}` for anything else still missing
   - **Built** — modules, actions, policies, components, schema (names and
     paths) that the next sprint inherits
   - **Not locally verifiable** — artefacts still pending their real run
     (from the plans' Checks and the step reports)
   - **Deferred** — every item the sprint's plans, reports or WORKLOG entries
     pushed to "the sprint boundary" or to a later sprint, and every
     Definition of Done item of Sprint {N-1} carried into this sprint
     (`— carried to Sprint {N}`) that is still open
   - **Contradictions** — cross-read `FEATURE.md` (Roadmap, UI),
     `SPRINT-{N}.md` → Out of scope, `SPRINT-{N+1}.md` if it exists,
     `docs/DESIGN.md` and `docs/DECISIONS.md`: list every item two files place
     differently (which files, which placements) — do not resolve them
   - **LEARNINGS** — entries of this sprint still marked `pending`
3. **Tick the Definition of Done in `SPRINT-{N}.md`** — every item whose
   evidence is in the repo now (all steps closed, the check command exit 0,
   docs match reality per the closes' self-checks, previous carried items
   settled) becomes `- [x] … — {evidence}`. Items marked
   `open — sprint boundary` stay `- [ ]` for Phase 2. Never tick an item
   without evidence; never delete or reword an item.
4. **Commit** on the sprint branch: `chore(sprint): close sprint {N}`.
5. **Merge into `main`** per the git model in root `CLAUDE.md`: the sprint
   branch → `main` with `--no-ff`, message `merge: sprint {N} — {name}`;
   then into the deployment branch(es) the git model names, if any. The
   check command exits 0 on `main` after the merge. A conflict is stopped
   and reported, never resolved silently. With the simple git model the
   work is already on `main` — say so. The sprint branch is kept (history).
6. **Stop before the deploy** — print:

   ```
   ## Sprint {N} — repo closed, boundary open
   - Definition of Done: {ticked}/{total} ticked; open: {items}
   - Merged: {sprint branch} → main ({merge commit}) [→ {deployment branches}]
   - Deploy: from `main`, by hand — {Environments & deploy of docs/ARCHITECTURE.md}; this run verifies: {Not locally verifiable items | none}
   - Retro agenda (10 min, over SPRINT-{N}-CLOSE.md): {pending LEARNINGS entries — each needs "Transferred to playbook": version / local / n/a}; Contradictions: {list | none}
   - Next: deploy, retro, then run `/close-sprint` again to settle the open items
   ```
   If nothing is open (no boundary item, no deploy), continue with Phase 3.

## Phase 2 — settle the boundary items (later runs)

For each still-unticked item of `SPRINT-{N}.md` → Definition of Done, in
this order:
- **Evidence the repo or git holds** (a merge, the check command on `main`,
  a file) → verify it and tick: `- [x] … — {merge commit / command}`.
- **Evidence only the user has** (a deploy done by hand, an outcome observed
  in an environment or by a client, a demo held) → list ALL such items in
  ONE message and ask whether each is done. "Done" → tick:
  `- [x] … — confirmed by user {date}`. "Not done" → stop: the boundary is
  not finished; say what is missing and end (run again later). Only if the
  user explicitly says to carry the item: rewrite it as
  `- [ ] … — carried to Sprint {N+1}: {reason}` — `/plan-step {N+1} 1` lists
  it under Checks → Docs vs reality and the next `/close-sprint` puts it
  into Deferred.
- **Retro results**, if the user names them in the reply: fill
  "Transferred to playbook" of the named `docs/LEARNINGS.md` entries with the
  value given (version / local / n/a) — nothing else in LEARNINGS changes.

Update the Definition of Done section of `SPRINT-{N}-CLOSE.md` to match,
commit on `main`: `chore(sprint): sprint {N} definition of done`.

## Phase 3 — closed

When every item is ticked or carried, add a WORKLOG entry at the TOP of
`docs/WORKLOG.md`: `## {date} — [{feature}] Sprint {N} closed` (3–4 lines:
merged, deployed, carried items), commit it on `main` with the ticks, then
check whether `docs/features/{feature}/sprints/SPRINT-{N+1}.md` exists and
end with exactly one of these lines:
- it exists → "Sprint {N} closed. Next: `/plan-step [feature] {N+1} 1`" —
  the next sprint starts from its file; re-planning is needed only if the
  retro changed its scope or SPRINT-{N}-CLOSE.md → Contradictions touches
  its scope (then: a re-planning chat in the Cowork Project first).
- it does not exist but `FEATURE.md` → Roadmap lists Sprint {N+1} →
  "Sprint {N} closed — SPRINT-{N+1}.md is missing: open a re-planning chat
  in the Cowork Project (DISCOVERY → Feature mode, Re-planning) — it reads
  SPRINT-{N}-CLOSE.md first — to write it; then
  `/plan-step [feature] {N+1} 1`".
- the Roadmap ends at Sprint {N} → "Sprint {N} closed — this was the
  feature's last sprint; the next feature starts with a discovery chat in
  the Cowork Project (Feature mode)".
Re-planning and discovery never run in Claude Code: you never write or
extend a sprint file. If the user asks you to "re-plan" or "plan Sprint
{N+1}", answer with the matching line above — the chat to open and the file
it will write — never with a bare "not my job".
