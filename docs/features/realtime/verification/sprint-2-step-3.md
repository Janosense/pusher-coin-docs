# Verification — `realtime` Sprint 2, Step 3: The relay lock the player can see

**What this step built — and the bug it found.** The machine has a relay that somebody
at the venue can open to take it out of service. The code had its meaning **backwards**:
it treated the relay's normal state as "the machine is paying out" and refused **every
toss while the machine was on**. That is fixed. The relay is now read once a minute, a
change is pushed to the room, and the toss button greys out — with the reason — while
the machine is out of service.

**What you are checking.** That a normal machine lets you play; that a machine out of
service greys the button out and says why; that nothing about this can cost you a coin;
and that a machine nobody can read is *not* treated as locked.

**Time:** about 20 minutes for sections 1–6, which need **neither the real machine nor
an Ably account**. Section 7 is the only one that touches the venue.

**Nothing here switches the machine's power.** Section 7 presses the machine's relay
button once and puts it straight back; everything before it runs against your laptop.

---

## 1. Get the code and start the local site

```bash
cd ~/Projects/full-stack/pusher-coin/backend
git checkout realtime/sprint-2
ddev start
```

```bash
cd ~/Projects/full-stack/pusher-coin
git checkout realtime/sprint-2

cd ~/Projects/full-stack/pusher-coin/frontend
git checkout realtime/sprint-2
npm ci
```

---

## 2. Run the automated checks

```bash
cd ~/Projects/full-stack/pusher-coin/backend
bin/check
```

**Expect** the last three lines, with `realtime-relay.php` in the list:

```
== Summary
php -l:      65 files OK
DDEV checks: passed — machine-ingest.php machine-poll.php machine-rooms.php realtime-channel.php realtime-queue.php realtime-relay.php stripe-client.php stripe-webhook.php wallet-rollback.php
```

To see the relay checks by name — **51 of them**:

```bash
ddev wp eval-file wp-content/themes/pc/tests/realtime-relay.php
```

**Expect** `Success: 51 checks passed.` The first line of the run is the one that
matters most:

```
PASS  lock: a closed relay is the machine working — the toss goes through
```

That check **fails** on the code as it was before this step. Then:

```bash
cd ~/Projects/full-stack/pusher-coin/frontend
bin/check
```

**Expect** `lint: OK` and `build: OK`.

> **Worth knowing.** The backend half of this step has 51 automated checks. The browser
> half has **none** — the player app has no test runner. Sections 4 and 5 below are the
> only thing that exercises what you actually see on screen.

---

## 3. What the server answers now

The room's stored answer for "is the machine out of service?" lives in one setting. It
is empty on your machine, which means *not* locked:

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp eval 'echo json_encode( PC\Queue_Service::state( 11 )["machine_locked"] ), "\n";'
```

**Expect** `false`. Now say the machine has been taken out of service:

```bash
ddev wp option update pc_realtime_relay_state --format=json '{"locked":true,"at":"2026-09-21T13:00:00+00:00"}'
ddev wp eval 'echo json_encode( PC\Queue_Service::state( 11 )["machine_locked"] ), "\n";'
```

**Expect** `Success: Updated …` and then `true`.

**Leave it set** — section 4 is about what the room does with it.

---

## 4. The button greys out, and says why

1. Start the player app:
   ```bash
   cd ~/Projects/full-stack/pusher-coin/frontend
   npm run dev
   ```
2. Open `http://localhost:5173/room/11` (the **Sunset Pusher** room) and sign in as a
   verified player who has coins — on your install, `jano4` has 8.
3. **Join the queue** with 1 coin. Because nobody else is queued, the turn is yours
   straight away.

**Expect**, where the **Toss a coin** button normally is:

- the button is **greyed out and cannot be clicked**;
- underneath it, instead of the usual "Your turn — 1 coin(s) left to play", the line
  **"The machine is out of service right now. Your turn is kept — you can toss as soon
  as it is back."**

Now put the machine back in service:

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp option delete pc_realtime_relay_state
```

Reload `http://localhost:5173/room/11`.

**Expect** the **Toss a coin** button to be live again, with the usual "Your turn —
1 coin(s) left to play" note under it.

> If you press **Toss a coin** on your laptop it will answer *"The machine is not set up
> yet."* — your local site has no Home Assistant token, so there is no machine to toss
> into. That is the expected local answer, not a failure of this step.

---

## 5. Negative check — you can still queue, and nothing can cost you a coin

With the room still open, set the machine out of service again and reload:

```bash
ddev wp option update pc_realtime_relay_state --format=json '{"locked":true,"at":"2026-09-21T13:00:00+00:00"}'
```

**Expect:**

