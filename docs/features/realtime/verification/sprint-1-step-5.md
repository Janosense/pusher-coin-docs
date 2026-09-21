# Verification — `realtime` Sprint 1, Step 5: The transport

**What this step built.** Until now the machine could pay out and nothing noticed.
Step 4 built a door for machine events; nothing knocked on it. This step built the
thing that knocks: once a minute WordPress asks the machine's Home Assistant what the
coin counter did since it last looked, works out how many coins fell, and credits the
player holding the turn — **with nobody running a command**.

**What you are checking here.** That the schedule is real and ticking; that the poller
refuses to guess when it is half-configured and loses nothing while it waits; and, on
production, that a real payout reaches a real player. The arithmetic itself (how many
coins a change is worth, paying once for an event that arrives twice, catching up after
an outage) is checked by 71 automated checks in section 2 — you do not have to
reproduce those by hand.

**Time:** about 20 minutes locally. The production part waits for the next deploy and a
venue day.

**Nothing here touches the machine.** No button is pressed, no power is switched. The
one optional step that talks to the real Home Assistant only *reads*, and credits
nothing.

---

## 1. Get the code and start the local site

```bash
cd ~/Projects/full-stack/pusher-coin/backend
git checkout realtime/sprint-1
ddev start
```

```bash
cd ~/Projects/full-stack/pusher-coin
git checkout realtime/sprint-1
```

---

## 2. Run the automated checks

```bash
cd ~/Projects/full-stack/pusher-coin/backend
bin/check
```

**Expect** the last three lines to be:

```
== Summary
php -l:      58 files OK
DDEV checks: passed — machine-ingest.php machine-poll.php machine-rooms.php stripe-client.php stripe-webhook.php wallet-rollback.php
```

`machine-poll.php` must appear in that list. If instead you see a box saying
`SKIPPED: DDEV checks did not run`, the site is not running — go back to `ddev start`
and run it again, because that is the run that checks the money.

Those 71 checks include a real credit: a player holding a turn in a room, the machine
"paying out" five coins, the five coins landing on their balance at the price they
bought coins at, and **no row in their money history** (a machine win is not a
transaction). They also check that the same payout arriving twice pays once.

---

## 3. Confirm the schedule exists and ticks

This is the "nobody runs a command" part.

```bash
ddev wp cron event list --fields=hook,next_run_relative,recurrence | head -1
ddev wp cron event list --fields=hook,next_run_relative,recurrence | grep pc_realtime_poll
```

**Expect** a line like:

```
pc_realtime_poll   21 seconds   1 minute
```

The recurrence must say **1 minute**. That is the schedule the whole promise rests on:
coins fall, and within about a minute the player sees them.

Now fire it by hand, exactly as the schedule will:

```bash
ddev wp cron event run pc_realtime_poll
```

**Expect:**

```
Executed the cron event 'pc_realtime_poll' in 0.00Xs.
Success: Executed a total of 1 cron event.
```

---

## 4. Set the two things the transport needs

The poller needs two settings, and **neither can be guessed**. One is a secret, one
names the machine.

```bash
SECRET=$(ddev wp eval 'echo wp_generate_password( 48, false );' 2>/dev/null | tr -d '\r\n')
ddev wp config set PC_MACHINE_INGEST_SECRET "$SECRET" --type=constant > /dev/null
unset SECRET
```

(The secret is generated and stored without ever being shown. If you already set one
while verifying Step 4, that one still works and you can skip this.)

Leave the machine id unset for now — the next section checks what happens without it.

---

## 5. Negative check A — half-configured credits nothing, and loses nothing

```bash
ddev wp option delete pc_realtime_poll_machine_id
ddev wp pc machine-poll
```

**Expect** the last line to be a warning:

```
Warning: Not configured: set `pc_realtime_poll_machine_id` and PC_MACHINE_INGEST_SECRET.
The cursor was held, so nothing inside Home Assistant's retention is lost.
```

and the lines above it to say `0 credited, 0 coin(s)`.

**This is the important behaviour, not an error.** A poller that ran anyway would file
every real payout as belonging to nobody and retire it unpaid. Instead it stops, keeps
its place, and says so. The moment the setting arrives, everything the machine still
remembers (ten days) is picked up and paid.

> The message names both settings without saying which one is missing. That is
> deliberate — it is the operator's checklist. Which one it was is recorded in the
> audit log.

Now give it the machine id:

```bash
ddev wp option update pc_realtime_poll_machine_id demo_sunset
```

`demo_sunset` is the machine id the local **Sunset Pusher** room carries. On production
this must be the id the live room carries, or payouts land on nobody.

---

## 6. Negative check B — an unreachable machine loses nothing either

