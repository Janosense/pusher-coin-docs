# {{CODE_AREA_NAME}} — code-area conventions

<!-- Code-area CLAUDE.md: ONE per code directory (an app, package, or
     service; in WP — a theme or plugin), even when
     several features live inside it. Delta only, ≤60 lines — everything
     shared (core rules, step protocol, git model) stays in the ROOT CLAUDE.md.
     Claude Code loads this file automatically when working inside this
     directory. Feature docs do NOT live here — they are in
     docs/features/{feature}/ (see the Features table in root CLAUDE.md). -->

{{1–2 sentences: what this area is (e.g. "the API service — hosts shared
middleware plus feature modules under src/features/", or "the {{theme-name}}
theme — site-level templates plus feature modules under inc/features/").}}

## Feature isolation (if this area hosts several features)
- Each feature lives in `{{src/features/{name}/ or inc/features/{name}/}}` with its own bootstrap file.
- `{{the area entry file, e.g. app entry or functions.php}}` contains exactly one registration line per feature — nothing else feature-specific.
- Shared code ({{helpers, base styles, common components}}) is changed ONLY as
  an explicit plan task marked **"touches shared code — may affect other
  features"**; the plan must list which features consume it.
- A feature never writes to data owned by another feature (see each FEATURE.md → Data).

## Area conventions
{{Naming, module layout, asset/build pipeline for this area, routing or
template rules, escaping/i18n conventions — whatever is specific to this
area.}}

## Local commands
```bash
{{only if they differ from root commands, e.g. this area's asset build}}
```
