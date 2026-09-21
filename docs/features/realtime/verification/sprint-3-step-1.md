# Verification — `realtime` Sprint 3, Step 1: The machine is unreachable during a broadcast window

**What this step built.** Until now, if the machine stopped answering during a
broadcast, nobody found out until a player wrote into support. The system now notices
and emails the operator.

**The whole difficulty is one thing.** The machine is switched off by hand at the venue
every single day. "Not answering" is therefore the *normal* state most of the time, and
an alert that fired on it would fire every evening and be ignored inside a week. So the
room's own broadcast schedule is the gate:

- **Not answering while the room is broadcasting** → one email, once.
- **Not answering outside broadcast hours** → written into the log, nobody emailed.
- **A brief hiccup** → nothing at all. The machine has to stay silent for five minutes
  (a setting) before it counts.
- **It comes back** → the recovery is recorded, and the next fault can raise a fresh
  alert.

**Where alerts go.** `pc_realtime_alert_email` if set, otherwise the support address,
otherwise the site admin's. You can point it at whatever you actually read.

**Time:** about 20 minutes. **No Ably account is needed.** **Nothing here touches the
venue, the real machine, or Home Assistant** — every step points the software at a dead
local address instead.

> **About the Home Assistant token.** This guide never asks for it and you must never
> type it into a command. Section 4 uses the literal word
> `placeholder-not-a-real-token`, exactly as written. Nothing here can reach Home
> Assistant anyway: the address is pointed at a dead local port first.

---

## 1. Get the code and start the local site

```bash
cd ~/Projects/full-stack/pusher-coin/backend
git checkout realtime/sprint-3
ddev start
```

If `ddev start` reports that it changed `wp-config-ddev.php`, that is expected and
harmless — leave it.

Confirm the new settings arrived (they apply themselves on the first page load):

```bash
ddev wp eval 'echo get_option("pc_db_version") . "\n";'
```

**Expected:** `1.14.0` or higher. If you see `1.13.0`, open
`https://pusher-coin.ddev.site` once in a browser and run it again.

---

## 2. Where would an alert go?

```bash
ddev wp eval 'echo PC\Realtime_Alerts::destination() . "\n";'
```

**Expected:** an email address — yours, since `pc_realtime_alert_email` is empty on a
fresh install and it falls back to the support address and then to the site admin's.
That fallback is the point: an install nobody has configured still reaches somebody.

Local mail does not leave your machine. DDEV catches it, and you read it at
**`https://pusher-coin.ddev.site:8026`** (or `ddev mailpit`). Open that page now and
leave it open — it should be empty.

---

## 3. An install nobody has connected does not cry wolf

Before setting anything up, check the starting state:

```bash
ddev wp pc machine-poll
```

**Expected:** among the output,

```
machine: not watched — no `pc_realtime_poll_machine_id`, or Home Assistant is not configured
```

**and no email in Mailpit.** This local site has no machine token, so as far as the
software is concerned no machine has ever been connected — and an install in that state
must never alert. **This is a real negative check, not a formality:** the opposite
behaviour would email you once a minute forever on any install that has not been set up
yet.

---

## 4. Set up a room that is broadcasting right now

Nothing in the project creates a room whose broadcast window covers *this moment*, so
this block makes one, points the software at it, and aims the machine address at a dead
local port. It also sets the five-minute wait to zero so you do not have to wait.

```bash
ddev wp eval '
global $wpdb;
$room = wp_insert_post( [ "post_type" => "pc_room", "post_title" => "Outage check room", "post_status" => "publish" ] );
update_post_meta( $room, PC\Post_Meta_Keys::ROOM_STATUS, "available" );
update_post_meta( $room, PC\Post_Meta_Keys::ROOM_MACHINE_ID, "outage-check" );
$today = (int) current_datetime()->format( "N" ) - 1;
$wpdb->insert( $wpdb->prefix . "pc_room_schedules", [
  "room_id" => $room, "weekday" => $today, "start_time" => "00:00:01", "end_time" => "23:59:59",
  "recurrence" => "always", "once_date" => null, "created_at" => current_time( "mysql" ),
] );
update_option( "pc_realtime_poll_machine_id", "outage-check" );
update_option( "pc_realtime_outage_grace_seconds", 0 );
update_option( "pc_machine_endpoint", "https://127.0.0.1:9/api" );
delete_option( "pc_realtime_outage_state" );
echo "Room #$room is broadcasting all day today, and the machine points at a dead address.\n";'
```

