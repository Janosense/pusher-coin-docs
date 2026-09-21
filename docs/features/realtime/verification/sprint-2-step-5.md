# Verification — `realtime` Sprint 2, Step 5: The duplicate-session race

**What this step fixed.** A "turn" is the record of one player's stretch at the front of
a room's queue — how many coins they played, how many they won, what that was worth. A
room is supposed to have exactly one open turn at a time, because that record is how the
system knows who a machine payout belongs to.

It didn't always. The code looked to see whether the room already had an open turn and
then created one, and two requests arriving in the same instant both saw "no" and both
created one. The spare turn then stayed open for ever, because the only thing that ever
closes a turn is the queue letting go of it. That is not untidiness: the room screen
shows one of those records and the payout code banked winnings on the other, so **a real
payout could land on a turn the player was not looking at** — money moved, the winnings
counter didn't.

Now the database itself refuses the second turn, and a request that loses the race joins
the winner's turn instead of starting a duplicate. Turns already left hanging are closed
automatically — closed, never deleted, each one written into the audit log with the coins
and money it was holding.

**What you are checking.** That the rule is actually in place; that a second turn is
genuinely impossible rather than merely unlikely; that a room still hands over normally
and is not wedged shut; that a real win lands on the turn the player is looking at; and
that an operator can see the state of any install.

**Time:** about 20 minutes. **No Ably account is needed** — nothing in this step touches
the live channel. **Nothing here touches the machine**, and no coin is thrown at the
venue.

---

## 1. Get the code and start the local site

```bash
cd ~/Projects/full-stack/pusher-coin/backend
git checkout realtime/sprint-2
ddev start
```

If `ddev start` reports that it changed `wp-config-ddev.php`, that is expected and
harmless — leave it.

The fix travels with a database change, which applies itself on the first page load.
Force it now and confirm:

```bash
ddev wp eval 'echo get_option("pc_db_version") . "\n";'
```

**Expected:** `1.13.0` or higher. If you see `1.12.0`, open
`https://pusher-coin.ddev.site` once in a browser and run it again.

---

## 2. The rule is in place

```bash
ddev wp pc queue-sessions
```

**Expected:** a table of the rooms that currently have an open turn — possibly empty,
which is fine — and, as the last line:

```
UNIQUE KEY open_room: present — the database refuses a second open session per room.
```

That sentence is the whole point of the step. If instead you see a yellow **Warning**
saying the key is missing, the step has not taken effect on this install; stop and say so.

The table's **protected** column says `yes` for every turn the rule covers. The
**duplicate** column says `no` for every row. If you see `YES` anywhere, go to section 6.

> On a development machine the table often lists turns belonging to `(no room post)` and
> `(deleted user)`. Those are leftovers from earlier test runs, not real players — they
> are a known piece of housekeeping tracked separately, and they do not affect anything
> here.

---

## 3. Negative check — a second turn must be impossible

This is the check that matters. It asks the database directly to open a second turn for
one room, which is exactly what a racing request does. It creates a throwaway room,
tries twice, and removes everything afterwards.

Paste the whole block:

```bash
ddev wp eval '
global $wpdb; $t = $wpdb->prefix . "pc_bet_sessions";
$room = wp_insert_post( [ "post_type" => "pc_room", "post_title" => "Probe room", "post_status" => "publish" ] );
$row = [ "user_id" => 1, "room_id" => $room, "open_room_id" => $room, "started_at" => current_time( "mysql" ) ];
$one = $wpdb->insert( $t, $row, [ "%d", "%d", "%d", "%s" ] );
$s = $wpdb->suppress_errors( true );
$two = $wpdb->insert( $t, $row, [ "%d", "%d", "%d", "%s" ] );
$wpdb->suppress_errors( $s );
echo "First turn opened:  " . ( false === $one ? "NO" : "yes" ) . "\n";
echo "Second turn opened: " . ( false === $two ? "NO - refused by the database" : "YES - THE RULE IS NOT BEING KEPT" ) . "\n";
echo "Open turns in the room: " . (int) $wpdb->get_var( $wpdb->prepare( "SELECT COUNT(*) FROM $t WHERE room_id = %d AND ended_at IS NULL", $room ) ) . "\n";
$wpdb->query( $wpdb->prepare( "DELETE FROM $t WHERE room_id = %d", $room ) ); wp_delete_post( $room, true );
echo "Probe cleaned up.\n";'
```