Your local site has no Home Assistant token, so the machine is unreachable — which is
exactly the failure worth checking.

```bash
ddev wp pc machine-poll
```

**Expect:**

```
window 2026-…T08:02:26+00:00 .. 2026-…T09:02:26+00:00
0 history row(s), 0 payout(s)
0 credited, 0 coin(s)
Warning: Home Assistant could not be read. The cursor was held and the next pass re-reads the same window.
```

Note the difference from section 5: it now names a real time window (it got far enough
to know what to ask for) and gives a different reason. Both keep their place.

Check that it wrote down what happened:

```bash
ddev wp pc machine-poll --status
```

**Expect** something like:

```
last pass: 2026-…T09:02:26+00:00
0 row(s), 0 payout(s), 0 coin(s)
stopped: read_failed
next WP-Cron tick due: 2026-…T09:02:58+00:00
```

That command is how you will answer "is the transport actually running?" on the
production server, where nobody has a terminal into the site.

---

## 7. Optional — point it at the real machine, read-only

Skip this if you would rather not handle the Home Assistant token. It proves the
poller can read the real machine's history; it credits nothing and presses nothing.

The token must not be typed on a command line or pasted into a terminal, so put it in
the config file with your editor:

1. Open `backend/wp-config-ddev.php`.
2. Below the Stripe lines, add: `define('PC_MACHINE_TOKEN', 'paste-the-token-here');`
   — the token is in `~/.pusher-coin-ha-token`.
3. Save. **Do not commit this file.**

```bash
ddev wp pc machine-poll --dry-run
```

**Expect** a real window, a non-zero number of history rows if the machine has been on
recently, and a line ending:

```
… history row(s), … payout(s) — dry run, nothing delivered and the cursor untouched
```

`--dry-run` is the safe way to look: it reads, works out what it *would* pay, and then
does nothing at all.

**When you are done, take the token back out:**

```bash
cd ~/Projects/full-stack/pusher-coin/backend
git checkout -- wp-config-ddev.php
```

(That file is tracked, so this restores it exactly. `ddev start` would also wipe your
edit — see `docs/LEARNINGS.md` 2026-09-15.)

---

## 8. Clean up the local site

```bash
ddev wp option update pc_realtime_poll_machine_id ''
```

Leave the ingest secret — it is local only, and Step 4's verification wants it too.

---

## 9. Production — what an operator must do, and the real proof

**This is the part no local check can stand in for.** Two settings live on the server
and no deploy writes either of them:

1. **The ingest secret** — add to `wp-config.php` on the host, over FTP:
   `define( 'PC_MACHINE_INGEST_SECRET', '<a long random string>' );`
2. **The machine id** — set `pc_realtime_poll_machine_id` to the id the live room
   carries.

Until both exist the transport runs and holds; nothing is lost for ten days, which is
how long the machine's own history keeps events.

**Optional, if the host offers real cron jobs.** The transport runs on WordPress's own
scheduler by default, which only ticks when the site gets traffic — fine here, because
a player holding a turn is generating traffic every three seconds. If you would rather
not depend on that, add:

```cron
* * * * * cd /path/to/wordpress && wp pc machine-poll --quiet >/dev/null 2>&1
```

Both together are safe — they cannot double-pay.

**The real check, during venue hours:** a player takes a turn and the machine pays out.

- **Within about a minute**, that player's **winnings** for the turn go up on the
  **Room** screen, with nobody running anything.
- Their **History** screen shows **no new row** — a machine win is not a transaction.

> **Expect the balance number to lag.** The winnings counter updates on its own; the
> balance figure only refreshes when the page is reloaded or the player tosses again.
> That is how the player app is built today, and the live push that fixes it is
> Sprint 2. It is not a fault in this step.

Afterwards, `wp pc machine-poll --status` on the host should show a recent pass with
rows and payouts, and no `stopped:` line.

---

## 10. What was checked by whom

**Checked by the agent, locally:**

- The 71 automated checks, including the arithmetic, a real credit to a player holding
  a turn at their own coin price with no ledger row, the same payout arriving twice
  paying once, a ten-minute catch-up, and every "stop and hold" case.
- That the schedule registers at one minute and the event fires.
- Both negative checks in sections 5 and 6, with the exact messages shown above.
- `wp pc machine-poll --status`, and `--dry-run` writing nothing.
- That the checks leave no test data behind.

**Not checked by the agent:**

- **The real Home Assistant** — no token was configured locally and nothing was read
  from the live machine. Section 7 is yours if you want it.
- **The browser.** The agent does not sign in to the player or admin app, so the Room
  screen and the History screen were not seen.
- **Production** — the secret, the machine id, the cron line and a real payout all wait
  for the next deploy from `main` and a venue day.