**Expected:** `Room #… is broadcasting all day today, and the machine points at a dead
address.`

---

## 5. The alert

```bash
ddev wp eval '
define( "PC_MACHINE_TOKEN", "placeholder-not-a-real-token" );
$r = PC\Realtime_Outage_Watch::run();
echo "reachable: " . var_export( $r["reachable"], true ) . "\n";
echo "incident:  " . var_export( $r["incident"], true ) . "\n";
echo "in_window: " . var_export( $r["in_window"], true ) . "\n";
echo "notified:  " . var_export( $r["notified"], true ) . "\n";'
```

**Expected, exactly:**

```
reachable: false
incident:  true
in_window: true
notified:  true
```

Now refresh **`https://pusher-coin.ddev.site:8026`**. **Expected:** one message,
subject `[Pusher Coin] Machine unreachable during a broadcast — Outage check room`.
Open it. The body must name the room, the machine, how long it has been down, when the
broadcast window ends, and what to check at the venue — everything needed to act without
opening a database:

```
The machine has not answered for 0 minutes, and "Outage check room" is broadcasting right now.

Room:            Outage check room (#…)
Machine id:      outage-check
Unreachable for: 0 minutes
Window ends:     …
Last error:      …

Players in that room cannot toss a coin. Check that the machine is switched on at the venue and that Home Assistant is reachable.
This is the only message you will get about this outage; a recovery is recorded in the audit log.
```

> It says **0 minutes** because section 4 set the five-minute wait to zero so you would
> not have to sit through it. In real use the first alert says 5 minutes or more.

Run the same command four more times. **Expected: still exactly one message in Mailpit.**
One incident, one email — not one per failed check.

---

## 6. The negative check — the nightly power-off must be silent

This is the check that matters most. Same dead address, same failures; the only change
is that the room is **not** broadcasting.

```bash
ddev wp eval '
global $wpdb;
define( "PC_MACHINE_TOKEN", "placeholder-not-a-real-token" );
$room  = (int) $wpdb->get_var( "SELECT post_id FROM {$wpdb->postmeta} WHERE meta_key=\"pc_room_machine_id\" AND meta_value=\"outage-check\" LIMIT 1" );
$other = ( (int) current_datetime()->format( "N" ) - 1 + 3 ) % 7;
$wpdb->update( $wpdb->prefix . "pc_room_schedules", [ "weekday" => $other ], [ "room_id" => $room ], [ "%d" ], [ "%d" ] );
delete_option( "pc_realtime_outage_state" );
$floor = (int) $wpdb->get_var( "SELECT COALESCE(MAX(id),0) FROM {$wpdb->prefix}pc_auth_audit_log" );
for ( $i = 0; $i < 5; $i++ ) { $r = PC\Realtime_Outage_Watch::run(); }
echo "incident:  " . var_export( $r["incident"], true ) . "\n";
echo "in_window: " . var_export( $r["in_window"], true ) . "\n";
echo "notified:  " . var_export( $r["notified"], true ) . "\n";
foreach ( [ "machine_outage_started", "machine_outage_notified", "operator_alert_sent" ] as $t ) {
  echo sprintf( "%-26s %d\n", $t, (int) $wpdb->get_var( $wpdb->prepare( "SELECT COUNT(*) FROM {$wpdb->prefix}pc_auth_audit_log WHERE id > %d AND event_type = %s", $floor, $t ) ) );
}'
```

**Expected, exactly:**

```
incident:  true
in_window: false
notified:  false
machine_outage_started     1
machine_outage_notified    0
operator_alert_sent        0
```

And **Mailpit must still show exactly one message** — the one from section 5. **Not two.**

Read that as: the machine being off was noticed and written down once, however many
times it was checked, and nobody was disturbed about it. **If a second email appears
here, the step has failed** — that is the every-evening alarm the whole design exists to
avoid.

---

## 7. It comes back

```bash
ddev wp eval '
global $wpdb;
define( "PC_MACHINE_TOKEN", "placeholder-not-a-real-token" );
update_option( "pc_machine_endpoint", "http://localhost" );
$floor = (int) $wpdb->get_var( "SELECT COALESCE(MAX(id),0) FROM {$wpdb->prefix}pc_auth_audit_log" );
$r = PC\Realtime_Outage_Watch::run();
echo "reachable: " . var_export( $r["reachable"], true ) . "\n";
echo "state cleared: " . var_export( false === get_option( "pc_realtime_outage_state" ), true ) . "\n";
$row = $wpdb->get_row( $wpdb->prepare( "SELECT metadata FROM {$wpdb->prefix}pc_auth_audit_log WHERE id > %d AND event_type = %s ORDER BY id DESC LIMIT 1", $floor, "machine_outage_recovered" ), ARRAY_A );
echo "recovered: " . ( $row ? $row["metadata"] : "NONE" ) . "\n";'
```

