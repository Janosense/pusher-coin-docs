# SPRINT {{N}} — {{Name}} ({{date range}})

<!-- playbook: v1.21. Written by discovery (Phase C) together with every
     other sprint of the plan — never by Claude Code. Rewritten by a
     re-planning chat only while no step is closed; afterwards steps may
     only be appended. This file
     describes the sprint and nothing else: goal, fixed decisions, steps.
     The only thing written here during the sprint is the tick in a step's
     heading, by /close-step. -->
**Branch:** {{per the git model in CLAUDE.md}}
**Goal:** {{one paragraph: what is demonstrably true when the sprint is done —
phrased as user/admin-visible outcomes, not as a task list}}

## Fixed decisions
{{Decisions already made that must NOT be reopened during the sprint, with links
to docs/DECISIONS.md entries. If an implementation finding challenges one of
these — stop and raise it, don't silently deviate.}}

## Steps
<!-- Ordered by dependency. One step = one /plan-step → /do-step → /close-step
     cycle and must fit one working session. Every step has all five
     subsections, even if a subsection is "—". The checkbox in the heading is
     ticked by /close-step. -->

### [ ] Step 1 — {{name}}
- **Tasks:**
  - {{…}}
- **Tests:** {{what must be covered; reference test-critical zones}}
- **Verification (manual):** {{what the user opens/runs/clicks and what must happen; a screen step names the screen as in FEATURE.md → UI}}
- **Docs to update:** {{which docs this step is expected to touch}}
- **Depends on:** —

### [ ] Step M — {{name}}
- **Tasks:**
- **Tests:**
- **Verification (manual):**
- **Docs to update:**
- **Depends on:** {{Step X, Step Y — every step that must be closed before this one; or "—"}}
