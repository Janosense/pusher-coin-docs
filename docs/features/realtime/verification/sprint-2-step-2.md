# Verification — `realtime` Sprint 2, Step 2: The queue subscribes instead of polling

**What this step built.** Until now the Room screen asked the server "anything new?"
every three seconds, for every player watching, forever. Now it is *told*: a join, a
leave or a toss is announced on the room's live channel, and the browser re-reads only
when it hears that something changed. What is left of the old poll is a small write
every 20 seconds that holds your place in the queue.

**What you are checking.** That a player cannot lose their place; that the room still
works when the live channel is not available; and — once there is an Ably account —
that two browsers see the same thing at the same moment.

**Time:** about 20 minutes for sections 1–6, which need **no Ably account**. Section 7
is the two-browser check and does need one.

**Nothing here touches the machine.**

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

`npm ci` matters this time: this step adds one new package (`ably`), so an old
`node_modules` will not build.

---

## 2. Run the automated checks

```bash
cd ~/Projects/full-stack/pusher-coin/backend
bin/check
```

**Expect** the last three lines:

```
== Summary
php -l:      63 files OK
DDEV checks: passed — machine-ingest.php machine-poll.php machine-rooms.php realtime-channel.php realtime-queue.php stripe-client.php stripe-webhook.php wallet-rollback.php
```

`realtime-queue.php` must be in that list. Then:

```bash
cd ~/Projects/full-stack/pusher-coin/frontend
bin/check
```

**Expect** `lint: OK` and `build: OK`.

> **Worth knowing before you go further.** The backend half of this step has 28
> automated checks. The browser half has **none** — the player app has no test runner
> and this step did not add one. Everything you do in sections 5–7 is the only thing
> that exercises the subscription, the reconnect and the fallback. That is why this
> guide asks you to do more clicking than usual.

---

## 3. The heartbeat holds a place, and says nothing else

This is the piece that replaced the poll. Run it by hand against the local **Sunset
Pusher** room:

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp eval '
$u = get_users(["role"=>"player","number"=>1])[0];
wp_set_current_user($u->ID);
echo "as: {$u->user_login}\n";
$r = rest_do_request( new WP_REST_Request("POST","/pc/v1/rooms/11/queue/heartbeat") );
echo "HTTP " . $r->get_status() . "\n" . json_encode($r->get_data(), JSON_PRETTY_PRINT) . "\n";
'
```

**Expect `HTTP 200`** and exactly this shape — a version and nothing else:

```json
{
    "version": "d75171398898"
}
```

**No queue, no player names, no coin counts.** That is the point: the live channel can
be listened to by anyone with an account, while the queue itself needs a fully
verified one. So the channel and this response carry only "something changed", and the
real queue is still fetched through the gate it always was.

---

## 4. The version moves only when something really changed

```bash
ddev wp eval '
$u = get_users(["role"=>"player","number"=>1])[0]; wp_set_current_user($u->ID);
$hb = fn() => ((array) rest_do_request( new WP_REST_Request("POST","/pc/v1/rooms/11/queue/heartbeat") )->get_data())["version"];
echo "1: " . $hb() . "\n2: " . $hb() . "  (must match)\n";
$j = rest_do_request((function(){ $r = new WP_REST_Request("POST","/pc/v1/rooms/11/queue/join"); $r->set_param("coins",1); return $r; })());
echo "join HTTP " . $j->get_status() . "\n";
echo "3: " . $hb() . "  (must differ)\n";
rest_do_request( new WP_REST_Request("POST","/pc/v1/rooms/11/queue/leave") );
echo "after leave: " . $hb() . "\n";
'
```

**Expect** lines 1 and 2 to be identical, line 3 to be different, and the last line to
return to the first value:

```
1: d75171398898
2: d75171398898  (must match)
join HTTP 200
3: 348d4b33b713  (must differ)
after leave: d75171398898
```

That is the whole mechanism: the browser holds the last version it saw, and re-reads
the queue only when the one it gets back is different. Two identical values mean the
room did nothing and no read was needed.

---

## 5. Negative check — the room still works with no live channel at all

**Your local site has no Ably key**, which is exactly the state to check first: the
room must keep working, falling back to the old 3-second poll.

1. Start the player app:
   ```bash
   cd ~/Projects/full-stack/pusher-coin/frontend
   npm run dev
   ```
2. Open `http://localhost:5173/room/11` and sign in as a verified player.
3. Join the queue with 1 coin.

