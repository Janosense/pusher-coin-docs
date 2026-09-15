# Learnings — Pusher Coin

<!-- Harness defect log. Every time the agent did the wrong thing (overreach,
     misread instruction, ignored a rule, broken assumption) — it goes here,
     framed as a PROCESS defect, not a one-off annoyance. At sprint boundaries
     the user reviews this file and transfers fixes into the playbook repo
     (new playbook version). This is the improvement loop. At the retro every
     entry's "Transferred to playbook" is filled: a version, "local" (fixed in
     this project only) or "n/a" — "pending" survives only until the next
     retro. -->

Entry format:

## {{YYYY-MM-DD}} — [{{feature}}] {{What happened, one line}}
- **Incident:** {{what the agent did vs. what was expected}}
- **Root cause:** {{which instruction was missing, ambiguous, or overridable}}
- **Fix applied here:** {{change to this project's CLAUDE.md/commands/docs}}
- **Transferred to playbook:** {{version, or "pending"}}

---

## 2026-09-15 — [core] `ddev start` silently rewrote a tracked config file
- **Incident:** During the wallet-rollback ad-hoc, `ddev start` regenerated `backend/wp-config-ddev.php` and dropped the hand-added `JWT_AUTH_SECRET_KEY` and `GOOGLE_CLIENT_ID` defines. It showed up only as an unexpected `M` in `git status`; committing it would have broken local JWT login, and missing it would have left local login broken.
- **Root cause:** The file still carries DDEV's `#ddev-generated` header, so DDEV treats it as its own and overwrites it on start, while the project keeps hand-written constants in it. The root `CLAUDE.md` lists `ddev start` as a routine command with no warning.
- **Fix applied here:** The file was restored with `git checkout -- wp-config-ddev.php` and not committed. Check `git status` in `backend/` after every `ddev start` and restore that file if it changed. The structural fix — moving those constants out of the DDEV-owned file — is review item 6 in `docs/BACKEND-REVIEW.md`.
- **Transferred to playbook:** n/a — project-specific
