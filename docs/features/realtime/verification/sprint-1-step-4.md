# Verification — `realtime` Sprint 1, Step 4: The ingest endpoint

**What this step was for.** Until now nothing outside WordPress could tell it that the
machine had paid coins out — the only way was to run a command by hand. This step builds
the **door** those events come in through: a web address the machine's events get posted
to. After this step:

- there is an address, `POST /pc/v1/machine/events`, that accepts a machine event and
  credits the player holding the turn;
- it is **locked with a password of your own** (a "secret") that only the sender knows;
- the **same event sent twice pays only once** — this is the whole point, because every
  delivery method can send the same thing twice;
- an event that arrives when **nobody is playing** is filed and paid to nobody;
- an event that could not even be filed says so, so the sender tries again instead of
  quietly losing a payout.

**What it does not do:** nothing sends events to this door yet. The machine still does
not reach WordPress by itself — that is Step 5. This check is you playing the part of the
sender, with `curl`.

Everything below happens on your own machine. Nothing is deployed.

```bash
cd ~/Projects/full-stack/pusher-coin
```

---

## 1. The branches are where they should be

```bash
echo "docs:     $(git rev-parse --abbrev-ref HEAD)"
for d in backend frontend admin; do echo "$d: $(git -C $d rev-parse --abbrev-ref HEAD)"; done
```

**Expected:**
```
docs:     realtime/sprint-1
backend: realtime/sprint-1
frontend: main
admin: main
```
`backend` is on the sprint branch so the local site runs the new code. The player and
admin apps did not change, so they stay on `main`.

---

## 2. Start the local site and run the backend check

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev start
git status --porcelain
bin/check; echo "exit code: $?"
```

**Expected:**
- `git status --porcelain` prints nothing. If it prints `M wp-config-ddev.php`, run
  `git checkout -- wp-config-ddev.php`; that is the known DDEV quirk.
- The check ends with `DDEV checks: passed — machine-ingest.php machine-rooms.php
  stripe-client.php stripe-webhook.php wallet-rollback.php` and `exit code: 0`.

`machine-ingest.php` is this step's automated test: **80 checks**, including the
crash-the-database cases you cannot produce by hand.

---

## 3. Give the door a secret (once)

The door is locked and has no key yet. Make one. **Stay in this same Terminal window for
the rest of the guide** — the secret lives in it and is never printed to your screen.

```bash
cd ~/Projects/full-stack/pusher-coin/backend
SECRET=$(ddev wp eval 'echo wp_generate_password( 48, false );' 2>/dev/null | tr -d '\r\n')
ddev wp config set PC_MACHINE_INGEST_SECRET "$SECRET" --type=constant > /dev/null && echo "secret written to wp-config.php"
```

**Expected:** `secret written to wp-config.php`.

`wp-config.php` is not in git, so this value stays on your machine only. If you ever want
to remove it: `ddev wp config delete PC_MACHINE_INGEST_SECRET --type=constant`.

> **Before this is set, every call to the door is refused.** If you want to see that for
> yourself, do section 8 *before* this one.

---

## 4. Check what a bonus is worth

```bash
ddev wp option get pc_machine_bonus_map 2>/dev/null
```

**Expected:** a list where `"7":5` appears — bonus number 7 is worth **5 coins**. If your
map says something else for 7, use your own number wherever this guide says 5.

---

## 5. Take a turn in the player app

Start the player app in a **second** Terminal window and leave it running:

```bash
cd ~/Projects/full-stack/pusher-coin/frontend
npm run dev
```

**Expected:** a line with `http://localhost:5173/`.

1. **Sign in.** Open `http://localhost:5173/sign-in` and sign in as **jano4@gmail.com**
   (a local test player with 8 coins). For the 6-digit code, open
   `https://pusher-coin.ddev.site:8026` (Mailpit), open the newest email and type the
   code in.
2. **Open the room.** Go to `http://localhost:5173/room/11` — **Sunset Pusher**.
3. **Join the queue.** In the **Place bet** panel, declare **1** coin and join. You should
   appear at the top of the queue as the player whose turn it is.
4. **Leave this browser tab open and visible.** The turn is only held while the page is
   open — it asks the server every 3 seconds, and a turn goes quiet after 60 seconds
   without that. If you close the tab, the payout in section 6 will land on nobody.
5. **Note your two numbers** on the screen: the **balance** (coins you own) and the
   **winnings** for this turn (it should be 0).

---

## 6. Send a bonus event — the main check

Back in the **first** Terminal window (the one holding `$SECRET`):

```bash
curl -sS -w "\nHTTP %{http_code}\n" \
  -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/machine/events \
  -H "Content-Type: application/json" \
  -H "X-PC-Machine-Secret: $SECRET" \
  -d '{"type":"bonus","event_key":"manual-check-1","machine_id":"demo_sunset","bonus_number":7}'
```

**Expected:**
```
{"received":true,"status":"credited","coins":5,"event_id":NN}
HTTP 200
```
(`NN` is a row number; any number is fine.)

