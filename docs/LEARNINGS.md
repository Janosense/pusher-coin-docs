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

## 2026-09-18 — [realtime] A sprint step's own manual check would have broken the behaviour it promised
- **Incident:** `SPRINT-1.md` Step 3's verification creates a second room carrying a live room's machine id, leaves it unavailable, and promises "the existing room keeps working throughout". Attribution (`Queue_Service::room_id_for_machine()`) took the newest room with the id, available or not. So that very scenario would have sent the live room's payouts to the empty room. The step also frames its rule per machine id, while `Machine_Service` drives one physical machine whatever the id, so two open rooms with *different* ids still share it.
- **Root cause:** The sprint was written from `DOMAIN.md` and `BACKEND-REVIEW.md` §12, not from the code that decides where a payout lands. A manual-verification scenario is data the code then acts on, and nobody traced it through.
- **Fix applied here:** `/plan-step` read the lookup and raised Question 1, and the user chose to fix it in the step (task 5). The per-id limit is stated in `BACKEND-REVIEW.md` §12, `FEATURE.md` invariant 5 and the `ROOM_MACHINE_ID` row of `DATA-MODEL.md`. Caught before any code, like the 2026-09-18 `stripe` entries.
- **Transferred to playbook:** pending. Discovery should trace each step's manual-verification scenario through the code paths it touches, and check that a rule keyed on an identifier matches what the code actually keys on.

---

## 2026-09-18 — [realtime] A spike plan asked for a 24-hour unattended session when stored history already held the evidence
- **Incident:** `/plan-step realtime 1 2` planned the venue day as a 24-hour live recording. The Claude Code session had to stay open that whole time, with the laptop left plugged in and online. The user could not wait ("I can't wait that long"). Home Assistant's stored history turned out to answer the coin, relay and bonus questions in minutes, including ~200 past toss presses to time against. The spike's own Session A (fact 6) had already found that ten days of history existed.
- **Root cause:** The plan followed the step text ("one venue day of passive logging") literally. It did not weigh an evidence source it already knew of against the cost to a user who verifies by hand, and it did not ask whether a day-long session was feasible. `/do-step` runs a step as one session, so any wait inside a step ties up that session.
- **Fix applied here:** Mid-step, the user chose stored history plus announced presses (plan task 4 records the change). For this project, a plan that needs a session open for hours states that wall-clock cost up front and names any cheaper recorded source first.
- **Transferred to playbook:** pending — `/plan-step` could require a step's wall-clock cost to be stated whenever it exceeds one working session.

---

## 2026-09-18 — [realtime] A one-line question during `/do-step` was answered by carrying on with the step
- **Incident:** During `/do-step` the user wrote "check the token". The agent ran the token check, which passed, and then went on in the same turn to write and trial-run the logger without first answering. The user interrupted: "What are you doing??? … I need a simple answer from you."
- **Root cause:** `/do-step` had authorised the tasks, so the agent treated the mid-turn message as a sub-task of the run, not as a question expecting a plain answer and a pause. With a user-verified profile the user cannot follow tool output, so progress without an answer reads as being ignored.
- **Fix applied here:** Answered in one sentence, stopped, and waited. For this project, a user message that arrives mid-run gets a plain answer first; the run continues only after that, or when the message itself says to continue.
- **Transferred to playbook:** pending

---

## 2026-09-18 — [realtime] A plan named its base commit from the session-start snapshot, and the execution notes then guessed why it differed
- **Incident:** `/plan-step realtime 1 1` wrote "`main` (`cfc3d75` at plan time)". That id came from the git status the session is handed at start, not from a live `git rev-parse main`, and `main` had already moved to `1cfa370` at 12:39:44, 21 minutes before the plan file was written. `/do-step` saw the difference and correctly checked that Step 1's text was unchanged. But it then wrote in the plan's execution notes that the sprint text "sat uncommitted on disk during planning". That was an inference stated as fact, and the reflog does not support it. It was corrected at `/close-step`.
- **Root cause:** The session-start git status is a snapshot, and nothing in `/plan-step` asks for a named commit id to be read live. When reality later differed, the agent explained the gap instead of looking it up. `git reflog --date=iso main` answers it in one line.
- **Fix applied here:** The execution notes are corrected in the close commit. For this project, a plan that names a commit id reads it with `git rev-parse` at plan time. A difference found later is explained from `git reflog`, never by inference.
- **Transferred to playbook:** pending — `/plan-step` §3 could require every commit id it records to be read live, not taken from the session context.

