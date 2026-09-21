# Verification — `realtime` Sprint 3, Step 2: A toss that moved nothing

**What this step promises.** When a player tosses a coin, Home Assistant answers "OK"
to the *button*, not to the machine acting on it. If the machine swallows the toss —
the player pays a coin and nothing goes in — nothing on our side used to know. Now the
system checks afterwards and writes down the ones the machine did not act on, with
enough detail to answer a player's complaint with the machine's own evidence.

**It writes a record. It does not send anything.** No email, no screen, no alert. That
is what the sprint step asks for; the sender built in Step 1 is one line away if you
later want one.

**Time:** about 15 minutes. **Where:** your own machine, with DDEV running. Nothing here
touches the venue, the real machine or Home Assistant — every "machine" in this guide is
a stand-in that only exists for the length of one command. Nothing asks you for the
Home Assistant token, and you should never type it anywhere.

Run everything from the `backend/` folder:

```
cd ~/Projects/full-stack/pusher-coin/backend
ddev start          # only if it is not already running
```

---

## 1. The automated checks pass

```
bin/check
```

**Expect** the last three lines to read:

```
== Summary
php -l:      73 files OK
DDEV checks: passed — machine-ingest.php machine-poll.php machine-rooms.php queue-sessions.php realtime-channel.php realtime-chat.php realtime-outage.php realtime-queue.php realtime-relay.php realtime-toss.php stripe-client.php stripe-webhook.php wallet-rollback.php
```

`realtime-toss.php` is this step's own script — 60 checks. To watch just those:

```
ddev wp eval-file wp-content/themes/pc/tests/realtime-toss.php
```

**Expect** it to end with `Success: All 60 checks passed.` and a cleanup table whose
"before" and "after" columns are identical on every row.

---

## 2. The watch is on the minute-by-minute pass, and says so

```
ddev wp pc machine-poll --status
```

**Expect** a `tosses:` line among the others:

```
relay: not watched — `pc_realtime_poll_machine_id` is not set
machine: not watched — no `pc_realtime_poll_machine_id`, or Home Assistant is not configured
tosses: not judged — Home Assistant is not configured
```

**This is also a negative check, and a real one.** This computer has no Home Assistant
token, so the watch correctly refuses to judge anything rather than reporting every
toss as a fault on an install nobody has connected to a machine. There is no new
scheduled job — the line appears on the pass that has been running since Sprint 1.

---

## 3. Make a toss the machine did not act on

Paste this whole block. It invents a machine that answers, tells it the coin counter
has been sitting at **5** and never moved, writes a toss as the Room screen writes one,
and runs the watch. Nothing leaves your computer: the "machine" is a made-up address
that only exists inside this one command.

```
ddev wp eval '
define( "PC_MACHINE_TOKEN", "verification-guide" );
add_filter( "pre_option_pc_machine_endpoint", function () { return "https://home-assistant.invalid/api"; } );
add_filter( "pre_http_request", function ( $pre, $args, $url ) {
	if ( false === strpos( $url, "home-assistant.invalid" ) ) { return $pre; }
	if ( false === strpos( $url, "/history/period/" ) ) {
		return [ "headers" => [], "body" => "{}", "response" => [ "code" => 200, "message" => "OK" ] ];
	}
	$rows = [ [ "state" => "5", "last_updated" => gmdate( "Y-m-d\TH:i:s", (int) get_option( "pc_guide_toss_at" ) - 10 ) . "+00:00" ] ];
	return [ "headers" => [], "body" => wp_json_encode( [ $rows ] ), "response" => [ "code" => 200, "message" => "OK" ] ];
}, 10, 3 );

global $wpdb;
$table   = $wpdb->prefix . "pc_machine_events";
$room    = (int) $wpdb->get_var( $wpdb->prepare( "SELECT ID FROM {$wpdb->posts} WHERE post_type = %s AND post_status = %s ORDER BY ID ASC LIMIT 1", "pc_room", "publish" ) );
$session = (int) $wpdb->get_var( "SELECT id FROM {$wpdb->prefix}pc_bet_sessions ORDER BY id DESC LIMIT 1" );
$at      = time() - 60;
update_option( "pc_guide_toss_at", $at, false );

$toss = PC\Machine_Ingest_Service::log_event(
	PC\Machine_Event_Log::TYPE_TOSS,
	[ "machine_id" => (string) get_post_meta( $room, "pc_room_machine_id", true ), "user_id" => 1 ],
	[ "room_id" => $room, "session_id" => $session ]
);
$wpdb->update( $table, [ "created_at" => gmdate( "Y-m-d H:i:s", $at ) . ".000000" ], [ "id" => $toss["event_id"] ] );
update_option( "pc_realtime_toss_cursor", gmdate( "Y-m-d H:i:s", $at - 1 ) . ".000000", false );

echo "A toss by player #1 in room " . get_the_title( $room ) . " was recorded 60 seconds ago.\n";
echo "The coin counter read 5 at that moment, and nothing has happened since.\n\n";

$out = PC\Realtime_Toss_Watch::run();
echo "judged: " . $out["judged"] . ", moved: " . $out["moved"] . ", unconfirmed: " . $out["unconfirmed"] . ", recorded: " . $out["recorded"] . "\n";
'
```

