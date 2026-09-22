# Verification — `realtime` Sprint 3, Step 3: Withdrawals piling up

Written for someone who does not read code. Every command below is meant to
be **copied whole** into a terminal that is inside the `backend/` folder of
this project, with DDEV already running (`ddev start`). Every block was run
exactly as written while this guide was being made, and the output shown
under each one is the real output from that run.

Nothing here touches the machine at the venue, switches anything on or off,
or sends an email to a real person: DDEV catches all mail in a local inbox.

---

## 0. What this step added, in one paragraph

A player can ask for their coins back as money. The coins leave their wallet
the moment they ask, and the request then sits in a queue until somebody at
the venue actually pays them and marks it approved. **Nothing in the system
used to complain when that queue grew.** No error, no warning, nothing on a
screen — just people waiting, until one of them wrote to support. This step
adds a watcher: when too many requests are waiting, or one has been waiting
too long, the operator gets one email about it.

The two numbers, and a third that controls how often it may speak, are
settings — not fixed in the code.

---

## 1. The three settings arrived on this install

```bash
ddev wp option get pc_db_version
ddev wp option get pc_realtime_withdrawal_alert_count
ddev wp option get pc_realtime_withdrawal_alert_age_seconds
ddev wp option get pc_realtime_withdrawal_alert_period_seconds
```

**Expected — exactly this:**

```
1.16.0
10
86400
86400
```

In words: more than **10** requests waiting, or an oldest one older than
**86400 seconds** (one day), raises the alarm; and at most **one** withdrawal
email per **86400 seconds** (one day), however the queue moves.

If `pc_db_version` says something lower than `1.16.0`, open any page of the
site once — the settings install themselves on the first page load after the
update — and run the block again.

---

## 2. With nobody waiting, the watch says so out loud

```bash
ddev wp pc machine-poll
```

**Expected — the last of the four watch lines:**

```
withdrawals: nobody is waiting to be paid
```

This line prints on **every** pass, including quiet ones. That is deliberate:
"nobody is waiting" is the answer you want to see, and silence could not be
told apart from "this was never checked at all".

The other three lines above it (`relay:`, `machine:`, `tosses:`) will say
things like "not watched" or "Home Assistant is not configured". That is
correct on this laptop — there is no machine here.

---

## 3. Make some people wait

This creates three practice players, gives each one coin, and has each one
ask for it back — the same path a real player takes. It then remembers them
so section 9 can remove them again.

```bash
ddev wp eval '
$made = [];
for ( $i = 1; $i <= 3; $i++ ) {
	$user = wp_insert_user( [
		"user_login" => "pc-guide-withdrawal-$i",
		"user_email" => "pc-guide-withdrawal-$i@example.invalid",
		"user_pass"  => wp_generate_password( 20, false ),
		"role"       => "subscriber",
	] );
	if ( is_wp_error( $user ) ) { echo "already there: ", $user->get_error_code(), PHP_EOL; continue; }
	PC\Wallet_Service::credit_lot( $user, 1, "40.00" );
	$r = PC\Wallet_Service::request_withdrawal( $user, 1 );
	$made[] = [ "user" => (int) $user, "txn" => (int) $r["txn_id"] ];
}
update_option( "pc_guide_withdrawals", $made, false );
$s = PC\Wallet_Service::pending_withdrawal_summary();
echo "made ", count( $made ), " request(s); the backlog is now ", $s["count"], " waiting, ", $s["total_money"], " UAH", PHP_EOL;
'
```

**Expected:**

```
made 3 request(s); the backlog is now 3 waiting, 120.00 UAH
```

Three waiting is well under the real threshold of ten, so lower the threshold
to two — the same thing an operator would do from a settings screen:

```bash
ddev wp option update pc_realtime_withdrawal_alert_count 2
```

---

## 4. The alarm goes off, once

```bash
ddev wp pc machine-poll
```

**Expected:**

```
withdrawals: 3 waiting to be paid, the oldest for 0 hour(s) — OVER the threshold, and the operator has just been told
```

