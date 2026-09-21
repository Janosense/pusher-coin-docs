# Verification — `realtime` Sprint 2, Step 4: Chat on the same channel

**What this step built.** The conversation in a room used to be fetched every three
seconds by every browser watching. Now a message is *pushed* the moment it is posted,
and so is a moderator hiding one — a hidden message disappears for everyone without a
reload. What used to be the transport, the `?after=` cursor, is now the catch-up: a
browser calls it when it opens the room and again whenever it reconnects.

**Two things still poll, on purpose.** A **guest** has no pass to the live channel —
the room and its chat are public, but a pass needs an account — so a guest's chat is
exactly as live as it was before. And any browser that cannot reach the channel falls
back rather than freezing.

**What you are checking.** That the conversation works for a signed-in player and for a
guest; that nobody can post who should not; that hiding a message removes it everywhere;
and that a browser that was away comes back with exactly what it missed.

**Time:** about 25 minutes for sections 1–7, which need **no Ably account**. Section 8
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

You will also want the operator app for section 7:

```bash
cd ~/Projects/full-stack/pusher-coin/admin
git checkout main
npm ci
```

`admin/` is **not** part of this step — it is untouched — but its **Chat moderation**
screen is how you hide a message.

---

## 2. Run the automated checks

```bash
cd ~/Projects/full-stack/pusher-coin/backend
bin/check
```

**Expect** the last three lines, with `realtime-chat.php` in the list:

```
== Summary
php -l:      66 files OK
DDEV checks: passed — machine-ingest.php machine-poll.php machine-rooms.php realtime-channel.php realtime-chat.php realtime-queue.php realtime-relay.php stripe-client.php stripe-webhook.php wallet-rollback.php
```

To see the chat checks by name — **40 of them**:

```bash
ddev wp eval-file wp-content/themes/pc/tests/realtime-chat.php
```

**Expect** `Success: 40 checks passed.` The ones worth reading as you go past:

```
PASS  payload: the body travels, because the read is public
PASS  payload: and nothing the read withholds does
PASS  muted: nothing is published
PASS  moderation: and carries NO body — the point of hiding it
PASS  cursor: it returns exactly what was missed, once each
```

Then:

```bash
cd ~/Projects/full-stack/pusher-coin/frontend
bin/check
```

**Expect** `lint: OK` and `build: OK`.

> **Worth knowing.** The backend half of this step has 40 automated checks. The browser
> half has **none** — the player app has no test runner. Sections 4 to 8 are the only
> thing that exercises the subscription, the catch-up and the fallback.

---

## 3. The cursor returns exactly what you missed

This is the mechanism that replaces the poll for a browser that was away. Run it against
the local **Sunset Pusher** room:

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp eval '
$u = get_users(["role"=>"player","number"=>1])[0];
$post = function($text) use ($u) { wp_set_current_user($u->ID); $r = new WP_REST_Request("POST","/pc/v1/rooms/11/messages"); $r->set_param("body",$text); $res = rest_do_request($r); wp_set_current_user(0); return (int) $res->get_status(); };
$read = function($after){ wp_set_current_user(0); $r = new WP_REST_Request("GET","/pc/v1/rooms/11/messages"); $r->set_param("after",$after); return (array) rest_do_request($r)->get_data(); };
echo "posted: " . $post("guide check one") . "\n";
$here = $read(0)["latest_id"];
echo "you are up to date at: $here\n";
$post("guide check two"); $post("guide check three");
$missed = $read($here);
echo "while you were away: " . implode(", ", array_map(fn($m)=>$m["body"], $missed["items"])) . "\n";
echo "read again with the new cursor: " . count($read($missed["latest_id"])["items"]) . " message(s)\n";
'
```

**Expect** exactly this shape — two messages missed, and nothing at all the second time:

```
posted: 201
you are up to date at: 114
while you were away: guide check two, guide check three
read again with the new cursor: 0 message(s)
```

The number after "up to date at" is whatever id your database is on and will differ from
the one above — the two lines that matter are the last two.

Nothing is sent twice and nothing is skipped. That is the whole catch-up.

---

## 4. The conversation works, signed in

1. Start the player app:
   ```bash
   cd ~/Projects/full-stack/pusher-coin/frontend
   npm run dev
   ```
2. Open `http://localhost:5173/room/11` and sign in as a verified player.
3. Open the chat panel and **send a message**.

**Expect:** your message appears in the list immediately, with your nickname, and the
three "guide check" messages from section 3 are above it.

> Your local site has no Ably key, so this is running on the fallback poll — which is
> exactly what section 6 is about. The conversation must work either way, and this is
> the "either way" half.

---

## 5. Negative check — a guest reads and cannot write

1. Open `http://localhost:5173/room/11` in a **private window** (so you are signed out).

**Expect:**

- the conversation is there and readable — a guest watching a broadcast sees what
  people are saying;
