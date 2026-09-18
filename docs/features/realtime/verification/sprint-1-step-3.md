# Verification — `realtime` Sprint 1, Step 3: One machine, one active room

**What this step was for.** A room points at a physical machine through its **Machine ID**.
If two rooms are both "Available" on the same machine, two queues play one physical
machine, and a payout cannot tell whose it is. After this step:

- the admin app **refuses** to make a second room Available on a machine id another
  Available room already uses;
- a player **cannot join** a room while another Available room claims its machine
  (this only matters for rooms saved like that before this step);
- payouts go to the **Available** room carrying the machine id, never to a closed room
  that happens to carry the same id;
- a new report, `wp pc machine-rooms`, lists every machine id used by more than one room.

**What it does not do:** the software drives **one** physical machine, whatever id a room
carries. Two Available rooms with **different** ids, like your local "Sunset Pusher"
(`demo_sunset`) and "Midnight Pusher" (`demo_midnight`), still share it. The protection
works when rooms that share the machine use the same id.

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
- The check ends with `DDEV checks: passed — machine-rooms.php stripe-client.php
  stripe-webhook.php wallet-rollback.php` and `exit code: 0`.

`machine-rooms.php` is this step's automated test: 37 checks, which include the "join is
refused" case you cannot easily click through yourself.

---

## 3. The report, before you change anything

```bash
ddev wp pc machine-rooms
```

**Expected:** `Success: No machine id is held by more than one room.`
(Lines starting with `Deprecated:` may appear above it; they come from the JWT plugin
and are not part of this step.)

---

## 4. The admin app refuses a second Available room — the main check

Start the admin app in a **second** Terminal window and leave it running:

```bash
cd ~/Projects/full-stack/pusher-coin/admin
npm run dev
```

**Expected:** a line with `http://localhost:5174/`.

1. **Sign in.** In your browser open `http://localhost:5174/sign-in` and sign in with your
   local admin account. For the 6-digit code, open `https://pusher-coin.ddev.site:8026`
   (Mailpit), open the newest email, and type the code in.
2. **Look at the Room list.** Open `http://localhost:5174/rooms`. You should see
   **Sunset Pusher** with the status **Available**.
3. **Try to add a second Available room on the same machine.** Click **+ New room** and
   fill in:
   - **Name:** `Duplicate test`
   - **Status:** `Available`
   - **Machine ID:** `demo_sunset`

   Click **Create room**.

   **Expected:** the form stays open and shows this message in red under the fields:
   > Machine "demo_sunset" is already used by the available room "Sunset Pusher" (#11). Make that room unavailable first, or give this room another machine id.

   **Must NOT happen:** a room called "Duplicate test" appearing in the list. Click
   **Cancel** and check the list: it is not there.
4. **The same room is fine while it is closed.** Click **+ New room** again, with the same
   Name and Machine ID, but **Status:** `Unavailable`. Click **Create room**.

   **Expected:** you are back on the list, and "Duplicate test" is there as
   **Unavailable**.
5. **It cannot be switched to Available.** Click **Edit** on "Duplicate test", change
   **Status** to `Available`, and click **Save**.

   **Expected:** the same red message as in step 3. Click **Cancel**: the list still
   shows "Duplicate test" as **Unavailable**.
6. **The original room is untouched.** **Sunset Pusher** is still **Available**.

---

## 5. The report and the payout routing see the duplicate correctly

Back in the first Terminal window (in `backend`):

```bash
ddev wp pc machine-rooms
```

**Expected:** a table with two rows, both with machine id `demo_sunset`: **Sunset
Pusher** (`available`) and **Duplicate test** (`unavailable`). The **conflict** column
says `no` for both, because only one of them is Available.

Now ask the system which room a payout from machine `demo_sunset` would go to:

```bash
ddev wp eval 'var_dump( \PC\Queue_Service::room_id_for_machine( "demo_sunset" ) );'
```

**Expected:** `int(11)`, which is Sunset Pusher, the Available room.

**Must NOT be:** the id of "Duplicate test". Before this step, the newest room with the
id won, so the closed "Duplicate test" would have taken Sunset Pusher's payouts.

---

## 6. Clean up

In the admin app's Room list, click **Trash** on **Duplicate test** and confirm.

**Expected:** it disappears from the list. Then:

```bash
ddev wp pc machine-rooms
```

**Expected:** `Success: No machine id is held by more than one room.` again. A trashed
room does not count.

---

## 7. The player app needs no change

A player who tries to join a room caught in a pair (only possible with rooms saved before
this step) gets this message on the **Room** screen's Place-bet panel, sent by the server:

> This room is closed for now: its machine is also assigned to another open room. Please try again later.

The admin app no longer lets you create such a pair, so this case is covered by the
automated test in section 2 instead of by clicking.

If any check in sections 1–6 does not match, report it with `/fix-step <what you saw>`.