**In the browser, within about 3 seconds:** the **winnings** for this turn go from 0 to
**5**.

**The balance is a known quirk:** it does not refresh by itself on a machine payout —
only winnings do. **Reload the page** (or open `http://localhost:5173/account`) and the
balance is 5 higher than you noted. Making the balance move by itself is Sprint 2's job.

If you would rather read the balance from the command line:

```bash
ddev wp eval 'echo PC\Wallet_Service::get_wallet( 5 )["balance_coins"], "\n";' 2>/dev/null
```

**If you got `"status":"unattributed"` instead:** nobody was holding the turn — the
browser tab was closed, or the turn had gone quiet. Re-join the queue (section 5), then
run the command again **with a different `event_key`** (say `manual-check-1b`), because
the door refuses to pay the same event twice — which is the next check.

---

## 7. Send the identical event again — nothing must happen

Run **exactly the same command as section 6**, unchanged:

```bash
curl -sS -w "\nHTTP %{http_code}\n" \
  -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/machine/events \
  -H "Content-Type: application/json" \
  -H "X-PC-Machine-Secret: $SECRET" \
  -d '{"type":"bonus","event_key":"manual-check-1","machine_id":"demo_sunset","bonus_number":7}'
```

**Expected:**
```
{"received":true,"status":"already_recorded","coins":0}
HTTP 200
```

**Must NOT happen:** the winnings or the balance changing again. Check both in the
browser — they stand still. This is the single most important behaviour in the step: a
real delivery method will send the same event twice, and the player must be paid once.

---

## 8. Two refusals — what must NOT be possible

**A wrong secret is refused, and tells the caller nothing:**

```bash
curl -sS -w "\nHTTP %{http_code}\n" \
  -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/machine/events \
  -H "Content-Type: application/json" \
  -H "X-PC-Machine-Secret: wrong-secret" \
  -d '{"type":"bonus","event_key":"manual-check-2","machine_id":"demo_sunset","bonus_number":7}'
```

**Expected:**
```
{"code":"machine_ingest_unauthorized","message":"Not authorised.","data":{"status":401}}
HTTP 401
```
**Must NOT happen:** any coins moving, or the answer hinting at *why* it was refused —
"Not authorised." is all an attacker learns, whether the secret was wrong, missing, or
never set up.

**An event with no identity is refused:**

```bash
curl -sS -w "\nHTTP %{http_code}\n" \
  -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/machine/events \
  -H "Content-Type: application/json" \
  -H "X-PC-Machine-Secret: $SECRET" \
  -d '{"type":"bonus","machine_id":"demo_sunset","bonus_number":7}'
```

**Expected:**
```
{"code":"missing_event_key","message":"An event_key is required: without one a repeat would be credited twice.","data":{"status":400}}
HTTP 400
```
That `event_key` is what makes section 7 possible. Without one there is no way to tell a
repeat from a new payout, so the door refuses the event outright rather than risk paying
twice.

---

## 9. The payout is not in the player's money history

In the browser, open `http://localhost:5173/history`.

**Expected:** no new entry for the 5 coins. The History screen lists money going in and
out — top-ups and withdrawals — and machine winnings are deliberately not itemised there.
They are on record separately:

```bash
ddev wp eval 'foreach ( PC\Machine_Event_Log::recent( 3 ) as $e ) { echo $e["created_at"], " ", $e["event_type"], " ", $e["status"], " coins=", $e["coins_credited"], " key=", $e["event_key"], "\n"; }' 2>/dev/null
```

**Expected:** a `bonus` line with `credited` and `coins=5`, keyed `manual-check-1` —
recorded **once**, even though you sent it twice.

---

## 10. Clean up

The 5 coins are real in your local database. To put the test player back where it was, and
remove the events this check created:

```bash
ddev wp eval '
global $wpdb;
$wpdb->query( "DELETE FROM {$wpdb->prefix}pc_machine_events WHERE event_key LIKE \"manual-check-%\"" );
echo "events removed\n";
' 2>/dev/null
```

Leaving the extra coins on jano4 is harmless — it is a local test account. You can also
leave the secret in `wp-config.php`; Step 5 will want it.

In the player app, leave the queue (or just close the tab).

---

## What was checked by whom

- **Checked by the agent:** every command in sections 1–4 and 6–9 was run against a
  throwaway room and player before this guide was written, over real HTTP — the credit,
  the repeat, both refusals, and that no money-history row appears. The throwaway data and
  the temporary secret were removed afterwards.
- **Your part:** the browser — signing in, holding the turn, and watching winnings and
  balance move. The agent does not sign in to the apps.
- **Not checkable here at all:** the production server. The ingest secret must be added to
  `wp-config.php` **on the host** before any real event can arrive; a deploy cannot write
  that file. Step 5's verification depends on it.

If any check in sections 1–9 does not match, report it with `/fix-step <what you saw>`.