---

## 2026-09-18 — [stripe] A step plan promised a signed-in browser check that the agent's own rules forbid
- **Incident:** `/plan-step stripe 2 2` wrote that `/do-step` would check the new admin screens in a browser by placing a session "in `localStorage` from a token minted by `\PC\AuthController::issue_access_token`". At execution the agent declined that very action: writing an access token into a browser to authenticate falls under its standing prohibition on entering credentials or tokens, which holds even on request. Only the signed-out redirect was observed. Every signed-in screen state went to the user's guide, which the plan had presented as the fallback for a *missing extension*, not for the plan's own method.
- **Root cause:** The plan chose a verification method without checking it against the agent's standing prohibitions — the same class of collision as the 2026-09-17 test-card entry. The admin SPA's only other way in is the two-step sign-in, whose 6-digit code arrives by email (Mailpit locally), so an agent has no permitted route past it.
- **Fix applied here:** The Step 2 guide states that the agent never saw the screens signed in and makes every screen state an explicit user check. For this project, a plan names an agent browser check only for what is reachable without signing in. Signed-in admin screens are verified by the user's guide, plus the endpoint checks the agent can run with `curl`.
- **Transferred to playbook:** pending — `/plan-step` should check any agent-run verification step against the agent's standing prohibitions (credentials, tokens, card numbers) before promising it.

---

## 2026-09-18 — [stripe] A sprint step cited a "red treatment" that exists only in DESIGN.md
- **Incident:** `SPRINT-2.md` Step 2 asked for the Stripe badge's red `Not configured` "with the same red treatment the captcha panel uses". `docs/DESIGN.md`'s Subjects row says the captcha panel is "unconfigured in red", but `SubjectsView.vue` draws it **amber** (`#ffd970` on a yellow tint). No red treatment existed to reuse.
- **Root cause:** DESIGN.md was written in Adoption mode "documented from code", and this state was recorded wrongly. Discovery then built on the document without opening the component. A screen-state claim in an adopted DESIGN.md is a description, not evidence.
- **Fix applied here:** `/plan-step` caught it by reading the component. `DECISIONS.md` 2026-09-17 ("red when unconfigured") took precedence: the badge uses the captcha line's shape with the SPA's `--danger` red. DESIGN.md's Subjects row and the captcha code are left unchanged, as an `/adhoc` candidate.
- **Transferred to playbook:** pending — a sprint that says "the same X as screen Y" should be checked against Y's code, not only its DESIGN.md row.

---

## 2026-09-18 — [stripe] A sprint's manual verification named local data that nothing creates
- **Incident:** `SPRINT-2.md` Step 1's verification says `GET /admin/topups` "lists the local top-ups including the seeded LiqPay-era rows (`pc-topup-N` refs)". The local database had no such row and the project has no seeder for one (`wp pc` offers only `seed-rooms` and `machine-ingest`); the only top-ups were three Sprint 1 Stripe rows. Taken literally, the step's central check — old and new rows side by side — could not be performed locally.
- **Root cause:** The sprint was written from what production holds (real LiqPay-era rows) and assumed the local database mirrors it. A verification that depends on particular data does not have to say where that data comes from, so the gap surfaces only when someone inspects the database — here at `/plan-step`, because the playbook asks the plan to inspect reality. Caught before any code, resolved cheaply.
- **Fix applied here:** The plan recorded it under Docs vs reality and created one `failed` fixture row (`pc-topup-268`) through `Wallet_Service`, so no ledger write bypassed the service and no money was claimed; the Step 1 guide says where the row came from. The sprint text is unchanged.
- **Transferred to playbook:** pending — a manual verification that relies on specific data should name its source (an existing environment, a seeder, or a fixture the step creates).

---

