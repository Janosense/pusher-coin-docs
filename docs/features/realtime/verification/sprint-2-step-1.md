# Verification — `realtime` Sprint 2, Step 1: Publish to Ably, and hand the SPA a token

**What this step built.** Sprint 1 got machine events *into* WordPress. This step
starts getting them *out to the browser*, so the room can stop asking "anything new?"
three times a second. Two pieces, neither visible on screen yet:

- **The publisher** — when a payout credits a player, WordPress posts a short message
  to that room's channel at Ably. Ably is the middleman that holds the live
  connections, because the shared host WordPress runs on cannot hold one open.
- **The doorkeeper** — the browser must never see the Ably key, so it asks WordPress
  for a *scoped pass* instead: signed, short-lived, and good only for the channels
  that caller is allowed to hear.

**What you are checking.** That the key never leaves the server; that a player cannot
listen to the operator channel; and that a broken Ably cannot touch a payout. The
arithmetic of all three is checked by 60 automated checks in section 2 — the manual
part is seeing the difference with your own eyes.

**Time:** about 15 minutes locally, with **no Ably account needed**. Section 7 is the
one part that needs a real account, and it can wait.

**Nothing here touches the machine.** No button is pressed, no power is switched.

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
php -l:      62 files OK
DDEV checks: passed — machine-ingest.php machine-poll.php machine-rooms.php realtime-channel.php stripe-client.php stripe-webhook.php wallet-rollback.php
```

`realtime-channel.php` must appear in that list. If you see a box saying
`SKIPPED: DDEV checks did not run`, the site is not running — `ddev start` and try
again, because that is the run that checks the money.

Those 60 checks include the ones that matter most here: a payout credited correctly
while Ably is dead, unreachable, rejecting the key, or not configured at all; and the
Ably key being absent from every byte of the token response.

---

## 3. Give the local site a made-up Ably key

**You do not need a real Ably account for sections 3–6.** The pass is *signed* with
the key, so any well-formed string proves the whole endpoint. Ably itself is never
contacted here.

```bash
ddev wp config set PC_ABLY_KEY 'demo.abcdef:not-a-real-secret-0123456789' --type=constant
```

---

## 4. Look at a player's pass

```bash
ddev wp eval 'wp_set_current_user( get_users(["role"=>"player","number"=>1])[0]->ID ); $r = rest_do_request( new WP_REST_Request("GET","/pc/v1/realtime/token") ); echo "HTTP " . $r->get_status() . "\n" . json_encode($r->get_data(), JSON_PRETTY_PRINT|JSON_UNESCAPED_SLASHES) . "\n";'
```

**Expect `HTTP 200`** and something like:

```json
{
    "token_request": {
        "keyName": "demo.abcdef",
        "ttl": 3600000,
        "capability": "{\"pc:room:*\":[\"subscribe\"]}",
        "clientId": "4",
        "timestamp": 1789990288537,
        "nonce": "IqYxb072ufNuIQ5EDaAtPpvpr9ejMDAC",
        "mac": "PmkJlNkqnJA352MF/onGvlWlPQf5UaQ3BvWVhJCVbcY="
    },
    "channels": {
        "room_pattern": "pc:room:*",
        "machine": null
    }
}
```

Three things to see, and they are the whole point of this step:

1. **`not-a-real-secret-0123456789` does not appear anywhere.** The secret half of
   your key is not in this response and never will be. `keyName` — the part before
   the colon — is public and Ably requires it.
2. **`capability` says `subscribe` and nothing else.** Not `publish`. The browser is
   never allowed to put anything on these channels; everything on them comes from the
   server.
3. **`"machine": null`** — a player is not given the operator channel.

---

## 5. Negative check A — a player cannot reach the operator channel, an admin can

Run the same thing as an administrator:

```bash
ddev wp eval 'wp_set_current_user( get_users(["role"=>"administrator","number"=>1])[0]->ID ); $r = rest_do_request( new WP_REST_Request("GET","/pc/v1/realtime/token") ); echo "HTTP " . $r->get_status() . "\n" . json_encode($r->get_data(), JSON_PRETTY_PRINT|JSON_UNESCAPED_SLASHES) . "\n";'
```

**Expect** the capability line to now read:

```
"capability": "{\"pc:room:*\":[\"subscribe\"],\"pc:machine\":[\"subscribe\"]}",
```

and `"machine": "pc:machine"` below it.

**Compare the two outputs.** The administrator's pass covers `pc:machine`; the
player's does not. That difference is the check — if a player's pass ever lists
`pc:machine`, anybody with an account could listen to the operator's machine feed.

---

## 6. Negative check B — an anonymous caller gets nothing

```bash
ddev wp eval '$r = rest_do_request( new WP_REST_Request("GET","/pc/v1/realtime/token") ); echo "HTTP " . $r->get_status() . "\n" . json_encode($r->get_data(), JSON_PRETTY_PRINT|JSON_UNESCAPED_SLASHES) . "\n";'
```

**Expect `HTTP 401`** and a body that says `rest_forbidden` / "Authentication
required." — **no `keyName`, no `mac`, no `capability`.** Somebody who is not signed
in learns nothing at all.

And with no key configured at all:

```bash
ddev wp config delete PC_ABLY_KEY
ddev wp eval 'wp_set_current_user( get_users(["role"=>"player","number"=>1])[0]->ID ); $r = rest_do_request( new WP_REST_Request("GET","/pc/v1/realtime/token") ); echo "HTTP " . $r->get_status() . "\n" . json_encode($r->get_data(), JSON_PRETTY_PRINT|JSON_UNESCAPED_SLASHES) . "\n";'
```

**Expect `HTTP 503`** and `realtime_not_configured`. A server with no Ably account
says so plainly — and, importantly, **still pays players normally**; it simply does
not push. That is what the automated checks in section 2 prove.

---

## 7. Optional, and it needs a real Ably account — watch a message arrive

This is the one part that cannot be faked, and the one part that is not done yet:
**there is no Ably account.** When you have one:

1. Sign up at [ably.com](https://ably.com) (the free tier is enough: 6M messages a
   month, 200 connections, 200 channels).
2. Copy an app key from the API keys page — it looks like `xxxxxx.yyyyyy:zzzzz…`.
3. Put it in the local site:
   ```bash
   ddev wp config set PC_ABLY_KEY 'paste-the-real-key-here' --type=constant
   ```
4. Open Ably's dashboard for that app, go to the **Dev console** and subscribe to
   `pc:room:11`.
5. Make a payout happen, with the manual replay. **`--player` matters:** a payout
   nobody is holding a turn for is filed as *unattributed*, credits nobody, and is
   therefore not pushed — you would watch an empty console and think it was broken.
   This picks the first player on the install and names them explicitly:
   ```bash
   PID=$(ddev wp eval 'echo get_users(["role"=>"player","number"=>1])[0]->ID;' 2>/dev/null | tr -d '\r\n')
   ddev wp pc machine-ingest coins --coins=3 --machine=demo_sunset --key=ably-check-1 --player=$PID
   ```
   **Expect** `event #NNN recorded as credited.` — "credited", not "unattributed".
   That word is what tells you a message was published.