- the message box says **"Sign in to chat"** and clicking it opens the sign-in prompt
  rather than letting you type.

Leave the private window open — section 7 uses it.

---

## 6. Negative check — a mute is enforced by the server, not the browser

The player's browser has no idea it has been muted. The refusal has to come from the
server anyway.

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp eval '
$u = get_users(["role"=>"player","number"=>1])[0];
PC\Chat_Service::mute($u->ID, 10);
echo "muted {$u->user_login} for 10 minutes\n";
'
```

Now, in the **signed-in** window from section 4, type a message and press send.

**Expect:** the message is **not** posted, and the chat shows **"You are muted in
chat."** The input was never disabled — the server refused it.

Lift the mute:

```bash
ddev wp eval '
$u = get_users(["role"=>"player","number"=>1])[0];
PC\Chat_Service::mute($u->ID, 0);
echo "mute lifted\n";
'
```

Send another message. **Expect** it to go through.

---

## 7. A hidden message disappears for everyone

1. Start the operator app:
   ```bash
   cd ~/Projects/full-stack/pusher-coin/admin
   npm run dev
   ```
2. Open `http://localhost:5174/chat` and sign in as an administrator.
3. Find one of the messages you sent in section 4 and **hide** it.
4. Look at **both** browser windows from sections 4 and 5 — the signed-in one and the
   guest one. Do not reload either.

**Expect:** the message disappears from both **within about three seconds**. (With an
Ably account it is instant for the signed-in window; without one, both are on the
fallback poll. Either way it must go.)

5. **Restore** it from the same screen.

**Expect:** it comes back in both windows, again without a reload.

**The negative check here:** the hidden message must still exist for the operator. While
it was hidden, the admin **Chat moderation** list still showed it — with its author, its
text and its timestamp — because moderation hides, it never deletes. That is what a
complaint review needs.

---

## 8. The real check — two browsers, one room (needs an Ably account)

**This cannot be done yet: there is still no Ably account.** It is now the third step in
a row waiting on one, and it is the whole of what this sprint has never once exercised
for real. When you have one (the free tier is enough — see section 7 of the Step 1
guide):

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp config set PC_ABLY_KEY 'paste-the-real-key-here' --type=constant
```

Then, with `npm run dev` running:

1. Open `http://localhost:5173/room/11` in two different browsers, signed in as **two
   different players**.
2. Send a message from browser A. **Expect** it in browser B **immediately** — not after
   a wait of up to three seconds.
3. Hide that message from the admin **Chat moderation** screen. **Expect** it to vanish
   from both at once.
4. **Take the network away from browser B for about twenty seconds** — turn off Wi-Fi,
   or use its developer tools' "Offline" setting — while browser A sends two more
   messages. Restore the network.
   **Expect** browser B to show both missed messages when it comes back, each exactly
   once, with nothing duplicated and nothing skipped. Nothing needs reloading.
5. Open the room in a **private window** as a guest. **Expect** the conversation to work
   there too — on the 3-second poll, because a guest has no pass.

**A negative check while you are here.** With the channel up, open browser B's developer
tools → **Network** and filter for `messages`. **Expect** a request when you open the
room, and then **nothing at all** while messages arrive — not a request every three
seconds. A steady stream means the fallback is running and the channel is not connected.
In the guest window, the opposite: a request every three seconds is exactly right.

---

## 9. Clean up

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev wp config delete PC_ABLY_KEY     # only if you set one

# The messages this guide created, in your local database only.
ddev wp eval '
global $wpdb;
echo "removed " . $wpdb->query("DELETE FROM {$wpdb->prefix}pc_room_messages WHERE body LIKE \"guide check%\"") . " test rows\n";
'
```

Anything you typed in the browser is real local chat and can be left or hidden from the
moderation screen, as you prefer.

---

## 10. What was checked by whom

**Checked by the agent, locally:**

- The 40 automated backend checks: a posted message is published exactly once on the
  right channel, carrying byte for byte what the public read returns and **nothing
  more** — no IP, no status, no muted-until; a refused body, a muted author and a
  rate-limited caller publish nothing at all; hiding publishes an id and a state with
  **no body anywhere in the message**, and the message leaves the public read at the
  same moment while the row survives for the operator; a restore is announced too; a
  non-admin still cannot moderate; a dead Ably cannot fail a post; and the cursor
  returns exactly what was missed, once each, with no gap left by a message hidden
  meanwhile.
- Every command in sections 3 and 9 above, with the outputs shown.
- Both check commands.

**Not checked by the agent:**

- **Everything in a browser** — sections 4 to 8. `frontend/` has no test runner and the
  agent does not sign in to either app. This step reworked the shared channel service
  that the *queue* also uses, so section 8's room is worth a moment's attention on the
  queue as well: joining and leaving must still work exactly as they did.
- **Anything on a real Ably channel** — still no account, so every local check stubs the
  network and the whole of section 8 is untested.