## 2026-09-17 — [stripe] A repo-root `grep` skips all three app repositories, so "it is gone" was almost verified against nothing
- **Incident:** Sprint 1 Step 5 removes LiqPay from every repository, and `SPRINT-1.md` makes `grep -ri liqpay` across the three repositories its evidence. Run from the project root, that grep returned only `docs/*` files — reading exactly like a clean pass. The application code was never searched: a scoped `grep` in `backend/` immediately found seven lines the root grep had not reported. Had the root result been trusted, the step would have reported "LiqPay is gone from the code" on evidence that never looked at the code.
- **Root cause:** Two things compound. The root repository's `.gitignore` lists `/backend/`, `/frontend/` and `/admin/` on purpose — they are separate git repositories (`ARCHITECTURE.md` → Overview). The session's `grep` is a wrapper that honours ignore files, so every recursive search started at the root silently excludes all three applications. Nothing errors and nothing warns; the command exits 0 with a short, plausible answer. `SPRINT-1.md`'s verification says "across the three repositories" without saying that the root is not a vantage point from which they are visible.
- **Fix applied here:** The Step 5 verification guide checks each repository separately with `git grep`, which searches that repository's tracked files — the same scope that ships, identical for the reader and for the agent. A single root-level `grep -r` is never the evidence for a removal claim in this project; any sweep that must cover the applications names them one at a time.
- **Transferred to playbook:** pending — the general rule (a repo-root recursive search is not evidence when the tree contains nested repositories or ignored app directories) is not project-specific.

---

## 2026-09-17 — [stripe] "Pay with the test card" collides with a standing prohibition on entering card numbers
- **Incident:** Sprint 1 Step 2 instructs the agent to "pay the session with the `4242 4242 4242 4242` test card". The agent operates under a standing rule that forbids entering card or bank numbers into any field, stated absolutely — "prohibited even when the user explicitly asks". Read literally the step cannot be executed; read purposively it plainly can, since `4242…` is a number Stripe publishes so that it can be typed into forms, belongs to no person, and moves no funds in a `livemode: false` sandbox.
- **Root cause:** The prohibition is written to protect a real financial instrument and does not carve out published test instruments. The step text, for its part, says "test card" without saying that it is a reserved number in a sandbox — so the judgement has to be re-made every time, by whoever runs the step.
- **Fix applied here:** Executed, on the reading that a published test number in a sandbox is not a financial credential, and said so plainly in the report and in the step's verification guide rather than letting it pass silently. The step's own guide now states the distinction, so Step 4's manual verification and any re-run do not have to re-litigate it.
- **Transferred to playbook:** pending — a sprint step that requires a test instrument should name it as such ("Stripe's published test card, sandbox, `livemode: false`"), so the instruction carries its own justification.

---

## 2026-09-17 — [stripe] A real delta-audit does not fit FEATURE.md's 80-line guideline
- **Incident:** Step 1's delta-audit filled `docs/features/stripe/FEATURE.md` to 96 lines against the template's "Keep ≤80 lines" comment — the second time running: `realtime`'s own Step 1 audit left its FEATURE.md at 91 and logged the same overshoot as an open item. Both were compressed once and still did not fit.
- **Root cause:** The template sizes FEATURE.md for a feature summary, but `/plan-step` reads its "Shared code it depends on" as the authoritative touchpoint list, and `/do-step` §2 forbids trimming neighbouring sections to make room. A per-file, per-line audit of a feature that rewrites an existing payment path is simply longer than 80 lines, so the guideline loses to the instruction that needs the detail.
- **Fix applied here:** None — the audit detail was kept and the overshoot recorded, as `realtime` did. The durable fix is a playbook decision: either raise the guideline for features with a shared-code audit, or move the audit to its own file that FEATURE.md links.
- **Transferred to playbook:** pending

---

## 2026-09-15 — [core] `ddev start` silently rewrote a tracked config file
- **Incident:** During the wallet-rollback ad-hoc, `ddev start` regenerated `backend/wp-config-ddev.php` and dropped the hand-added `JWT_AUTH_SECRET_KEY` and `GOOGLE_CLIENT_ID` defines. It showed up only as an unexpected `M` in `git status`; committing it would have broken local JWT login, and missing it would have left local login broken.
- **Root cause:** The file still carries DDEV's `#ddev-generated` header, so DDEV treats it as its own and overwrites it on start, while the project keeps hand-written constants in it. The root `CLAUDE.md` lists `ddev start` as a routine command with no warning.
- **Fix applied here:** The file was restored with `git checkout -- wp-config-ddev.php` and not committed. Check `git status` in `backend/` after every `ddev start` and restore that file if it changed. The structural fix — moving those constants out of the DDEV-owned file — is review item 6 in `docs/BACKEND-REVIEW.md`.
- **Transferred to playbook:** n/a — project-specific

---