**Expect:** the queue shows you, the turn is yours, the toss button behaves as it
always did. Everything works — it is simply polling rather than listening.

> **The 3-second interval is still in the code, deliberately.** The sprint's own
> notes expected it to be deleted; it is kept as this fallback, because a room that
> polls is better than a room that has frozen. The real check is not that the code is
> gone, but that it does not *run* while the channel is up — which is section 7.

Leave the queue when you are done.

---

## 6. Negative check — a place is not lost to a moment's silence

The heartbeat runs every 20 seconds and a place survives 60, so two missed beats are
harmless. Confirm the timeout still bites when it should:

```bash
ddev wp eval '
$u = get_users(["role"=>"player","number"=>1])[0]; wp_set_current_user($u->ID);
$r = new WP_REST_Request("POST","/pc/v1/rooms/11/queue/join"); $r->set_param("coins",1);
rest_do_request($r);
echo "joined: " . (PC\Queue_Service::entry(11, $u->ID) ? "yes" : "no") . "\n";
global $wpdb;
$wpdb->query( $wpdb->prepare("UPDATE {$wpdb->prefix}pc_room_queues SET last_seen_at = %s WHERE room_id = 11 AND user_id = %d", gmdate("Y-m-d H:i:s", strtotime(current_time("mysql")) - 600), $u->ID) );
PC\Queue_Service::prune_stale(11);
echo "after 10 minutes of silence: " . (PC\Queue_Service::entry(11, $u->ID) ? "still there (WRONG)" : "pruned (correct)") . "\n";
'
```

**Expect** `joined: yes` then `pruned (correct)`. A player who stops talking to the
server loses their place — that rule is unchanged; only what counts as "talking" has.

---

## 7. The real check — two browsers, one room (needs an Ably account)

**This cannot be done yet: there is no Ably account.** When you have one (the free
tier is enough — see section 7 of the Step 1 guide):

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp config set PC_ABLY_KEY 'paste-the-real-key-here' --type=constant
```

Then, with `npm run dev` running:

1. Open `http://localhost:5173/room/11` in two different browsers (or one normal and
   one private window), signed in as **two different verified players**.
2. In browser A, **join the queue**.
   **Expect** browser B's queue to show A **immediately** — not after a wait of up to
   three seconds. This is the difference the whole sprint exists for.
3. In browser B, join too. **Expect** browser A to show B immediately.
4. **Take the network away from browser B for about 20 seconds** — turn off Wi-Fi, or
   use its developer tools' "Offline" setting — then restore it.
   **Expect** when it comes back: B's queue is correct, B is still in it in the same
   position, and B has not lost its turn. Nothing needs reloading.
5. In browser A, **leave the queue**. **Expect** B to see it at once.

**A negative check while you are here.** With the channel up, browser B should not be
polling. Open its developer tools → **Network**, filter for `queue`, and watch for ten
seconds.
**Expect:** a `heartbeat` request roughly every 20 seconds, and a `queue` request only
when something actually happened. **What must NOT appear is a steady stream of `queue`
requests every 3 seconds.** If you see that, the fallback is running and the channel
is not connected.

---

## 8. Clean up

```bash
ddev wp config delete PC_ABLY_KEY     # only if you set one
```

Leave the queue in any browser still in it.

---

## 9. What was checked by whom

**Checked by the agent, locally:**

- The 28 automated backend checks: a join, a leave and a play each announce the room;
  the message carries a version and no queue, no nickname and no coin count; the
  version moves only on real change; a dead Ably cannot fail a join, a leave or a
  toss; the heartbeat writes and does not read, holds the caller's place, prunes the
  absent, promotes the next player, and is gated exactly as the queue read is.
- Every command in sections 3, 4 and 6 above, with the outputs shown.
- `frontend/bin/check` — lint and build.

**Not checked by the agent — and this is the step's real gap:**

- **Everything the browser does.** The subscription, the reconnect, the fallback to
  polling and the winnings update have **no automated coverage at all**: `frontend/`
  has no test runner, and the agent does not sign in to the player app. Sections 5 and
  7 are the only verification these get.
- **Anything involving real Ably** — still no account.
