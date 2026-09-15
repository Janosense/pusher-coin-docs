# Feature — {{FEATURE_NAME}}

<!-- Lightweight ARCHITECTURE + DATA-MODEL for one feature. Lives at
     docs/features/{{name}}/FEATURE.md next to its sprints/. Root
     ARCHITECTURE.md holds only one row + a link here. Keep ≤80 lines. -->

## Purpose & scope
{{2–4 sentences: what this feature does for whom. Explicitly: what is OUT of
scope for this feature.}}

## Fit into the host
- **Code location:** `{{path from the Features table}}`
- **Host area:** {{area name}} — obeys its `CLAUDE.md` isolation rules
- **Entry point:** {{bootstrap file — the single registration point in shared
  code: an import, route registration, or hook line}}
- **Shared code it depends on:** {{helpers/components it reads — a change there
  affects this feature; or "—"}}

## Data
{{Entities, tables, config/state keys, caches this feature owns (in WP
terms: CPTs, taxonomies, options, transients). Schema details → root
docs/DATA-MODEL.md; here list what belongs to this feature and the ownership
rule: no other feature writes to this data.}}

## Invariants
{{Feature-specific non-negotiables, e.g. API rate limits and token handling,
caching rules, "public pages never trigger external API calls synchronously".}}

## Interfaces
{{What this feature exposes to the rest of the system (API endpoints,
events, UI components/embeds; in WP terms: shortcodes, blocks, hooks) — the
surface other features may rely on. Changing it = "touches shared surface"
in a plan.}}

## UI
<!-- Delete this section if the feature has no screens. Screen names are the
     ones sprint steps use in Verification (manual); shared tokens and
     components are indexed in docs/DESIGN.md, not repeated here. -->
- **Screens:** {{name → `design/{{file}}` + the screen's name inside that export (path under docs/features/{{name}}/design/; one export per Claude Design project, so several screens share a file), one per line}}
- **Reuses:** {{components from docs/DESIGN.md this feature uses}}
- **Introduces:** {{new tokens or shared components this feature adds — each one is a docs/DESIGN.md update; or "—"}}

## Roadmap
<!-- One line per sprint: goal as a user-visible outcome + link to its file.
     EVERY sprint listed here has a full sprints/SPRINT-N.md, all written by
     discovery in one pass; a line without a file is a discovery defect. A
     not-yet-started sprint is rewritten by a re-planning chat only if its
     scope changed. -->
- Sprint 1 — {{goal}} (`sprints/SPRINT-1.md`)
- Sprint 2 — {{goal}} (`sprints/SPRINT-2.md`)
- Sprint N — {{goal}} (`sprints/SPRINT-N.md`)
