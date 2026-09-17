# Verification — `stripe` Sprint 1, Step 1: Delta-audit and the check command

**What this step was for.** Two things, neither of which changes how the product
behaves. Nothing a player or an operator can see has changed; nothing is deployed.

1. The Stripe branch got the project's **check command** — the one script that
   inspects the code and refuses a commit when something is wrong. It already existed
   on the `realtime` branch; this step copied it across, unchanged.
2. A **delta-audit**: reading the real code to confirm (or correct) the list of
   existing files the Stripe work will have to touch.

**Before you start.** Open Terminal and run each block below exactly as written.
`cd` means "go to this folder". Everything happens on your own machine.

```bash
cd ~/Projects/full-stack/pusher-coin
```

---

## 1. You are on the right branch, and `main` was not touched

```bash
for d in . backend frontend admin; do
  echo "$d -> $(git -C $d rev-parse --abbrev-ref HEAD)"
done
```

**Expected:** all four lines end in `stripe/sprint-1`.

> If any line says `main`, stop and say so — nothing should have landed on `main`.

---

## 2. Start the machine database, then run the backend check

The backend check needs the local database running. Start it first:

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev start
```

DDEV has a known habit of overwriting a config file when it starts. Check:

```bash
git status --porcelain
```

**Expected:** no output at all. If it prints `M wp-config-ddev.php`, run
`git checkout -- wp-config-ddev.php` and carry on — that is the known DDEV quirk
recorded in `docs/LEARNINGS.md`, not a fault of this step.

Now the check itself:

```bash
bin/check
```

**Expected — this is the most important check in this guide.** The output ends with:

```
== Summary
php -l:      45 files OK
DDEV checks: passed — wallet-rollback.php
```

and higher up, a long list of `PASS` lines finishing with:

```
Success: All 53 checks passed.
```

Those 53 are the **money checks** — they prove the wallet cannot be left half-updated
if the database fails mid-payment. The point of this step is that they now run
automatically.

> **It must NOT say this:**
> ```
> SKIPPED: DDEV checks did not run
> ```
> A big boxed `SKIPPED` message means the money checks did **not** run. If you see it,
> DDEV is not up — go back and run `ddev start`, then try again.

---

## 3. The check actually catches a mistake (the negative check)

A check that passes everything is worthless. Let's break something on purpose and
confirm it complains. This makes a temporary mess and then undoes it.

```bash
cd ~/Projects/full-stack/pusher-coin/backend
echo "this is deliberately broken" >> wp-content/themes/pc/functions.php
bin/check
```

**Expected:** it stops quickly and ends with a failure naming that file, something like:

```
CHECK FAILED: syntax error in wp-content/themes/pc/functions.php
```

Now undo the damage and confirm the file is clean again:

```bash
git checkout -- wp-content/themes/pc/functions.php
git status --porcelain
```

**Expected:** no output (the file is back to normal). Run `bin/check` once more if you
want to see it go green again.

---

## 4. The player app's check runs, and stops rewriting your files

```bash
cd ~/Projects/full-stack/pusher-coin/frontend
bin/check
```

**Expected:** ends with

```
== Summary
lint:  OK
build: OK
```

Some yellow warnings about "chunks larger than 500 kB" and a Google sign-in import
appear during the build. Those are pre-existing and expected — not this step's doing.

Now the behaviour change that matters:

```bash
git status --porcelain
```

**Expected:** no output. Previously, `npm run lint` silently *edited* your files while
pretending to only inspect them. Now it reports and leaves the code alone. If you
actually want it to fix things, that is a separate command: `npm run lint:fix`.

---

## 5. The operator app still builds

`admin/` has no check script of its own yet — that is expected and written down.

```bash
cd ~/Projects/full-stack/pusher-coin/admin
npm run lint && npm run build
```

**Expected:** the lint prints nothing (no complaints), the build ends with
`✓ built in …`. Then:

```bash
git status --porcelain
```

**Expected:** no output.

---

## 6. The copied scripts really are identical to the originals

This is the step's own promise: the files were copied, not rewritten. Git compares
them for you.

```bash
cd ~/Projects/full-stack/pusher-coin
git -C backend  diff realtime/sprint-1 -- bin/check
git -C frontend diff realtime/sprint-1 -- bin/check
```

**Expected:** **no output whatsoever** from either command. Any output means the two
branches have drifted, which this step promised would not happen.

---

## 7. Read the audit — it should point at real code

Open this file:

```
docs/features/stripe/FEATURE.md
```

Find the section **Fit into the host → Shared code it depends on**. Every entry now
names a file and a line number, plus which upcoming step changes it.

**What to look for:** it should read as *findings*, not predictions. In particular it
should contain these five corrections, which the audit turned up by reading the code:

1. `services/walletService.js:22-31` — **was missing** from the earlier list.
2. The admin router and `AdminLayout.vue` carry **no** LiqPay reference — Sprint 2 only.
3. `wp-config-sample.php` / `wp-config-ddev.php` define **no** LiqPay constant, so
   Step 5 *adds* the Stripe ones rather than replacing anything.
4. `PaymentController.php` is the LiqPay callback **and nothing else**, so the whole
   file goes — along with two loader lines nobody had listed.
5. Two more files name the LiqPay secret: `CAPTCHA_SETUP.md:46` and
   `captcha-verifier.php:20`.

> **If the section says things like "probably" or "should be", the step is not done.**

You can spot-check any claim without reading code. For example, to confirm the
`$wpdb->update` shortcut really sits at line 127:

```bash
sed -n '127p' backend/wp-content/themes/pc/app/rest-api/WalletController.php
```

**Expected:** a line containing `$wpdb->update(`.

---

## 8. The documentation no longer claims there is no check command

```bash
cd ~/Projects/full-stack/pusher-coin
grep -n "There is none" docs/TECH-STACK.md
```

**Expected:** no output. That sentence is gone.

```bash
grep -n "stripe" CLAUDE.md
```

**Expected:** a line listing `stripe` in the feature table. (Copying the commands
section across risked deleting this row — it must still be there.)

---

## When you are done

Stop the local database if you don't need it:

```bash
cd ~/Projects/full-stack/pusher-coin/backend && ddev stop
```

**If everything above matched, the step passes.** If any step did not match, say which
number failed and what you saw instead — that goes to `/fix-step`, not a direct patch.

## Known and accepted, not faults

- `docs/features/stripe/FEATURE.md` is 96 lines against its own ≤80-line guideline.
  The audit detail is what the step asked for; recorded in `docs/LEARNINGS.md`.
- `docs/PROJECT-TREE.md` lists a `frontend/CLAUDE.md` that no longer exists (the
  frontend repo deleted it). Pre-existing, outside this step — an `/adhoc` candidate.
- `admin/` has no `bin/check`. Expected; its gate is `npm run lint && npm run build`.
- `main` still has no check command anywhere. By design — it arrives at the sprint
  boundary, per `DECISIONS.md` 2026-09-17.