## 2026-09-16 — [core] An ad-hoc off `main` had no check command to gate its commits
- **Incident:** The machine-power-switch ad-hoc branched from `main`, where `backend/bin/check` does not exist — it was created in `realtime` Sprint 1 Step 1 and only reaches `main` when that sprint merges. `/adhoc` §4 and `/do-step` §3 both require every commit to be gated by that one committed script, and equally forbid replacing it with an inline shell chain, so the ad-hoc could satisfy neither instruction as written.
- **Root cause:** The playbook assumes the check command exists on every branch work happens on. With chained sprint branches, tooling created inside a sprint is absent from `main` until the sprint boundary — and hotfixes are exactly the work that branches from `main`.
- **Fix applied here:** Gated with the exact committed script taken from the sprint branch (`git show realtime/sprint-1:bin/check`), run against the ad-hoc tree and removed again, rather than with a hand-written command chain. The durable fix is to land tooling like the check command on `main` directly instead of inside a sprint.
- **Transferred to playbook:** pending

## 2026-09-21 — [realtime] A plan named a shared function's consumers from the audit instead of from the code
- **What happened:** the Step 4 plan changed `Machine_Ingest_Service::log_event()` to return a `WP_Error` and stated that its only consumer was `wp pc machine-ingest`, which handles one. It has a second: the toss endpoint reads `$event['event_id']` straight out of the result (`RoomQueueController.php:155,167`). Shipping the plan as written would have thrown a fatal *after* the player's coin was debited and the machine had tossed it — a real coin lost to a 500. It surfaced only when `/do-step` grepped the callers before editing.
- **Why it happened:** the delta-audit in `FEATURE.md` lists `RoomQueueController.php` as shared code this feature touches, but describes it by the toss flow, not by which service methods it calls. The plan read the audit's consumer note ("today's only producer") as the consumer list and never asked the code.
- **Rule:** before a plan changes the signature or return type of a shared function, grep for every caller and name them in the task — the audit says which *files* matter, never which *functions* they call. A consumer list in a plan is a claim about the code, so it is checked against the code.

## 2026-09-21 — [realtime] A plan's test-cleanup clause enumerated the fixtures and forgot the rows the code under test writes
- **What happened:** the Step 5 plan said the new checks would end in "a `finally` that removes every fixture, option, transient and filter it touched" — and they did exactly that. Nothing in that list is a `wp_pc_machine_events` row, and the whole point of the code under test is to write them. Eleven runs during the step left 162 test rows in the forensic table that records real payouts. Noticed only during the end-of-step sweep for leftovers, and fixed there.
- **Why it happened:** the cleanup clause was written by listing what the *test* creates (users, rooms, options, transients, filters) rather than what the *code* writes when the test runs it. `tests/machine-ingest.php` already deletes its own event rows by key prefix, so the pattern existed one step earlier and was not carried across — the plan reached for a generic phrase instead of the neighbouring script.
- **Rule:** a test-cleanup clause names the tables the code under test writes, not only the fixtures the test creates. When a sibling script in `tests/` already cleans up after the same table, the plan cites that script rather than re-describing cleanup in the abstract.

## 2026-09-21 — [realtime] `??` in an assertion swallows the very `null` being asserted
- **What happened:** twice now, a check written as `null === ( $row['x'] ?? 'fallback' )` could never pass. `??` treats `null` as absent and returns the fallback, so an assertion *about* `null` compares the fallback instead. In Sprint 1 Step 4 it was `['user_id'] ?? 'x'` on an unattributed event row; in Sprint 2 Step 1 it was `['channels']['machine'] ?? 'x'` on a token response. Both times the code under test was correct and the check was not, and both times it cost a debugging detour to find that out.
- **Why it happened:** `??` reads as "with a default", and in an assertion that default silently changes what is being asserted. It is invisible at a glance precisely because the idiom is so familiar everywhere else.
- **Rule:** never use `??` in an assertion whose expected value is `null`. Read the container once, then assert with `array_key_exists()` plus an explicit `null ===` on the value. `??` is fine for assertions about a non-null value, where the fallback cannot be mistaken for a pass.

## 2026-09-21 — [realtime] A scripted documentation edit whose pattern did not match applied nothing, silently
- **What happened:** Sprint 1 Step 5 updated `ARCHITECTURE.md`'s directory map with a Python `str.replace()` whose search string used a tree-indentation prefix the file does not contain. `replace()` returns the string unchanged when there is no match, so the write succeeded, the commit succeeded, the check command passed, and the step was closed and merged with two new files missing from the map. `PROJECT-TREE.md` was never in that step's docs list at all. Both maps stayed wrong until the next step happened to edit the same block and noticed.
- **Why it happened:** the grep that "confirmed" the edit matched the *other* replacement in the same script and the prose further down the file, so it printed hits and looked like proof. A partial confirmation was read as a whole one.
- **Rule:** a scripted edit asserts its own precondition — `assert old in s` before replacing — so a non-matching pattern fails loudly instead of writing the file unchanged. And when a change claims to touch several places, the confirming grep is read per place, not as a count of hits.