**Expect** (the room name is whichever room your install lists first):

```
A toss by player #1 in room Sunset Pusher was recorded 60 seconds ago.
The coin counter read 5 at that moment, and nothing has happened since.

judged: 1, moved: 0, unconfirmed: 0, recorded: 1
```

`recorded: 1` is the finding: the machine should have reset its counter to zero within
about two seconds of accepting that toss, and it did not.

---

## 4. Read the record the way you would in a dispute

```
ddev wp eval '
global $wpdb;
$row = $wpdb->get_row( $wpdb->prepare(
	"SELECT * FROM {$wpdb->prefix}pc_machine_events WHERE event_type = %s ORDER BY id DESC LIMIT 1",
	"toss_no_movement"
), ARRAY_A );

if ( ! $row ) { echo "No record found.\n"; return; }

$p    = json_decode( (string) $row["payload"], true );
$user = get_userdata( (int) $row["user_id"] );

echo "Record #" . $row["id"] . " — a toss that moved nothing\n";
echo "  written at:     " . $row["created_at"] . " (UTC)\n";
echo "  player:         #" . $row["user_id"] . " (" . ( $user ? $user->user_login : "unknown" ) . ")\n";
echo "  turn (session): #" . $row["correlation_id"] . "\n";
echo "  room:           " . get_the_title( (int) $p["room_id"] ) . " (#" . $p["room_id"] . ")\n";
echo "  the toss:       event #" . $p["toss_event_id"] . " at " . $p["toss_at"] . "\n";
echo "  machine:        " . $row["machine_id"] . ", sensor " . $p["coin_sensor"] . "\n";
echo "  counter before: " . $p["before"] . "\n";
echo "  counter after:  " . $p["after"] . " (" . $p["changes"] . " changes in " . $p["window_seconds"] . " seconds)\n";
echo "  coins credited: " . $row["coins_credited"] . "\n";
'
```

**Expect** something shaped like this — your ids and names will differ:

```
Record #2115 — a toss that moved nothing
  written at:     2026-09-21 19:26:39.443430 (UTC)
  player:         #1 (tymofii.synianskyi)
  turn (session): #521
  room:           Sunset Pusher (#11)
  the toss:       event #2114 at 2026-09-21 19:25:39.000000
  machine:        demo_sunset, sensor sensor.coin
  counter before: 5
  counter after:  5 (0 changes in 30 seconds)
  coins credited: 0
```

**What to look for:** every line a support answer needs is on the one record — who was
playing, which turn, which toss, which machine, and what the machine's own counter read
on both sides of the window. `coins credited: 0` says plainly that this record moves no
money; it is evidence, not a refund.

---

## 5. NEGATIVE CHECK — a toss the machine did act on produces nothing

The counter reads 5, and one second after the toss the machine resets it to 0. That
reset **is** the machine confirming it acted.

