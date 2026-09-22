# Domain — Pusher Coin

<!-- The world the code models, in the customer's own words. Read before
     implementing any domain logic. Rules here are laws of the domain, not
     choices — a choice belongs in DECISIONS.md. Adoption mode: the terms are
     taken from the code and the existing documentation; the rules are the
     operator's to confirm. -->

## Glossary

| Term | Meaning |
|---|---|
| **Coin pusher** | The physical arcade machine in the venue. A coin is dropped onto a moving shelf crowded with coins; if the push sends coins over the edge, they fall into the payout tray. |
| **Machine** | One coin pusher, reachable only through Home Assistant. Identified by a machine id that a room points at (`pc_room_machine_id`). |
| **Room** | What the player sees: one machine, its live broadcast, its queue, its chat and its schedule. A room is `available`, in `maintenance`, or `unavailable`. |
| **Broadcast** | The live video of the machine. Streamed from the venue over RTMP into Mux and played back as LL-HLS. |
| **Schedule** | The weekly windows during which a room broadcasts. Rules are weekly and recurring, or a one-off on a given date. The *next window* is always computed, never stored. |
| **Coin** | The unit of play. A player buys coins with money and spends them one per toss. Every coin remembers the price it was bought at. |
| **Coin lot** | A batch of coins bought at one price. Lots are spent oldest first (FIFO), so a coin won pays back at the price of the coin being spent. |
| **Wallet** | A player's money balance and coin balance. |
| **Top-up** | Buying coins with money, through Stripe's hosted Checkout page. |
| **Withdrawal** | Turning coins back into money. Requested by the player, paid out by hand by an operator. |
| **Withdrawal backlog** | Withdrawal requests that have been asked for and not yet paid. *Named 2026-09-22 (`realtime` Sprint 3 Step 3).* Nothing is broken while one grows — no error, no failed call — it simply means nobody has been approving payouts, and because a player may have one request open at a time it is also a count of people waiting. It becomes worth interrupting an operator for on either of two counts: too many waiting at once, or one that has been waiting too long while newer ones were paid. Both numbers are settings. |
| **Queue** | The waiting line for a room, in arrival order. |
| **Turn** (bet session) | One player's stretch at the head of the queue: how many coins they played, how many they won, how much that was worth. |
| **Toss** | One coin dropped into the machine. The single moment a coin is spent. |
| **A toss that moved nothing** | A toss the machine accepted — Home Assistant answered 200 — and then did not act on: the player paid a coin and no coin went in. *Named 2026-09-21 (`realtime` Sprint 3 Step 2).* It is **not** the same as a toss that paid nothing, which is most tosses and is the game working normally. The machine's own coin counter is the witness: it resets to zero within about two seconds of a toss it accepts, so a counter standing above zero that never moves is the machine saying it did nothing. A counter already at zero cannot say anything either way — the reset would change nothing visible — so those tosses are counted and not written down. |
| **Bonus** | The machine's bonus wheel, landing on 1–12. Each number is worth a number of coins, set by the operator in the bonus map. *Observed 2026-09-18 (`DECISIONS.md`, the `realtime` spike): no bonus appeared in ten days of the machine's history, so how one is reported, and whether its coins also count as a normal payout, is still unknown.* |
| **Relay** | The machine's service switch — **not** a payout gate. *Settled 2026-09-21 (`DECISIONS.md`, `realtime` Sprint 2 Step 3), on the 2026-09-18 spike's observations.* `sensor.relay_on` idles **closed** (`1`) whenever the machine is working, never moves during a payout, and follows only the two relay buttons someone at the venue presses. So **closed is normal and open means the machine has been taken out of service by hand**: while it is open nobody may toss, because Home Assistant would still answer 200 to the toss and the coin would buy nothing. Nothing is ever credited from the relay — the earlier "when it closes, a configured number of coins is credited" rule described a payout signal the machine does not have, and `pc_machine_relay_coin_count` stays unreachable. |
| **Theme song** | Optional per-room audio the player can switch on. A device-local preference, never sent to the server. |
| **Nickname** | The name a player is known by in the queue and the chat. Chosen once, required before chatting or playing. |
| **Support subject** | An operator-maintained line in the support form's dropdown. |
| **Ticket** | One support request, with the subject, the description, and whether the sender's email was verified at the time. |

## Rules & invariants

