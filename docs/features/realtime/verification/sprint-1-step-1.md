# Verification — `realtime` Sprint 1, Step 1: Delta-audit

**What this step was for.** It was a reading step. The agent went through the existing
code that the `realtime` feature will build on, and wrote down what it found in one
document: `docs/features/realtime/FEATURE.md`, section **Fit into the host**. For each
existing file the feature will lean on, that section gives the exact place in the code,
one line on why it matters, and which later step will change it.

**Nothing a player or an operator can see has changed.** No code was changed in any of
the three apps (`backend/`, `frontend/`, `admin/`), and nothing is deployed or pushed.

**Before you start.** Open Terminal and run each block below exactly as written, one
block at a time. `cd` means "go to this folder". Everything happens on your own machine.

```bash
cd ~/Projects/full-stack/pusher-coin
```

---

## 1. The branches are where they should be

```bash
echo "docs:     $(git rev-parse --abbrev-ref HEAD)"
for d in backend frontend admin; do
  echo "$d: $(git -C $d rev-parse --abbrev-ref HEAD)"
done
```

**Expected:**

```
docs:     realtime/sprint-1
backend: main
frontend: main
admin: main
```

The step created the sprint branch in the docs folder only, because only a document
changed. The three apps stay on `main` until a step changes code in them.

**Must NOT be there — the old abandoned branches.** You deleted them before the step
ran; this confirms none came back:

```bash
for d in backend frontend admin; do
  echo "$d: [$(git -C $d branch --list 'realtime/*' | tr -d ' ')]"
done
```

**Expected:** three lines, each ending in empty brackets `[]`.

---

## 2. Only documentation changed, and `main` was not touched

```bash
git diff --stat main realtime/sprint-1
```

**Expected:** only files under `docs/` — `FEATURE.md`, `SPRINT-1.md`, `SPRINT-1-PLAN.md`,
`WORKLOG.md`, `LEARNINGS.md` and this guide. No file from `backend/`, `frontend/` or
`admin/` appears (those are separate repositories, and they were not changed).

```bash
git log -1 --format='%h %s' main
```

**Expected:** `1cfa370 docs(realtime): update Sprint 2 plan and goal — align details
with fixed decisions`. That is your own commit from before the step. The step added
nothing to `main`.

---

## 3. Read the audit — this is the main check

Open `docs/features/realtime/FEATURE.md` in your editor and scroll to **Fit into the host**
(near the top, after "Purpose & scope").

**Expected:**

- **Shared code it depends on** — a list split into *Backend* (13 files) and *Player SPA*
  (7 files). Every entry starts with a real file path in `backticks`, has at least one
  place in the code (`:` followed by a line number), says in a few words **why it matters
  here**, and names the step that will change it. The short codes are explained in
  the heading: "S1.4" means Sprint 1 Step 4, and "S2.5" means Sprint 2 Step 5.
- **Conflicts with the siblings' invariants** — three bullets: two about `core`, one
  about `stripe` (which says "none").
- **Entry point** — says the new feature plugs in with one line in `functions.php`, next
  to the `stripe` line.
- **Check coverage**, **`admin/`** and **`BACKEND-REVIEW.md` items** — three more
  bullets. The last one says, for review items §10, §12 and §15, which step deals with
  each, and marks two of §10's bullets **unassigned**.

**Must NOT be there:** an entry that names only a class or an area without a file path,
e.g. "`Rate_Limiter`" on its own. That was the old wording, and the step's job was to
replace it.

---

## 4. Spot-check three claims against the code

The audit says where things are. These commands look at those exact places, so you can
confirm the document matches the code without reading any code yourself.

**a) "A last declared coin closes the turn on the spot"** (the `queue-service.php` entry):

```bash
git -C backend grep -n "Declaration spent" -- wp-content/themes/pc/app/utils/queue-service.php
```

**Expected:** one line, starting with `…queue-service.php:238:`, containing
`Declaration spent — hand the machine to the next player.`

**b) "Seeds no `pc_machine_*` option"** (the `install-schema.php` entry):

```bash
git -C backend grep -n "add_option( 'pc_machine_" -- wp-content/themes/pc/app/utils/install-schema.php; echo "matches found: $?"
```

**Expected:** only the line `matches found: 1`. `1` here means "nothing found", which is
what the audit claims. The file does contain the words `pc_machine_events`, but that is a
table name, not a setting, so the command searches only for settings (`add_option`).

**c) "The admin Machine screen names `sensor.relay_on`"** (the `admin/` bullet):

```bash
git -C admin grep -n "sensor.relay_on" -- src/views/MachineView.vue
```

**Expected:** one line starting with `src/views/MachineView.vue:168:`.

---

## 5. Both check commands pass on the code the sprint starts from

The backend check needs the local database running. Start it first:

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev start
git status --porcelain
```

**Expected:** `git status --porcelain` prints nothing. If it prints
`M wp-config-ddev.php`, run `git checkout -- wp-config-ddev.php` and carry on. That is
the known DDEV quirk in `docs/LEARNINGS.md`, not a fault of this step.

```bash
bin/check; echo "exit code: $?"
```

**Expected:** the output ends with

```
== Summary
php -l:      49 files OK
DDEV checks: passed — stripe-client.php stripe-webhook.php wallet-rollback.php
exit code: 0
```

**Must NOT happen:** a box saying `SKIPPED: DDEV checks did not run`. If you see it, the
money checks did not run. Make sure `ddev start` finished, then run `bin/check` again.

```bash
cd ~/Projects/full-stack/pusher-coin/frontend
bin/check; echo "exit code: $?"
```

**Expected:** the output ends with `lint:  OK`, `build: OK` and `exit code: 0`. A
warning about chunks larger than 500 kB may appear above it. It was there before this
step and is not a failure.

---

## 6. What the audit asks you to decide (nothing to check, something to know)

The audit found problems that no step of this feature's three sprints covers. They are
written down, not fixed. Each needs a decision from you at some point: add a step to a
sprint (a re-planning chat), or handle it separately through `/adhoc`.

1. **A late payout can go to the wrong player.** When a player tosses their last declared
   coin, their turn ends immediately. If that toss makes the machine pay out a moment
   later, the coins go to the *next* player in the queue, or to nobody if the queue is
   empty. This goes against this sprint's goal. Step 2's measurement of how long a
   payout takes to arrive will show how often it would happen.
2. **Two problems from the backend review have no step:** a failed refund of a coin
   (the player loses it with no record), and the 2-second machine timeout (a slow but
   successful toss gets refunded, so it is free).
3. **The balance on the Room screen doesn't move on a machine payout.** Only the
   winnings counter does. The balance updates after the next toss or a page reload.
   Sprint 1 Step 5's manual check says to watch the balance, so its plan must say which
   of the two you watch.
4. **Smaller, `/adhoc` candidates:** the admin **Machine** screen's sensor labels may
   become wrong after Step 2. `docs/TECH-STACK.md` says machine settings have
   install-time defaults that the code does not actually seed.

If any check in sections 1–5 does not match, report it with `/fix-step <what you saw>`.