```
ddev wp eval '
define( "PC_MACHINE_TOKEN", "verification-guide" );
add_filter( "pre_option_pc_machine_endpoint", function () { return "https://home-assistant.invalid/api"; } );
add_filter( "pre_http_request", function ( $pre, $args, $url ) {
	if ( false === strpos( $url, "home-assistant.invalid" ) ) { return $pre; }
	if ( false === strpos( $url, "/history/period/" ) ) {
		return [ "headers" => [], "body" => "{}", "response" => [ "code" => 200, "message" => "OK" ] ];
	}
	$at   = (int) get_option( "pc_guide_toss_at" );
	$rows = [
		[ "state" => "5", "last_updated" => gmdate( "Y-m-d\TH:i:s", $at - 10 ) . "+00:00" ],
		[ "state" => "0", "last_updated" => gmdate( "Y-m-d\TH:i:s", $at + 1 ) . "+00:00" ],
	];
	return [ "headers" => [], "body" => wp_json_encode( [ $rows ] ), "response" => [ "code" => 200, "message" => "OK" ] ];
}, 10, 3 );

global $wpdb;
$table   = $wpdb->prefix . "pc_machine_events";
$records = function () use ( $wpdb, $table ) {
	return (int) $wpdb->get_var( $wpdb->prepare( "SELECT COUNT(*) FROM $table WHERE event_type = %s", "toss_no_movement" ) );
};
$room = (int) $wpdb->get_var( $wpdb->prepare( "SELECT ID FROM {$wpdb->posts} WHERE post_type = %s AND post_status = %s ORDER BY ID ASC LIMIT 1", "pc_room", "publish" ) );
$at   = time() - 60;
update_option( "pc_guide_toss_at", $at, false );

$toss = PC\Machine_Ingest_Service::log_event( PC\Machine_Event_Log::TYPE_TOSS, [ "machine_id" => (string) get_post_meta( $room, "pc_room_machine_id", true ), "user_id" => 1 ], [ "room_id" => $room ] );
$wpdb->update( $table, [ "created_at" => gmdate( "Y-m-d H:i:s", $at ) . ".000000" ], [ "id" => $toss["event_id"] ] );
update_option( "pc_realtime_toss_cursor", gmdate( "Y-m-d H:i:s", $at - 1 ) . ".000000", false );

$before = $records();
$out    = PC\Realtime_Toss_Watch::run();
echo "The counter read 5 before the toss and was reset to 0 one second after it.\n";
echo "judged: " . $out["judged"] . ", moved: " . $out["moved"] . ", recorded: " . $out["recorded"] . "\n";
echo "records on file before: " . $before . ", after: " . $records() . "\n";
'
```

**Expect:**

```
The counter read 5 before the toss and was reset to 0 one second after it.
judged: 1, moved: 1, recorded: 0
records on file before: 1, after: 1
```

**`recorded: 0`, and the number of records on file did not change.** A normal toss must
never produce a record — including a toss that paid nothing, which is most tosses.

---

## 6. NEGATIVE CHECK — a toss that cannot be judged is not called a fault

This is the subtle one, and it is why the record means anything at all. When the counter
is **already at zero**, the reset a toss causes writes a zero over a zero — and Home
Assistant only records *changes*, so it keeps no trace whatsoever. Nothing can be
concluded, so nothing is written down.