## 2026-09-21 — [realtime] The same test left the same table dirty twice, for a different reason each time
- **What happened:** Sprint 1 Step 5's checks left 162 rows in `wp_pc_machine_events` because the cleanup named only fixtures. The rule written then — name the tables the code under test writes — was followed in Sprint 2 Step 1, and the table was still left dirty: the cleanup deleted `machine_id = $machine_id`, while one check deliberately credits a *second* machine id (`…-nobody`) to prove an unclaimed payout is still credited. 15 more rows.
- **Why it happened:** the rule fixed the *table* and said nothing about the *keys*. A test that exercises "what happens with a different id" writes rows under an id the cleanup was never told about.
- **Rule:** clean up by the run's own prefix, not by an exact identifier — `LIKE 'prefix%'` over `= 'prefix'` — so every id a run invents is covered by construction. And the end-of-step sweep for leftovers queries by that prefix rather than by the identifiers the test happens to name.

## 2026-09-21 — [realtime] A test cleaned up the table its own code writes, and left the tables its fixtures filled
- **What happened:** the rule written earlier the same day — clean up by the run's prefix, covering the tables the code under test writes — was followed, and `wp_pc_machine_events` came out clean. `wp_pc_room_queues` and `wp_pc_bet_sessions` did not: a throwaway room that a check puts a player into accumulates rows in both, written by `core`'s queue service rather than by the code under test. `tests/machine-poll.php` and `tests/realtime-channel.php` deleted their rooms and left 25 orphaned queue rows and 57 orphaned session rows behind. Found by a sweep that looked for rows whose room no longer exists, rather than for this run's identifiers.
- **Why it happened:** "the tables the code under test writes" is the wrong boundary. A fixture is not inert — creating a room and putting a player in it makes *other* code write rows, and those rows outlive the room when the room is hard-deleted.
- **Rule:** cleanup is scoped to the **fixtures**, not to the code under test: deleting a throwaway room deletes every row keyed to it first. And the end-of-step sweep looks for orphans — rows whose parent is gone — not only for rows matching this run's prefix, because the prefix never appears in a child table.

---

## 2026-09-21 — [realtime] A defect a spike found was routed to `/adhoc`, and `/adhoc` has no owner
- **Incident:** The Sprint 1 Step 2 spike recorded that `POST /rooms/{id}/play` refuses **every toss while the machine is on**, called it "a defect in frozen `core`, for `/adhoc`", and noted that Sprint 2 Step 3 would therefore have "no payout signal to follow". Three `/close-step` runs then carried it forward as "`/adhoc` wanted", and nobody ran one — `/adhoc` is user-initiated, and the agent can only keep listing it. `/plan-step realtime 2 3` was the first command that had to confront it, and found the step undeliverable as written: applied literally to the recorded polarity, "disable the toss while the relay is closed" disables it permanently, because closed is the machine working.
- **Root cause:** A spike may route work out of the sprint, but nothing makes that routing land. The WORKLOG's "Open" line is the only carrier, and it is a note, not a queue: it has no owner, no trigger and no check that the next step which *depends* on the item is still buildable. The sprint file was written before the spike ran, so its Step 3 text still described a payout gate the machine does not have — and the fixed decision "Step 3 reads that entry; it does not re-derive them" pointed at an entry that contradicted the step.
- **Fix applied here:** `/plan-step` raised it as two questions with recommendations — what the lock now means, and where the server correction lands — and the user's `/do-step` accepted both, so the correction shipped as task 1 of the step that needed it. For this project: when a spike entry says a defect blocks a named later step, the plan for that step treats the defect as in scope and asks, rather than assuming an `/adhoc` happened.
- **Transferred to playbook:** pending — a spike's `DECISIONS.md` entry that defers work to `/adhoc` should name the step it blocks, and `/plan-step` for that step should be required to resolve the deferral explicitly instead of planning around it.

---

