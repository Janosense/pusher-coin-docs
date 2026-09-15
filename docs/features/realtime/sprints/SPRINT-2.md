# SPRINT 2 — The room stops polling (after Sprint 1)

**Branch:** `realtime/sprint-2` ← `main`, task branches `realtime/sprint-2-{short-name}` merged `--no-ff`. Same branch name in `backend/` and `frontend/`; they merge together at the sprint boundary.
**Goal:** A player watching a room sees coins, bonuses and the relay lock as they happen, pushed rather than polled. The toss button greys out *before* the player tries, because the room knows the relay is closed. Three 3-second polls — queue, chat, admin machine state — are gone, and with them the duplicate-session race they caused. Two browsers open on the same room show the same event at the same time.

## Fixed decisions
- The channel is **Ably**, free tier — `DECISIONS.md` 2026-09-15. The key is a wp-config constant; the SPA gets a scoped token and never the key.
- Publishing is **fire-and-forget** — `FEATURE.md` → Invariants #2. A failed publish is logged and swallowed. It may never fail, delay or roll back the money path that triggered it.
- The queue is persisted and pruned on read — `DECISIONS.md` 2026-07-30. Removing the poll does not mean removing the heartbeat; it means the heartbeat stops being a full read.
- Chat reads are cursor-based on `id` — `DECISIONS.md` 2026-09-07. The cursor survives the move to push; it is how a reconnecting client catches up.
- `PlaceBet`, `stores/queue.js` and `stores/chat.js` belong to `core`. Every step here that edits them is a plan task marked **"touches shared code"**.

## Steps

### [ ] Step 1 — Publish to Ably, and hand the SPA a token
- **Tasks:**
  - `app/realtime/` gains a publisher: on `pc_machine_event_credited` and on a relay-state change, publish a compact event to the room's channel. Wrapped so any failure is logged and swallowed.
  - `GET /pc/v1/realtime/token` — a scoped Ably token request for a signed-in caller, limited to the channels that caller may read. Admins additionally get the machine channel.
  - Channel naming convention fixed and written down: one channel per room, one for the machine/admin view.
  - `PC_ABLY_KEY` as a wp-config constant, documented in `DATA-MODEL.md` alongside the other secrets. `Install_Schema` seeds any new option default.
- **Tests:** A publish failure (network error, bad key) does not change the outcome or the timing of the credit that triggered it. The token endpoint refuses an anonymous caller and never returns the key itself. A non-admin cannot obtain a token for the machine channel.
- **Verification (manual):** Trigger an ingest event from Sprint 1 and watch it arrive in Ably's own dashboard on the expected channel. Then break the key deliberately and repeat — the player's balance still moves correctly and the error appears in the log.
- **Docs to update:** `docs/CONTRACTS.md` (token endpoint); `docs/DATA-MODEL.md` (the constant and any option); `docs/ARCHITECTURE.md` → Integrations (Ably row) and the data flow.
- **Depends on:** —

### [ ] Step 2 — The queue subscribes instead of polling
- **Tasks:**
  - `frontend/src/services/realtime.js`: connect using the token endpoint, subscribe to the room channel, expose an event stream to the stores. Reconnect with backoff; on reconnect, catch up rather than assuming nothing was missed.
  - `stores/queue.js` stops its 3-second full read. **Touches shared code** — `core` owns this store.
  - Keep a heartbeat, because `pc_queue_idle_timeout_seconds` pruning depends on it, but make it a cheap write rather than a full queue read.
  - `UserControls` updates winnings from the pushed credit instead of waiting for the next poll.
- **Tests:** A dropped connection followed by a reconnect leaves the queue state correct, with no duplicated entries and no lost turn. The heartbeat still holds a place across a reconnect.
- **Verification (manual):** Open the **Room** screen in two browsers with two accounts. One joins the queue — the other sees it without waiting three seconds. Kill the network on one for twenty seconds and restore it — its queue and winnings are correct afterwards, and it did not lose its place.
- **Docs to update:** `docs/ARCHITECTURE.md` (the queue data flow says "polled every 3s" in several places); `docs/features/realtime/FEATURE.md` → Interfaces; `docs/TECH-STACK.md` → ANTI-PATTERNS ("do not add a cron for queue housekeeping" justifies itself with the 3s poll being the heartbeat — the reason changes here, the rule does not); `docs/DECISIONS.md` — a **new** entry superseding 2026-07-30 on what the heartbeat is now, never an edit to the old one (the log is append-only).
- **Depends on:** Step 1