```
ddev wp eval '
define( "PC_MACHINE_TOKEN", "verification-guide" );
add_filter( "pre_option_pc_machine_endpoint", function () { return "https://home-assistant.invalid/api"; } );
add_filter( "pre_http_request", function ( $pre, $args, $url ) {
	if ( false === strpos( $url, "home-assistant.invalid" ) ) { return $pre; }
	if ( false === strpos( $url, "/history/period/" ) ) {
		return [ "headers" => [], "body" => "{}", "response" => [ "code" => 200, "message" => "OK" ] ];
	}
	$rows = [ [ "state" => "0", "last_updated" => gmdate( "Y-m-d\TH:i:s", (int) get_option( "pc_guide_toss_at" ) - 10 ) . "+00:00" ] ];
	return [ "headers" => [], "body" => wp_json_encode( [ $rows ] ), "response" => [ "code" => 200, "message" => "OK" ] ];
}, 10, 3 );

global $wpdb;
$table   = $wpdb->prefix . "pc_machine_events";
$records = function () use ( $wpdb, $table ) {
	return (int) $wpdb->get_var( $wpdb->prepare( "SELECT COUNT(*) FROM $table WHERE event_type = %s", "toss_no_movement" ) );
};
$room = (int) $wpdb->get_var( $wpdb->prepare( "SELECT ID FROM {$wpdb->posts} WHERE post_type = %s AND post_status = %s ORDER BY ID ASC LIMIT 1", "pc_room", "publish" ) );
$at   = time() - 60;
update_option( "pc_guide_toss_at", $at, false );

$toss = PC\Machine_Ingest_Service::log_event( PC\Machine_Event_Log::TYPE_TOSS, [ "machine_id" => (string) get_post_meta( $room, "pc_room_machine_id", true ), "user_id" => 1 ], [ "room_id" => $room ] );
$wpdb->update( $table, [ "created_at" => gmdate( "Y-m-d H:i:s", $at ) . ".000000" ], [ "id" => $toss["event_id"] ] );
update_option( "pc_realtime_toss_cursor", gmdate( "Y-m-d H:i:s", $at - 1 ) . ".000000", false );

$before = $records();
$out    = PC\Realtime_Toss_Watch::run();
echo "The counter read 0 before the toss and nothing happened after it.\n";
echo "judged: " . $out["judged"] . ", unconfirmed: " . $out["unconfirmed"] . ", recorded: " . $out["recorded"] . "\n";
echo "records on file before: " . $before . ", after: " . $records() . "\n";
'
```

**Expect:**

```
The counter read 0 before the toss and nothing happened after it.
judged: 1, unconfirmed: 1, recorded: 0
records on file before: 1, after: 1
```

**`unconfirmed: 1` and no new record.** The system says "I cannot tell" instead of
accusing the machine. Those are counted and reported on `wp pc machine-poll`, so they
are never hidden from you — but they never become evidence against a machine that may
have worked perfectly.

---

## 7. One record per toss — even if the bookkeeping is wound back

```
ddev wp eval '
define( "PC_MACHINE_TOKEN", "verification-guide" );
add_filter( "pre_option_pc_machine_endpoint", function () { return "https://home-assistant.invalid/api"; } );
add_filter( "pre_http_request", function ( $pre, $args, $url ) {
	if ( false === strpos( $url, "home-assistant.invalid" ) ) { return $pre; }
	if ( false === strpos( $url, "/history/period/" ) ) {
		return [ "headers" => [], "body" => "{}", "response" => [ "code" => 200, "message" => "OK" ] ];
	}
	$rows = [ [ "state" => "5", "last_updated" => gmdate( "Y-m-d\TH:i:s", (int) get_option( "pc_guide_toss_at" ) - 10 ) . "+00:00" ] ];
	return [ "headers" => [], "body" => wp_json_encode( [ $rows ] ), "response" => [ "code" => 200, "message" => "OK" ] ];
}, 10, 3 );

global $wpdb;
$table = $wpdb->prefix . "pc_machine_events";
$room  = (int) $wpdb->get_var( $wpdb->prepare( "SELECT ID FROM {$wpdb->posts} WHERE post_type = %s AND post_status = %s ORDER BY ID ASC LIMIT 1", "pc_room", "publish" ) );
$at    = time() - 60;
update_option( "pc_guide_toss_at", $at, false );

$toss = PC\Machine_Ingest_Service::log_event( PC\Machine_Event_Log::TYPE_TOSS, [ "machine_id" => (string) get_post_meta( $room, "pc_room_machine_id", true ), "user_id" => 1 ], [ "room_id" => $room ] );
$wpdb->update( $table, [ "created_at" => gmdate( "Y-m-d H:i:s", $at ) . ".000000" ], [ "id" => $toss["event_id"] ] );
$rewind = function () use ( $at ) { update_option( "pc_realtime_toss_cursor", gmdate( "Y-m-d H:i:s", $at - 1 ) . ".000000", false ); };
$mine   = function () use ( $wpdb, $table, $toss ) {
	return (int) $wpdb->get_var( $wpdb->prepare( "SELECT COUNT(*) FROM $table WHERE event_key = %s", "toss:" . $toss["event_id"] . ":no-movement" ) );
};

$rewind();
$out = PC\Realtime_Toss_Watch::run();
echo "first pass:  judged " . $out["judged"] . ", recorded " . $out["recorded"] . " -> records for this toss: " . $mine() . "\n";
$out = PC\Realtime_Toss_Watch::run();
echo "second pass: judged " . $out["judged"] . ", recorded " . $out["recorded"] . " -> records for this toss: " . $mine() . "\n";
$rewind();
$out = PC\Realtime_Toss_Watch::run();
echo "third pass with the cursor wound back: judged " . $out["judged"] . ", recorded " . $out["recorded"] . " -> records for this toss: " . $mine() . "\n";
'
```