**Expected, exactly:**

```
First turn opened:  yes
Second turn opened: NO - refused by the database
Open turns in the room: 1
Probe cleaned up.
```

**This must not say `YES - THE RULE IS NOT BEING KEPT`.** That is the failure the whole
step exists to prevent.

*(Run against the code as it was before this step, the second line would read `yes` and
the count would be `2`.)*

---

## 4. The race itself, with real simultaneous requests

Section 3 proves the database refuses a duplicate. This proves the *application* no
longer asks for one — twelve genuinely simultaneous requests at a single room, which is
what a busy room is.

Paste the whole block. It sets up a throwaway room and player, waits ten seconds so all
twelve processes start together, then reports.

```bash
cd ~/Projects/full-stack/pusher-coin/backend

ROOM=$(ddev wp eval '
$r = wp_insert_post( [ "post_type"=>"pc_room", "post_title"=>"Race probe", "post_status"=>"publish" ] );
update_post_meta( $r, PC\Post_Meta_Keys::ROOM_STATUS, "available" );
$u = wp_insert_user( [ "user_login"=>"race-probe", "user_email"=>"race-probe@example.test", "user_pass"=>wp_generate_password(18,false), "role"=>"player" ] );
global $wpdb; $wpdb->insert( $wpdb->prefix."pc_room_queues", [ "room_id"=>$r, "user_id"=>$u, "coins_declared"=>5, "coins_remaining"=>5, "joined_at"=>current_time("mysql"), "last_seen_at"=>current_time("mysql") ], ["%d","%d","%d","%d","%s","%s"] );
echo $r;' 2>/dev/null | grep -v Deprecated)

GO=$(( $(date +%s) + 10 ))
for i in $(seq 1 12); do
  ddev wp eval "while (time() < $GO) { usleep(2000); } PC\\Queue_Service::sync_turn($ROOM);" >/dev/null 2>&1 &
done
wait

ddev wp eval "global \$wpdb; echo 'Open turns after 12 simultaneous requests: ' . (int) \$wpdb->get_var(\"SELECT COUNT(*) FROM {\$wpdb->prefix}pc_bet_sessions WHERE room_id = $ROOM AND ended_at IS NULL\") . \"\n\";" 2>&1 | grep -v Deprecated
```

**Expected:**

```
Open turns after 12 simultaneous requests: 1
```

**Anything above 1 is a failure.** For scale: the same twelve processes run against the
pre-step logic opened **twelve** turns for that one room.

Now clean up (note the room number printed by the block above is reused automatically):

```bash
ddev wp eval "
global \$wpdb;
\$wpdb->query( \"DELETE FROM {\$wpdb->prefix}pc_bet_sessions WHERE room_id = $ROOM\" );
\$wpdb->query( \"DELETE FROM {\$wpdb->prefix}pc_room_queues WHERE room_id = $ROOM\" );
wp_delete_post( $ROOM, true );
\$u = get_user_by( 'login', 'race-probe' );
if ( \$u ) { require_once ABSPATH.'wp-admin/includes/user.php'; wp_delete_user( \$u->ID ); }
echo \"cleaned up\n\";" 2>&1 | grep -v Deprecated
```

**Expected:** `cleaned up`.

---

## 5. A room still hands over, and is not wedged shut

The risk of a rule like this is the opposite failure: a room that can never open a
*second* turn even when the first one properly ends. The automated checks cover it, and
you can see it for yourself in the browser.

1. Start the player app (it runs separately from the WordPress site) and open it:

   ```bash
   cd ~/Projects/full-stack/pusher-coin/frontend
   npm run dev
   ```

   then open `http://localhost:5173`.
2. Sign in as a player who has coins, open a room, and **join the queue** declaring 2
   coins.