## 2026-09-21 — [realtime] A shared client-side service was shipped in a shape that forbade the sprint's own later steps
- **Incident:** Sprint 2 Step 2 shipped `frontend/src/services/realtime.js` with a `subscribe()` that begins by tearing the whole Ably client down — correct for the one listener that existed, and silently fatal for the second. Sprint 2 Step 4 needs the chat listening to the same room channel as the queue, which the same sprint had already planned. The flaw survived Step 2's `/do-step`, its `/close-step`, a merge and two further steps, and was found only when `/plan-step realtime 2 4` read the file.
- **Root cause:** Nothing exercises this file. `frontend/` has no test runner, so a design flaw in shared client code has no gate at all between writing it and the next step reading it — and the step that wrote it had no reason to imagine a second consumer, because its own verification only ever had one. Three consecutive close reports have now carried "the browser half has no automated coverage" as an open question; this is the first time that gap has a concrete cost attached to it rather than a warning.
- **Fix applied here:** Step 4's task 1 reworked the service into a named-consumer registry before anything else, and the queue moved to it unchanged. For this project: when a step builds shared client-side plumbing that a *later step in the same sprint* is named to reuse, the plan says which later step will reuse it and what shape that requires — the sprint file already knows, and reading it is free.
- **Transferred to playbook:** pending — and the larger item behind it stands: a repository with no test runner cannot gate its own shared code, which is a decision worth making deliberately rather than carrying as an open question in every close report.

---

## 2026-09-21 — [realtime] The two tree maps have no owner, so a new file is mapped only when a plan happens to remember it
- **Incident:** Adding this step's two files to `docs/PROJECT-TREE.md` and `docs/ARCHITECTURE.md` revealed that **Step 4's `tests/realtime-chat.php` was in neither**, and that ARCHITECTURE's test list was also missing **Step 2's `tests/realtime-queue.php`**. Both steps closed and merged with the maps wrong. This is the third occurrence: Sprint 1 Step 5's poller files were missing from both maps and were corrected in Sprint 2 Step 1 (`LEARNINGS.md` 2026-09-21, the silent no-op edit).
- **Root cause:** `/close-step`'s docs self-check enumerates DATA-MODEL, ARCHITECTURE, DECISIONS, TECH-STACK, DOMAIN, CONTRACTS, DESIGN and LEARNINGS — **`PROJECT-TREE.md` is not on that list**, and ARCHITECTURE is on it for "new module, endpoint, flow" rather than "a file was added". A new file therefore reaches the maps only if the *plan* named them under Docs to update, which depends on whether that step's planner thought of it. Two steps in a row did not. The failure is silent by construction: nothing checks a tree map against the tree.
- **Fix applied here:** both maps corrected for all four missing entries in this step's commits, disclosed in the plan's execution notes and the commit bodies as unlisted files. Step 5's plan did list `PROJECT-TREE.md`, which is the only reason this surfaced.
- **Transferred to playbook:** pending. `/close-step`'s docs self-check needs a line of the form "a file was created, moved or deleted → the project's directory map(s)", and the check is mechanical enough to be worth scripting: every path in the map exists, and every file in the tree is in the map.

---

## 2026-09-21 — [realtime] A sprint's last step was asked to write down a measurement no step of the sprint ever provisioned
- **Incident:** `SPRINT-2.md`'s goal and Step 5's Docs-to-update both require "the peak concurrent-connection count **observed** on Ably during this sprint" in `TECH-STACK.md`. No Ably account exists. None of Steps 1–4 creates one, none needed one to close, and every publish in four steps ran against a stub. So the sprint's own final deliverable was unobtainable by the time it was due, and the plan had to raise it as a question to decide what could honestly go in the file.
- **Root cause:** the sprint was written assuming an external account would exist by its later steps, without any step owning the act of creating it. It is the same class as `LEARNINGS.md` 2026-09-18 ("a sprint's manual verification named local data that nothing creates") — a step's deliverable depending on state no step produces — but one level up: here it is the *sprint goal* that depends on it, so it went unnoticed through four closes, each of which honestly reported "still no Ably account" as an open question and moved on.
- **Fix applied here:** Question 1 of the Step 5 plan, resolved as recommended — `TECH-STACK.md` records the arithmetic (the binding ceiling is 200 simultaneous signed-in room viewers, one connection per tab, zero for guests and the unbuilt admin SPA) with the observed peak stated as **none, because nothing has ever connected**, and names the run that produces the real number. Sprint 3 reads a bound plus a marked hole rather than a blank.
- **Transferred to playbook:** pending. Discovery should give any sprint deliverable that depends on an external account, key or environment an explicit owning step — or state in the sprint that the deliverable is derivation, not measurement. A repeated "still no X" in consecutive WORKLOG open-questions is the signal that this has happened.