**Expect:**

```
first pass:  judged 1, recorded 1 -> records for this toss: 1
second pass: judged 0, recorded 0 -> records for this toss: 1
third pass with the cursor wound back: judged 1, recorded 1 -> records for this toss: 1
```

The second pass skips a toss it has already judged. The third is deliberately made to
judge it again — and the database still holds **one** record. One toss can never produce
two records, however many times a pass runs or a backup is restored.

---

## 8. The waiting time is a setting, not something baked in

The machine is given 30 seconds by default. Change it to 5 and a 10-second-old toss
becomes judgeable at once:

```
ddev wp option update pc_realtime_toss_window_seconds 5
```

Then run the block from section 3 again, but change the one line that says
`$at = time() - 60;` to `$at = time() - 10;`.

**Expect** `judged: 1 … recorded: 1` — under the default 30-second setting the same
10-second-old toss would have been left alone until its window closed. Put it back:

```
ddev wp option update pc_realtime_toss_window_seconds 30
```

---

## 9. Clean up

Removes everything this guide created and leaves the install exactly as it was.

```
ddev wp eval '
global $wpdb;
$table = $wpdb->prefix . "pc_machine_events";
$since = gmdate( "Y-m-d H:i:s", time() - HOUR_IN_SECONDS );

$before = (int) $wpdb->get_var( "SELECT COUNT(*) FROM $table" );

$removed  = (int) $wpdb->query( $wpdb->prepare( "DELETE FROM $table WHERE event_type = %s", "toss_no_movement" ) );
$removed += (int) $wpdb->query( $wpdb->prepare( "DELETE FROM $table WHERE event_type = %s AND user_id = 1 AND created_at >= %s", "toss", $since ) );

delete_option( "pc_guide_toss_at" );
delete_option( "pc_realtime_toss_cursor" );

echo "removed " . $removed . " row(s) the guide created.\n";
echo "machine events before: " . $before . ", after: " . (int) $wpdb->get_var( "SELECT COUNT(*) FROM $table" ) . "\n";
echo "toss cursor: " . ( get_option( "pc_realtime_toss_cursor" ) ? "still set" : "cleared" ) . "\n";
'
```

**Expect** the "after" count to match what your install had before you started (3 on a
freshly seeded one), and `toss cursor: cleared`.

Then confirm the settings are where they should be:

```
ddev wp option get pc_realtime_toss_window_seconds     # 30
ddev wp option get pc_realtime_toss_max_age_seconds    # 604800
ddev wp option get pc_db_version                       # 1.15.0
```

---

## 10. At the venue — the one check this guide cannot do

Everything above uses a stand-in machine. **No real toss has ever been timed against
this code.** The step's own verification names the check that settles it, and it can
only be done at the venue, during opening hours, with coins in the machine:

1. Make one ordinary toss from the player app while the machine is switched on.
2. Wait two minutes (one poll pass plus the window).
3. Run `wp pc machine-poll --status` on the production host. The `tosses:` line must
   report the toss as **moved** or **unconfirmed** — never as recorded.

A real toss that produces a record is the one result that would mean the 30-second
window is too short for this machine, and the fix is `wp option update
pc_realtime_toss_window_seconds 60` on the host — no deploy, no code change.

**Also still true:** the production host has never had a `PC_MACHINE_TOKEN` or a
`pc_realtime_poll_machine_id` set, so until an operator sets both, every pass there
reports `tosses: not judged — Home Assistant is not configured`, exactly as section 2
shows locally.