### [ ] Step 3 — The relay lock the player can see
- **Tasks:**
  - Publish relay open/closed transitions to the room channel.
  - `PlaceBet` disables the toss control while the relay is closed, with the reason visible, and re-enables it on the opening transition. **Touches shared code.**
  - Keep the server-side 423 refusal as the authority — the button is a courtesy, not a gate.
  - This closes the SPA half of ROADMAP Phase 5 §5; update its tag in the same commit.
- **Tests:** The button state follows the pushed transitions. A toss attempted in the gap between the machine closing the relay and the event arriving is still refused by the server with 423 and the coin is not lost.
- **Verification (manual):** Close the relay on the machine from the admin **Machine** screen while a player has the **Room** screen open — the toss button greys out within the latency Sprint 1 measured, before any attempt. Open it again — the button returns.
- **Docs to update:** `docs/ROADMAP.md` (Phase 5 §5 → `[done]`); `docs/DESIGN.md` → Components (PlaceBet gains a state).
- **Depends on:** Step 2

### [ ] Step 4 — Chat on the same channel
- **Tasks:**
  - Publish new messages and moderation events to the room channel; `stores/chat.js` subscribes and drops its 3-second poll. **Touches shared code.**
  - Keep the `after` cursor as the catch-up path for a reconnecting or newly-opened client.
  - A hidden message must disappear for everyone without a reload; a muted account must be refused server-side regardless of what its client believes.
- **Tests:** A reconnecting client catches up through the cursor without duplicating or skipping messages. Moderation reaches other viewers. The rate limit is still enforced server-side.
- **Verification (manual):** Two browsers on the **Room** screen: a message from one appears in the other without a poll delay. Hide it from the admin **Chat moderation** screen — it disappears in both. Mute one account — its next message is refused.
- **Docs to update:** `docs/ARCHITECTURE.md` (the chat data flow); `docs/DECISIONS.md` if the cursor's role changed.
- **Depends on:** Step 2

### [ ] Step 5 — The duplicate-session race
- **Tasks:**
  - `BACKEND-REVIEW.md` §15: simultaneous queue requests can open two bet sessions for one room that never close. Removing the polls reduces the traffic that caused it but does not remove the race.
  - Make "open a session for the head of the queue" atomic, so the `ended_at IS NULL` invariant — at most one open session per room — holds under concurrency.
  - Report and close any orphaned sessions already in the database.
- **Tests:** Money zone. Concurrent joins produce exactly one open session. An orphan left by the old code is closed by the cleanup, not by a payout landing in it.
- **Verification (manual):** Fire concurrent joins at one room and query the sessions table — exactly one row has `ended_at IS NULL`. Then run a real turn and confirm the win lands on it.
- **Docs to update:** `docs/DATA-MODEL.md` → Invariants (state how the single-open-session rule is now enforced, not just asserted); `docs/BACKEND-REVIEW.md` (mark §15).
- **Depends on:** Step 2

## Definition of Done
- [ ] Every step closed via /close-step (report + verification guide + worklog)
- [ ] The check command exits 0 on the sprint branch in both repositories
- [ ] Docs match reality (ARCHITECTURE no longer describes the removed polls; DATA-MODEL, CONTRACTS, DECISIONS current)
- [ ] Sprint boundary: merged into `main` in both repositories; `frontend` deployed by Vercel from `main` and `backend` released by the FTP push — that pair of deploys is what verifies the channel in production
- [ ] No 3-second poll remains in `frontend/src/` — grep for the intervals and confirm
- [ ] Tymofii saw the toss button grey out *before* an attempt, on a real relay close
- [ ] Two browsers on one room showed the same event at the same time, observed by Tymofii
- [ ] A full day of production traffic recorded against the Ably free tier, with the peak concurrent-connection count written into `docs/TECH-STACK.md` — 200 is the ceiling to watch

## Out of scope
- Ops alerts and anything that notifies an operator — Sprint 3.
- The admin **Machine** screen's own 3-second poll: it can move to the machine channel here if Step 1 makes it free, but it is not a Definition of Done item; the player-facing polls are.
- The operations dashboard — a future `ops` feature.

## Risks / notes
- The Ably free tier caps at 200 concurrent connections and 200 channels. One channel per room is fine; one channel per *viewer* would not be. Fix the convention in Step 1 and do not drift from it.
- Removing a poll removes a side effect somebody may be relying on — the queue poll is also the heartbeat and the pruning trigger. Step 2 must keep the heartbeat deliberately, not accidentally.
- A player on a flaky mobile connection is the normal case, not the edge case. Reconnect-and-catch-up is the feature, not an afterthought.