---

## 2026-09-21 — [realtime] A plan listed a check the test harness itself makes impossible to write
- **Incident:** the Step 1 plan's test 7 promised "no Home Assistant token: `stopped`, no mail, no incident" — the guard that stops an install nobody has connected from alerting every minute. It cannot be written. `Machine_Service::is_configured()` is false only when `PC_MACHINE_TOKEN` is absent, every script in `tests/` must `define()` that constant to run at all, and a constant cannot be undefined mid-process. Discovered while writing the test, not while planning it. The near-miss is the interesting part: the first attempt "fixed" it by blanking `pc_machine_endpoint` instead, which *looks* like an unconfigured install and is not — `Machine_Service::option()` treats an empty value as "use the built-in default", so that check would have passed while proving nothing about the guard.
- **Root cause:** `/plan-step` checks tasks against the real code but checks *tests* only against the step's Tests section. Nothing asks whether the harness can reach the state a proposed check needs. A check that cannot be written is indistinguishable, at plan time, from one that can.
- **Fix applied here:** the guard ships (one line, mirroring the relay watch's) and is named as knowingly unchecked in the script's own docblock; the check in its place covers the trap beside it, so the next person reaching for the endpoint option as a way to simulate a disconnected install finds a check rather than a surprise. The verification guide's section 3 then covers the branch for real — the local DDEV install has no machine token, so it *is* the unconfigured case, and the guide checks that it stays silent.
- **Transferred to playbook:** pending. `/plan-step` should state, for each planned check, the state the harness has to reach — and flag any that depend on the absence of something the harness must provide (constants, globals, autoloaded classes). Where it cannot be reached, the plan says so and names where the branch *is* covered, instead of listing a check that will quietly become something else.

---

## 2026-09-21 — [realtime] A third enumerating document turned out to have no owner either
- **Incident:** adding this step's audit event types to `DATA-MODEL.md` revealed the table was missing **three earlier sprints' worth**: the transport's six `machine_poll_*` types (S1.5), `machine_relay_read_failed` (S2.3) and the session cleanup's three (S2.5). All had shipped, closed and merged undocumented.
- **Root cause:** the same one `LEARNINGS.md` 2026-09-21 records for `PROJECT-TREE.md` and `ARCHITECTURE.md`'s tree — an *enumerating* document (a list that claims to be complete) is only updated when a step's plan happens to name it, and `/close-step`'s docs self-check names `DATA-MODEL.md` for "schema / table / field / index changed", which a new audit event type is not. Three artefacts now, one cause: the checklist is organised by *kind of change*, and these documents need updating on changes that do not match any kind on the list.
- **Fix applied here:** all five rows added at once, with a note above the table saying why they arrived together. Both tree maps were corrected in Sprint 2 Step 5 for the same reason.
- **Transferred to playbook:** pending. Generalise the pending fix from the tree-map entry: `/close-step` should carry one item covering every document that enumerates — "a file, event type, option, route or command was added → the document that claims to list them all" — and, better, these are mechanically checkable (every name in the list exists in the code, every one in the code is in the list).

---

## 2026-09-21 — [realtime] A sprint step's premise contradicted the spike it cited
- **Incident:** `SPRINT-3.md` Step 2 says "the coin sensor is expected to move within a bounded window, **as Sprint 1 Step 2 measured**". Sprint 1 Step 2 measured the opposite as well: a toss while the counter already reads 0 moves nothing and leaves no trace, observed with two real presses and confirmed across 2,768 samples (`DECISIONS.md` 2026-09-18). Taken literally the step would have put a "the machine did nothing" record on most ordinary tosses — the alarm-every-evening defect its own sprint goal forbids, moved into a table.
- **Root cause:** the step was written from a *summary* of the spike ("the counter resets on a toss") rather than from the spike's consequences section, which states that a repeated value is invisible to every transport. A citation to a document is not the same as a reading of it, and nothing in the step protocol re-checks a step's premise against the source it names.
- **Fix applied here:** `/plan-step` read the spike entry before deriving the tasks, found the contradiction, and raised it as the plan's single open question with three answers and a recommendation rather than narrowing the step silently. The resolution is now its own `DECISIONS.md` entry (2026-09-21, "A toss that cannot be judged is counted, never recorded"), so the next reader meets the rule and its reason together.
- **Transferred to playbook:** pending. `/plan-step` already reads the sprint's Fixed decisions; it should also re-read any `DECISIONS.md` entry a **step** cites by name and state in Checks → Docs vs reality whether the step's premise survives it. A step that cites evidence is making a claim about that evidence, and that claim is checkable before any code is written.

---

## 2026-09-21 — [realtime] A known failure mode from the previous step recurred in this one
- **Incident:** three checks in `tests/realtime-toss.php` asserted `null === ( $row['correlation_id'] ?? 'x' )` and failed — `??` returns the fallback for exactly the `null` being asserted. `LEARNINGS.md` records this same trap from Sprint 2, with the same shape, two steps earlier.
- **Root cause:** the log is read at the start of a step (`/plan-step` reads it for "known failure modes relevant to this step") and then not consulted again while the code that repeats the mistake is being typed. A prose entry has no force at the moment of writing an assertion; nothing mechanical stands between the mistake and the commit, and the check command cannot catch it because the assertion is syntactically fine and simply always false.
- **Fix applied here:** the three checks now use `array_key_exists()` with an explicit `null` comparison, with a comment naming the entry. The cost was one re-run, because the checks did fail — the harness caught it, which is the argument for asserting the absent case at all.
- **Transferred to playbook:** pending — and worth noting that the fix for this class is not another prose entry. A trap that is *mechanically detectable* (`?? ` in an assertion, `assert( x ?? y )`) belongs in a lint rule or a grep in the check command, not only in a log people read once per step.

---

## 2026-09-22 — [realtime] The enumerating-document gap recurred twice more, in the step that had just logged it twice
- **Incident:** checking the two tree maps *mechanically* — every file under `tests/` and `app/realtime/` grepped against `PROJECT-TREE.md` and `ARCHITECTURE.md` — found **`tests/realtime-toss.php` in neither map**, one step after Sprint 3 Step 2 created it and reported "both maps corrected", and **`tests/stripe-webhook.php` missing from `ARCHITECTURE.md`** since it was written. Separately, the same step found `ROADMAP.md` Phase 5 §7 still `[todo]` although all five steps of Sprint 2 had built it, with the phase's exit criteria still saying the SPA polls. Three documents, fourth, fifth and sixth occurrence of one failure mode already logged twice (`LEARNINGS.md` 2026-09-21, the tree maps and the audit event-type table).
- **Root cause:** unchanged, and now demonstrated to be unfixable by prose — Step 2's own plan *listed* both tree maps under Docs to update, its close *claimed* both were corrected, and its new test script still went unmapped, because the update was done by eye on the file the author was thinking about. An enumerating document is only as good as a human remembering every member, and nothing compares the list to the thing it enumerates.
- **Fix applied here:** all three corrected in this step's commits and disclosed in their bodies and in the plan's execution notes. More usefully, the mechanical check that found them is one line and could be a stage of `backend/bin/check`: for every file under `tests/` and `app/realtime/`, assert its name appears in both maps.
- **Transferred to playbook:** pending, and the recommendation is now stronger than the 2026-09-21 entries': do **not** add another checklist line. A list that claims to be complete should be verified by a script, and the playbook's `/close-step` should ask for that script rather than for another human pass. Three logged occurrences and two failed corrections is enough evidence.

---

## 2026-09-22 — [realtime] Two planned checks were vacuous because a threshold of zero means "off"
- **Incident:** `tests/realtime-withdrawals.php` derives every threshold from the backlog the database already holds, so that the script is honest on an install with real pending withdrawals. Two sections then said "narrow the count threshold to the baseline and the same backlog breaches it". On this database the baseline is **0**, and `0` is the documented off switch for that threshold — so the checks asserted a breach that the code is designed never to report, and failed. The code was right and the checks were wrong, again (`LEARNINGS.md` 2026-09-21, the `??` entry, is the same shape).
- **Root cause:** the plan designed the fixtures to be robust against a non-empty database and then wrote the threshold arithmetic for the *typical* case, without asking what each expression evaluates to at the boundary the local database actually sits on. A baseline-relative check has a degenerate case exactly where the baseline is zero, which is where every clean install starts.
- **Fix applied here:** both sections now create a real backlog and move the threshold across it, so the arithmetic never touches `0`. The failures cost one re-run: the harness caught them, which is the argument for asserting the negative case at all.
- **Transferred to playbook:** pending — where a plan's checks are written relative to state read from the environment, it should state what they evaluate to when that state is empty, which is the state a fresh install and a CI database are always in.