- A room shows exactly one machine. What happens on screen is what happens in the venue.
- And the reverse: a machine is played through one room at a time. Two rooms pointed at the same machine would run two queues over one physical shelf, and a payout could not say whose it was.
- Only one player plays a room at a time. The queue is first come, first served by arrival time; the player at the head holds the turn. One turn per room is a fact the database keeps rather than a rule the software remembers to follow: two requests arriving in the same instant cannot open two turns for one room. They could until `realtime` Sprint 2 Step 5, and a payout could then be recorded against a turn the player never saw.
- A turn survives a page reload. It ends when the player leaves, goes quiet past the idle timeout, or the next player takes over.
- A coin is spent only when it is actually thrown. If the machine does not confirm the toss, the coin was not spent.
- A coin bought for a price is worth that price when it comes back. Money owed to a player never changes because the operator changed the coin price afterwards.
- Payouts are physical: the machine decides, not the software. The software only records what the machine reports and credits it to whoever holds the turn at that moment.
- If nobody holds the turn when a payout happens, the payout belongs to nobody and no wallet moves.
- A payout reaches the player about a minute after the coins fall, not instantly. The software asks the machine what happened on a schedule rather than being told, so the delay is the waiting between two questions (`DECISIONS.md` 2026-09-18 measured 65 s on a 60-second schedule). Shortening the schedule shortens the wait; nothing is lost while it waits, because the machine's own history still holds it.
- The operator is told when the machine stops answering **during a broadcast**, once per outage, by email. Outside broadcast hours the machine is switched off by hand and its silence is normal, so it is written down and nobody is disturbed — an alarm that went off every evening would be ignored within a week. A machine no room is pointed at is written down too, never sent: without a schedule there is no way to tell an evening from a fault.
- When a player's toss is accepted by the machine and the machine then does nothing, it is written down against that player and that turn, with what the machine's counter read on both sides. It is a record, not an alarm: nothing is sent to anybody, and it exists so a player's complaint can be answered with the machine's own evidence instead of a guess. The window the machine is given to respond is a setting.
- The operator is told by email when withdrawal requests pile up — too many waiting, or one waiting too long. They are told **once per pile-up**: not again while it lasts, however long that is, and again only after it has been cleared and grows back. There is a floor under that too, so a queue sitting right on the threshold cannot send a message every time it wobbles across. Nothing reminds and nothing escalates — an operator who ignores the first message is not chased by the software, because an alarm that repeats is an alarm people switch off. Paying or refusing the requests is what clears it, and the clearing is written down.
- The currency is the Ukrainian hryvnia. Every amount is exact to the kopiyka.
- Money leaves the system only by hand: a player requests a withdrawal, an operator approves and pays out, or rejects and the coins come back at their original prices. A player may have one withdrawal request open at a time.
- Before a player can move money or play, they must have accepted the current terms, chosen a nickname, and confirmed their email address. Chatting requires the terms and the nickname but not the confirmed email — chat moves no money.
- Anyone, signed in or not, may watch a broadcast and read a room's chat.
- Terms are versioned: when the operator publishes a new version, every player accepts again before they can play.
- A player's own record of what happened is the money trail — top-ups and withdrawals. Individual tosses are never itemised for them.
- Nothing a player or a moderator did disappears: a hidden chat message, a retired subject, a rejected withdrawal all stay on record.
- A muted player is muted everywhere, for as long as the operator set.

## Roles

| Role | Can do |
|---|---|
| **Guest** | Browse rooms and schedules, watch a broadcast, read a room's chat, file a support ticket (with an email address, and a captcha when the operator has configured one). |
| **Player** | Everything a guest can, plus: buy coins, join a queue, take a turn, toss coins, chat, request a withdrawal, see their own history. Gated on terms + nickname, and on a confirmed email for anything that touches money or play. |
| **Operator (admin)** | Create and schedule rooms, set the coin price and the bonus map, switch the machine on and off and watch its sensors, approve or reject withdrawals, triage support tickets and edit the subject list, moderate chat (hide a message, mute an account). Works exclusively in the admin SPA. |

## Explicitly out of scope

- Automated payouts and any automated KYC pipeline. Withdrawals are an operator paying by hand, out of band.
- Any product workflow inside `/wp-admin/`.
- Per-room mutes, itemised toss history for the player, and a lifetime winnings counter.
- Age gating, jurisdiction restrictions and the rest of the compliance surface — open, tracked in `ROADMAP.md`.