- **Leave queue / Give up the turn still works.** Being locked out of the toss must not
  trap you in the queue.
- **Joining still works.** Leave the queue and join again — a player may line up for a
  machine that is being serviced. Only the toss is refused.
- **Your coin balance never moves.** Check it in the header before and after everything
  above: the number is the same. Nothing in this step debits anything — the refusal
  happens before any coin is taken, and six of the automated checks exist to say so.

Then clear it again:

```bash
ddev wp option delete pc_realtime_relay_state
```

---

## 6. Negative check — a machine nobody can read is NOT a locked machine

This is the failure that would be worst in practice: the venue's internet drops, the
site cannot read the machine, and every room greys out its button for no reason. It
must not happen.

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp option update pc_realtime_poll_machine_id demo_sunset
ddev wp pc machine-poll --dry-run
```

Your local site has no Home Assistant token, so this is exactly the "cannot be read"
case. **Expect** these two lines:

```
relay: could not be read; the last known state was left alone
Warning: Home Assistant could not be read. …
```

And the room is still playable:

```bash
ddev wp eval 'echo json_encode( PC\Realtime_Relay_Watch::locked() ), "\n";'
```

**Expect** `false` — **not** `true`. An unreadable machine is not a locked one.

Put the setting back:

```bash
ddev wp option update pc_realtime_poll_machine_id ''
```

---

## 7. The real machine — one relay press, at the venue (optional)

**Read this whole section before running anything in it.**

This is the only part that touches the real machine, and it **takes the venue's machine
out of service for as long as it takes you to run the next two commands**. Do it when
nobody is playing. It presses the machine's **relay** button — the same one the venue's
own controls press — and **never** its power.

You need the Home Assistant token on your local site. Put it in by hand, never on a
command line:

1. Open `~/Projects/full-stack/pusher-coin/backend/wp-config-ddev.php` in an editor.
2. Add, with your real token pasted between the quotes:
   `define('PC_MACHINE_TOKEN', '…');`
3. Save. **Remove this line again when you are finished.**

Then, with `npm run dev` running and `http://localhost:5173/room/11` open on your turn:

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp option update pc_realtime_poll_machine_id demo_sunset

# Take the machine out of service.
ddev wp eval 'var_dump( PC\Machine_Service::relay_open() );'
ddev wp pc machine-poll
```

**Expect** `relay: OPEN — the machine is out of service and tosses are refused` and, in
the room within a few seconds of that command, the **Toss a coin** button greyed out
with the out-of-service line under it. (Without an Ably account the room finds out on
its next queue read rather than instantly; with one it is immediate.)

**Put it straight back:**

```bash
ddev wp eval 'var_dump( PC\Machine_Service::relay_close() );'
ddev wp pc machine-poll
```

**Expect** `relay: closed — the machine is working normally`, and the button live again.

**The negative check here:** while the relay was open, pressing **Toss a coin** — if you
got there before the button greyed out — must answer **"The machine is out of service.
Try again shortly."** and **must not** take a coin. The server reads the relay itself on
every toss, so it refuses even when the room has not heard yet.

Finally, remove the `PC_MACHINE_TOKEN` line from `wp-config-ddev.php` and reset the
setting:

```bash
ddev wp option update pc_realtime_poll_machine_id ''
ddev wp option delete pc_realtime_relay_state
```

---

## 8. What was checked by whom

**Checked by the agent, locally:**

- The 51 automated backend checks: a closed relay lets a toss through (this is the bug
  that was fixed, and ten of the first sixteen checks fail on the old code); an open
  relay refuses before any coin is debited; a refused toss leaves the wallet, the coin
  lots, the declared coins and the event log untouched; a change is announced exactly
  once and only when it is a change; the message carries `locked` and names neither the
  sensor nor the machine; a dead Ably changes nothing; an unreadable relay locks
  nothing; the relay watch and the coin transport cannot stop each other; the queue
  reply carries the lock without ever calling Home Assistant.
- Every command in sections 3 and 6 above, with the outputs shown.
- `wp pc machine-poll`, `--dry-run` and `--status`, all three printing the relay line.
- Both check commands.

**Not checked by the agent:**

- **Everything you see on screen** — sections 4, 5 and the room half of 7. `frontend/`
  has no test runner and the agent does not sign in to the player app.
- **Anything against the real machine.** Nobody has yet watched the relay move for real,
  and — this is worth knowing — **nobody has ever observed what an open relay does to a
  toss physically**. The lock is a precaution taken on what the machine's own buttons
  are documented to mean (`DECISIONS.md` 2026-09-21).
- **Anything on a real Ably channel** — still no account, so the push half of section 7
  falls back to the room's next read.