(`http://localhost` is just an address that answers, standing in for a machine that is
back.)

**Expected:**

```
reachable: true
state cleared: true
recovered: {"down_since":"…","incident_at":"…","duration_seconds":…,"failures":…,"notified":false}
```

The outage now has an **end**, with how long it lasted and whether anyone was told.
`notified: false` is correct here — that particular outage happened outside broadcast
hours, so nobody was emailed about it.

---

## 8. Put the local site back

```bash
ddev wp eval '
global $wpdb;
$room = (int) $wpdb->get_var( "SELECT post_id FROM {$wpdb->postmeta} WHERE meta_key=\"pc_room_machine_id\" AND meta_value=\"outage-check\" LIMIT 1" );
if ( $room ) {
  $wpdb->delete( $wpdb->prefix . "pc_room_schedules", [ "room_id" => $room ], [ "%d" ] );
  wp_delete_post( $room, true );
}
delete_option( "pc_realtime_outage_state" );
update_option( "pc_realtime_poll_machine_id", "" );
update_option( "pc_realtime_outage_grace_seconds", 300 );
update_option( "pc_machine_endpoint", "" );
echo "torn down.\n";'
```

**Expected:** `torn down.` Then clear Mailpit with the **Delete all** button on
`https://pusher-coin.ddev.site:8026`.

---

## 9. The automated checks

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp eval-file wp-content/themes/pc/tests/realtime-outage.php
```

**Expected:** `Success: All 64 checks passed.` — and, just above it, a
`Cleanup — rows before / after:` block in which all six numbers are **identical** on both
sides.

The ones worth reading by name in that output:

```
PASS  power-off: NOTHING is sent — this is the whole point of the step
PASS  power-off: exactly one outage is recorded, however many passes fail
PASS  handover: the window opens and the same outage alerts
PASS  resilience: a mailer that THROWS does not escape the watch
```

The third one covers a case the manual walkthrough above does not: a machine that goes
down *outside* broadcast hours and is still down when the window opens gets its alert
then — at the moment it starts costing you something.

---

## 10. The whole backend suite

```bash
cd ~/Projects/full-stack/pusher-coin/backend
bin/check
```

**Expected:** a `== Summary` block showing `php -l: 71 files OK` and `DDEV checks:
passed —` followed by twelve script names, `realtime-outage.php` among them. Exit 0.

If the summary instead shows a boxed **`SKIPPED: DDEV checks did not run`** notice, DDEV
is not running and none of the checks executed.

---

## 11. On the real server, after the next deploy

Two things to do once `main` is deployed, both in the admin or over WP-CLI:

1. **Set where alerts go.** `pc_realtime_alert_email` — point it at an address you
   actually read. Left empty, alerts go to the support address, which is the inbox this
   whole sprint exists to stop you depending on.
2. **Confirm mail actually leaves the host.** This is the one thing that cannot be
   checked locally — DDEV catches mail before the last hop, and the production server's
   own mail path has never been confirmed, including for the support-ticket
   notifications that already rely on it. The cheapest proof is to **file one test
   support ticket** through the site and see whether it arrives.

If mail does not leave the host, alerts are recorded but never delivered, and the next
step is a mail plugin or a different channel — the sender is one function, so that is a
small change.

---

## What must NOT happen, in one list

- **Section 6 must never produce a second email.** That is the nightly power-off, and an
  alert on it is the failure this whole design exists to prevent.
- Section 3 must never send anything — an install with no machine connected stays quiet.
- Section 5 repeated must never send a second email for the same outage.
- Nothing in this guide may contact the venue, the real machine, or Home Assistant.
- The real Home Assistant token must never appear in any command here.

---

## What this step did *not* touch

- **The player app and the operator app are unchanged.** No screen, no button, no text —
  the sprint deliberately builds no operator screen.
- **Nothing to do with Ably, the live channel, the queue, chat or money.**
- **Nothing switches the machine on or off.** Power is a hand at the venue.