3. In a terminal, confirm the room has exactly one turn:

   ```bash
   cd ~/Projects/full-stack/pusher-coin/backend && ddev wp pc queue-sessions
   ```

   **Expected:** one row for that room, **protected** `yes`, **duplicate** `no`.
4. Back in the browser, **leave the queue**.
5. Run the command again. **Expected:** that room no longer appears — its turn closed.
6. **Join the queue again** and run the command once more. **Expected:** the room appears
   again, with a *different* session id. This is the proof the room was not locked
   against its own next turn.
7. Leave the queue and stop the dev server.

---

## 6. If a room ever does show a duplicate

On a database that was running before this step, a room may still be carrying a spare
turn from the old code. The report flags it:

```bash
ddev wp pc queue-sessions
```

A flagged room prints a **Warning** naming the room, how many turns it holds, and which
one is the real one — the turn the queue is pointing at. To fix it:

```bash
ddev wp pc queue-sessions --close
```

**Expected:** a line like `Closed 1 session(s) across 1 room(s). Each one is in the audit
log as queue_session_orphan_closed.`, then a clean report.

Run it a second time. **Expected:** `Nothing to close.` — it is safe to repeat.

Nothing is deleted by this. The spare turn keeps its row, its coins played, its coins won
and its money won; it is only marked as ended. That is deliberate: a spare turn may have
collected a real payout, and the record of it has to survive.

---

## 7. A real win lands on the turn the player is looking at

This is the half of the bug that cost money, so it is worth seeing directly. No machine
is involved — the win is injected the same way a real payout arrives.

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp eval-file wp-content/themes/pc/tests/queue-sessions.php
```

**Expected:** `Success: All 54 checks passed.` — and, just above it, a `Cleanup — rows
before / after:` block in which every one of the eight numbers is **identical** on both
sides, ending with `PASS  cleanup: every table is back where it started`.

Among those 54, these five are the money ones and are worth reading by name in the
output:

```
PASS  money: the win is banked on the room's open session
PASS  money: which is the session the queue head points at
PASS  money: and the room screen shows it as this turn's winnings
PASS  money: on the same session id the player is looking at
PASS  money: coins played are booked on that same row
```

---

## 8. The whole backend suite

```bash
cd ~/Projects/full-stack/pusher-coin/backend
bin/check
```

**Expected:** ends with a `== Summary` block showing `php -l: 68 files OK` and
`DDEV checks: passed —` followed by eleven script names, `queue-sessions.php` among them.
Exit code 0.

If the summary instead shows a boxed **`SKIPPED: DDEV checks did not run`** notice, DDEV
is not running and none of the money checks executed — a green run that shows that box
has verified nothing.

---

## 9. On the real server, after the next deploy

The cleanup and the database rule apply themselves on the first request after the code
reaches the host — there is nothing to run by hand. Once `main` has been deployed, and if
the host has WP-CLI:

```bash
wp pc queue-sessions
```

**Expected:** the same last line as in section 2 — `UNIQUE KEY open_room: present`. Any
room flagged as a duplicate there is a real one from before the fix; `--close` settles
it, and each closure is in the audit log.

**If the host has no WP-CLI**, there is nothing to check by hand and nothing to worry
about: the same code runs on the first page load either way. The only way to see the
result would be the database directly.

---

## What must NOT happen, in one list

- Section 3 must never print `YES - THE RULE IS NOT BEING KEPT`.
- Section 4 must never print a number above `1`.
- Section 2 must never print the **Warning** that `UNIQUE KEY open_room` is missing.
- Section 5 step 6 must never fail to open a new turn — a room that cannot start its next
  turn is this change failing in the opposite direction.
- `--close` must never reduce the number of rows in the turns table. It ends turns; it
  deletes nothing.

---

## What this step did *not* touch

- **The player app and the operator app are unchanged.** No screen, no button, no text.
- **Nothing to do with the machine, the relay or Home Assistant.**
- **Nothing to do with Ably or the live channel** — this step needs no account and no key.
- **The guest chat poll is unchanged** (still every 3 seconds, on purpose).