6. **Expect** a `credit` message to appear in the Dev console within a second, naming
   the room, the player, `"coins": 3` and an event id — **and no price and no
   balance.** A room channel is readable by everyone watching that room, so what
   another player's coins cost them is deliberately not on it.
7. Then break it on purpose: set the key to something wrong and repeat step 5.
   **The player is still credited** (`wp pc machine-ingest` reports the credit), and
   the failure is recorded:
   ```bash
   ddev wp db query "SELECT created_at, metadata FROM wp_pc_auth_audit_log WHERE event_type='realtime_publish_failed' ORDER BY id DESC LIMIT 3;"
   ```

---

## 8. Clean up the local site

If you ran section 7, the replay really credited a player 3 coins and wrote an event
row. Undo both:

```bash
ddev wp eval "PC\\Wallet_Service::debit_fifo( $PID, 3 );"
ddev wp db query "DELETE FROM wp_pc_machine_events WHERE event_key LIKE 'ably-check-%';"
```

Then remove the key:

```bash
ddev wp config delete PC_ABLY_KEY
```

(Or leave a real key in place if you set one — `wp-config.php` is not tracked by git,
so nothing of yours gets committed either way.)

---

## 9. Production — what will be needed later

Nothing to do yet; the sprint is not finished. When it merges, an operator adds
`PC_ABLY_KEY` to `wp-config.php` on the host over FTP, exactly as with the machine
ingest secret — no deploy writes that file. Until then, production credits players
normally and pushes nothing.

---

## 10. What was checked by whom

**Checked by the agent, locally:**

- The 60 automated checks, including a payout credited correctly while Ably is dead,
  unreachable, rejecting the key or unconfigured; the key being absent from the token
  response; the signature recomputed independently; and a player's pass never
  covering the machine channel.
- Every command in sections 3–6 above, with the outputs shown.

**Not checked by the agent:**

- **Anything involving real Ably.** No account exists, so nothing was published to a
  real channel and no dashboard was seen. Section 7 is the first time that happens.
- **The browser.** The agent does not sign in to the player or admin app. Nothing in
  this step is visible there yet in any case — that starts in Step 2.