Now open the local mail inbox in a browser: **https://pusher-coin.ddev.site:8026**

**Expected — one new email**, subject:

```
[Pusher Coin] Withdrawals are piling up — 3 waiting
```

and its text reading (the date will be today's):

```
3 withdrawal request(s) are waiting to be paid, and the oldest has been waiting 0 minutes.

Waiting:      3 request(s)
Total owed:   120.00 UAH
Oldest:       2026-09-22 08:06:41
Threshold:    more than 2 waiting

Nobody is paid until an operator approves these. Open the Withdrawals screen in the admin app, approve the ones you have paid out and reject the rest — a rejected request gives the coins back at the price they were bought at.
This is the only message you will get about this backlog; clearing it is what arms the next one.
```

If your DDEV sends mail somewhere else, the address it went to is whatever
`pc_realtime_alert_email` holds, falling back to the support address and then
to the site administrator's.

### 4a. NEGATIVE CHECK — it must not say it twice

Run the same pass again, two or three times:

```bash
ddev wp pc machine-poll
ddev wp pc machine-poll
```

**Expected — the line changes and NO new email appears in the inbox:**

```
withdrawals: 3 waiting to be paid, the oldest for 0 hour(s) — OVER the threshold; already alerted, or within the quiet period since the last alert
```

Refresh https://pusher-coin.ddev.site:8026 — there must still be **exactly
one** "piling up" email. A backlog that lasts all week is one email, not one
every minute. **If a second email appears here, this step has failed.**

---

## 5. Paying people clears it, and that is written down

```bash
ddev wp eval '
foreach ( (array) get_option( "pc_guide_withdrawals", [] ) as $row ) {
	PC\Wallet_Service::approve_withdrawal( (int) $row["txn"] );
}
echo "approved; the backlog is now ", PC\Wallet_Service::pending_withdrawal_summary()["count"], " waiting", PHP_EOL;
'
ddev wp pc machine-poll
```

**Expected:**

```
approved; the backlog is now 0 waiting
withdrawals: nobody is waiting to be paid
```

The clearing itself is recorded in the operator's audit trail. Read the last
few rows:

```bash
ddev wp eval '
global $wpdb;
$rows = $wpdb->get_results( "SELECT event_type, created_at FROM {$wpdb->prefix}pc_auth_audit_log WHERE event_type LIKE \"withdrawal_backlog%\" OR event_type LIKE \"operator_alert%\" ORDER BY id DESC LIMIT 5", ARRAY_A );
foreach ( $rows as $r ) { echo $r["created_at"], "  ", $r["event_type"], PHP_EOL; }
'
```

**Expected — three rows, newest first:**

```
2026-09-22 08:07:12.248439  withdrawal_backlog_cleared
2026-09-22 08:06:51.092472  withdrawal_backlog_alerted
2026-09-22 08:06:51.091870  operator_alert_sent
```

`withdrawal_backlog_cleared` is the one that matters here: it is the system
saying "that is dealt with", and it is what allows the next alert.

> **Note on the wording:** the pass that clears the backlog prints
> `nobody is waiting to be paid` rather than saying that it just cleared an
> alert. The clearing is in the audit trail, as above. Making that line say
> so is a small cosmetic improvement, noted as a follow-up.

---

## 6. NEGATIVE CHECK — crossing again the same day stays quiet

Make three more people wait:

```bash
ddev wp eval '
$made = [];
for ( $i = 4; $i <= 6; $i++ ) {
	$user = wp_insert_user( [ "user_login" => "pc-guide-withdrawal-$i", "user_email" => "pc-guide-withdrawal-$i@example.invalid", "user_pass" => wp_generate_password( 20, false ), "role" => "subscriber" ] );
	if ( is_wp_error( $user ) ) { continue; }
	PC\Wallet_Service::credit_lot( $user, 1, "40.00" );
	$r = PC\Wallet_Service::request_withdrawal( $user, 1 );
	$made[] = [ "user" => (int) $user, "txn" => (int) $r["txn_id"] ];
}
update_option( "pc_guide_withdrawals", array_merge( (array) get_option( "pc_guide_withdrawals", [] ), $made ), false );
echo "the backlog is ", PC\Wallet_Service::pending_withdrawal_summary()["count"], " waiting again", PHP_EOL;
'
ddev wp pc machine-poll
```

**Expected:**

```
the backlog is 3 waiting again
withdrawals: 3 waiting to be paid, the oldest for 0 hour(s) — OVER the threshold; already alerted, or within the quiet period since the last alert
```

Refresh the inbox: **still exactly one** "piling up" email. The queue was
cleared and crossed again, but an email went out less than a day ago, so
nothing is sent. **This is the check that a wobbling queue cannot flood the
inbox. If a second email appears here, this step has failed.**

---

## 7. And once the day is up, it speaks again

You are not going to wait a day, so move the clock backwards on the
*last alert* instead:

```bash
ddev wp eval '
$state = get_option( "pc_realtime_withdrawal_alert_state" );
$state["last_alert_at"] = gmdate( "c", time() - 2 * DAY_IN_SECONDS );
update_option( "pc_realtime_withdrawal_alert_state", $state, false );
echo "the last alert now looks two days old", PHP_EOL;
'
ddev wp pc machine-poll
```

**Expected:**

```
the last alert now looks two days old
withdrawals: 3 waiting to be paid, the oldest for 0 hour(s) — OVER the threshold, and the operator has just been told
```

The inbox now holds **two** "piling up" emails. The backlog had cleared and
crossed again, and the quiet period had passed — both conditions, which is
what it takes.

---

## 8. The other trigger: one person waiting too long

Switch the count trigger effectively off and set the age trigger to one hour:

```bash
ddev wp option update pc_realtime_withdrawal_alert_count 500
ddev wp option update pc_realtime_withdrawal_alert_age_seconds 3600
ddev wp option delete pc_realtime_withdrawal_alert_state
ddev wp pc machine-poll
```

**Expected — three people waiting is nobody's emergency when they only just
asked:**

```
withdrawals: 3 waiting to be paid, the oldest for 0 hour(s) — under both thresholds
```

Now make the oldest one two days old, the way a forgotten request would be:

```bash
ddev wp eval '
global $wpdb;
$rows = (array) get_option( "pc_guide_withdrawals", [] );
$oldest = end( $rows );
$wpdb->update( $wpdb->prefix . "pc_transactions",
	[ "created_at" => gmdate( "Y-m-d H:i:s", strtotime( current_time( "mysql" ) ) - 2 * DAY_IN_SECONDS ) ],
	[ "id" => (int) $oldest["txn"] ], [ "%s" ], [ "%d" ] );
$s = PC\Wallet_Service::pending_withdrawal_summary();
echo "the oldest request is now ", (int) round( $s["oldest_age_seconds"] / 3600 ), " hours old", PHP_EOL;
'
ddev wp pc machine-poll
```

**Expected:**

```
the oldest request is now 48 hours old
withdrawals: 3 waiting to be paid, the oldest for 48 hour(s) — OVER the threshold, and the operator has just been told
```

And the newest email in the inbox names the *waiting*, not the number:

```
3 withdrawal request(s) are waiting to be paid, and the oldest has been waiting 2 days.

Waiting:      3 request(s)
Total owed:   120.00 UAH
Oldest:       2026-09-20 08:07:57
Threshold:    one waiting longer than 1 hour
```

### 8a. NEGATIVE CHECK — the off switch really is off

```bash
ddev wp option update pc_realtime_withdrawal_alert_count 0
ddev wp option update pc_realtime_withdrawal_alert_age_seconds 0
ddev wp option delete pc_realtime_withdrawal_alert_state
ddev wp pc machine-poll
```

**Expected — and no new email, however many people are waiting:**

```
withdrawals: not watched — both thresholds are 0
```

Setting both numbers to zero switches the watcher off completely. Nothing is
commented out and nothing is deleted; the empty setting *is* the switch.

---

## 9. Put this laptop back as you found it

```bash
ddev wp eval '
global $wpdb;
require_once ABSPATH . "wp-admin/includes/user.php";
$rows = (array) get_option( "pc_guide_withdrawals", [] );
foreach ( $rows as $row ) {
	$uid = (int) $row["user"];
	$wpdb->delete( $wpdb->prefix . "pc_transactions", [ "user_id" => $uid ], [ "%d" ] );
	$wpdb->delete( $wpdb->prefix . "pc_coin_lots", [ "user_id" => $uid ], [ "%d" ] );
	$wpdb->delete( $wpdb->prefix . "pc_wallets", [ "user_id" => $uid ], [ "%d" ] );
	wp_delete_user( $uid );
}
delete_option( "pc_guide_withdrawals" );
delete_option( "pc_realtime_withdrawal_alert_state" );
update_option( "pc_realtime_withdrawal_alert_count", 10, false );
update_option( "pc_realtime_withdrawal_alert_age_seconds", 86400, false );
update_option( "pc_realtime_withdrawal_alert_period_seconds", 86400, false );
echo "removed ", count( $rows ), " practice request(s); the backlog is ", PC\Wallet_Service::pending_withdrawal_summary()["count"], " waiting and the settings are back at their defaults", PHP_EOL;
'
ddev wp pc machine-poll
```

**Expected:**

```
removed 6 practice request(s); the backlog is 0 waiting and the settings are back at their defaults
withdrawals: nobody is waiting to be paid
```

The practice emails stay in the local inbox; delete them there if you like.

---

## 10. The automatic checks, run in one command

```bash
cd .. && backend/bin/check
```

**Expected — at the end:**

```
== Summary
php -l:      75 files OK
DDEV checks: passed — machine-ingest.php machine-poll.php machine-rooms.php queue-sessions.php realtime-channel.php realtime-chat.php realtime-outage.php realtime-queue.php realtime-relay.php realtime-toss.php realtime-withdrawals.php stripe-client.php stripe-webhook.php wallet-rollback.php
```

`realtime-withdrawals.php` is this step's own: **68 checks**, including the
two negative ones above, the off switch, a mailer that refuses, a mailer that
crashes, and a count of every money row before and after a real pass — because
this watcher only ever reads money, never moves it.

---

## 11. The one check only you can do — the admin screen

The **Withdrawals** screen in the operator app is what an operator actually
uses to clear a backlog, and it is the screen the alert email points at. It
sits behind a sign-in whose code arrives by email, so the assistant that
built this step never opened it. Please confirm once, by hand:

1. Start the operator app: `cd admin && npm run dev`, then open
   **http://localhost:5174**.
2. Sign in, and go to **Withdrawals**.
3. Redo section 3 of this guide in another terminal, and reload the screen:
   the three practice requests appear as `pending`.
4. **Approve** them on the screen.
5. In the terminal, run `ddev wp pc machine-poll` — it must say
   `nobody is waiting to be paid`.
6. Run section 9 to clean up.

What matters is that approving on the screen is what clears the alarm: the
alert and the screen are talking about the same queue.

---

## 12. On the real site, after the next deploy

Three things, in order of how much they matter:

1. **Set the address alerts go to.** `pc_realtime_alert_email` is empty by
   default, so alerts land in the support inbox — which is the inbox these
   alerts exist to stop you depending on. Set it once on the live site to an
   address you actually watch.
2. **Prove that email leaves the host at all.** Every check here happens
   inside DDEV, which catches mail locally. Whether the production hosting
   really delivers a `wp_mail` message has never been confirmed — for these
   alerts or for the support notifications that already rely on it. The
   cheapest proof is still the one Step 1's guide names: file one test
   support ticket on the live site and see whether it arrives.
3. **Check the thresholds against your own week.** Ten waiting and one day of
   waiting are guesses made at a desk. After a week of real traffic you will
   know what a normal Friday looks like; both are settings, changed with a
   command and no deploy.
