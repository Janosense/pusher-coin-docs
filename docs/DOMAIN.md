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
| **Top-up** | Buying coins with money, through LiqPay Checkout. |
| **Withdrawal** | Turning coins back into money. Requested by the player, paid out by hand by an operator. |
| **Queue** | The waiting line for a room, in arrival order. |
| **Turn** (bet session) | One player's stretch at the head of the queue: how many coins they played, how many they won, how much that was worth. |
| **Toss** | One coin dropped into the machine. The single moment a coin is spent. |
| **Bonus** | The machine's bonus wheel, landing on 1–12. Each number is worth a number of coins, set by the operator in the bonus map. |
| **Relay** | The machine's payout gate. While it is closed the machine is paying out and no toss is allowed; when it closes, a configured number of coins is credited. |
| **Theme song** | Optional per-room audio the player can switch on. A device-local preference, never sent to the server. |
| **Nickname** | The name a player is known by in the queue and the chat. Chosen once, required before chatting or playing. |
| **Support subject** | An operator-maintained line in the support form's dropdown. |
| **Ticket** | One support request, with the subject, the description, and whether the sender's email was verified at the time. |

## Rules & invariants

- A room shows exactly one machine. What happens on screen is what happens in the venue.
- And the reverse: a machine is played through one room at a time. Two rooms pointed at the same machine would run two queues over one physical shelf, and a payout could not say whose it was.
- Only one player plays a room at a time. The queue is first come, first served by arrival time; the player at the head holds the turn.
- A turn survives a page reload. It ends when the player leaves, goes quiet past the idle timeout, or the next player takes over.
- A coin is spent only when it is actually thrown. If the machine does not confirm the toss, the coin was not spent.
- A coin bought for a price is worth that price when it comes back. Money owed to a player never changes because the operator changed the coin price afterwards.
- Payouts are physical: the machine decides, not the software. The software only records what the machine reports and credits it to whoever holds the turn at that moment.
- If nobody holds the turn when a payout happens, the payout belongs to nobody and no wallet moves.
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
