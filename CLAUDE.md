# CLAUDE.md

Guidance for Claude Code working in this repository.

## Project documentation

These files at the monorepo root are the project's source of truth. Read
the relevant ones before starting work; treat them as canonical when they
disagree with code comments or commit history.

- **`ROADMAP.md`** — phased plan of work. Each item is tagged
  `[done]` / `[partial]` / `[todo]`.
- **`ARCHITECTURE.md`** — high-level shape of the SPA + WordPress backend
  and how they communicate.
- **`PROJECT-TREE.md`** — directory layout for `frontend/` and
  `backend/wp-content/themes/pc/`.
- **`PUSHER-COIN-COMMANDS.txt`** — Home Assistant API for the physical
  machine.
- **`INVENTORY.md`** — frozen Phase 0 audit of the REST surface, player
  role, user meta keys, frontend stubs, and resolved drift.
- **`API-CONTRACT.md`** — canonical request/response/error shapes for
  the `pc/v1` namespace, current and planned. **Source of truth for any
  endpoint work.**
- **`DATA-MODEL.md`** — storage decisions per entity (table / CPT / user
  meta / WP option) and the decision matrix for new entities.
- **`ADMIN-DECISION.md`** — admin surfaces are a separate Vue SPA, not
  WP admin. Explains scope and trade-offs.

There is also `frontend/CLAUDE.md` covering frontend-only conventions
(Vue 3 + Vite + Pinia stack notes). Read it when working in `frontend/`.

## After making changes — documentation upkeep

Code changes outpace docs unless we keep them in lock-step. After every
non-trivial change, decide whether each doc above needs an update and do
it in the same change.

- Touched a REST controller, route, request shape, response shape, or
  error code? → Update **`API-CONTRACT.md`**. If it's a new endpoint,
  also move it from "planned" to "current". If it's a new error code,
  add it to the error-code registry.
- Added or removed a user meta key, custom table, CPT, or WP option? →
  Update **`DATA-MODEL.md`**, and (for meta keys) add a constant to
  `backend/wp-content/themes/pc/app/utils/user-meta-keys.php`.
- Changed the directory layout, added top-level files, or moved
  significant code? → Update **`PROJECT-TREE.md`**.
- Changed cross-cutting architecture (auth flow, deploy targets,
  trust boundaries, state stores)? → Update **`ARCHITECTURE.md`**.
- Reversed or refined an admin-surface decision? → Update
  **`ADMIN-DECISION.md`**.
- Resolved drift between frontend and backend, or surfaced new drift? →
  Update the relevant section of **`INVENTORY.md`**.

If a change does not affect any doc, that's fine — just confirm rather
than assume.

## After completing roadmap work — update ROADMAP.md

`ROADMAP.md` is not a one-time plan; it is the project status board.
When a phase, sub-item, or task gets done:

- Update the inline status tag (`[done]` / `[partial]` / `[todo]`) on
  the affected sub-item.
- If a whole phase is finished, mark the phase header so it's visually
  obvious (e.g. add `— DONE` after the title).
- Keep the **Tracking matrix** at the bottom of `ROADMAP.md` in sync
  with the body — if you change a sub-item's phase, update the matrix.
- Resolve or remove items from **Open questions to resolve** when they
  are answered.

Do this in the same change that completes the work — never leave the
roadmap stale "for the next pass."
