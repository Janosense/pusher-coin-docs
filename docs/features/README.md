# Features

One folder per feature, resolved via the Features table in root CLAUDE.md:

    docs/features/{feature}/
    |-- FEATURE.md          # feature mini-architecture (+ UI section if the feature has screens)
    |-- design/             # design files: Claude Design exports, .dc.html artboards, HTML mockups
    |-- sprints/            # SPRINT-N.md + SPRINT-N-PLAN.md
    `-- verification/       # sprint-N-step-M.md, written by /close-step

Sprints and verification guides never live anywhere else. The verification
folder is the feature's growing manual regression suite. Files in design/
are design artifacts, never application code: nothing there is copied into
a code area; the shared tokens and components they use are indexed in
docs/DESIGN.md.
