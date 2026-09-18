# Verification — `realtime` Sprint 1, Step 2: Spike — how machine events leave Home Assistant

**What this step was for.** It was an experiment, not a feature. It watched what the real
machine reports through its Home Assistant, then wrote down one decision: how machine
events will travel into WordPress. You chose **WordPress polls Home Assistant's history
every minute**. The experiment's scripts and logs were throwaway and were never committed.
Only documents changed.

**Nothing a player or an operator can see has changed.** No code changed in any of the
three apps, and nothing is deployed or pushed.

**Before you start.** Open Terminal and run each block below exactly as written, one
block at a time.

```bash
cd ~/Projects/full-stack/pusher-coin
```

---

## 1. The branches are where they should be

```bash
echo "docs:     $(git rev-parse --abbrev-ref HEAD)"
for d in backend frontend admin; do echo "$d: $(git -C $d rev-parse --abbrev-ref HEAD)"; done
git branch --list 'realtime/*'
```

**Expected:** `docs:     realtime/sprint-1`, then `backend: main`, `frontend: main`,
`admin: main`. The last command lists **only** `realtime/sprint-1` (the star marks the one
you are on). The step's own branch, `realtime/sprint-1-spike-transport`, was merged and
deleted.

---

## 2. Only documents changed — no experiment code, no logs, no token

```bash
git diff --stat b596b2d realtime/sprint-1
```

(`b596b2d` is where Step 1 ended, so this shows Step 2 only.)

**Expected:** only these files, all under `docs/`: `DECISIONS.md`, `DOMAIN.md`,
`LEARNINGS.md`, `PUSHER-COIN-COMMANDS.txt`, `ROADMAP.md`, `WORKLOG.md`, `SPRINT-1.md`,
`SPRINT-1-PLAN.md`, and this guide.

**Must NOT be there:** any `.mjs`, `.sh` or `.jsonl` file (the experiment's recorder,
press script and logs). Nothing from `backend/`, `frontend/` or `admin/`.

**Must NOT be there — the machine token.** Home Assistant tokens start with `eyJ`:

```bash
git grep -c -I 'eyJ[A-Za-z0-9_-]\{20,\}' -- . ; echo "search exit: $?"
```

**Expected:** only the line `search exit: 1`. `1` means "nothing found".

---

## 3. Read the decision — this is the main check

Open `docs/DECISIONS.md` and scroll to the **last** entry: **"2026-09-18 — Spike: machine
events reach WordPress by polling Home Assistant's history; the coin counter is the only
payout signal"**.

**Expected — it contains each of these, in plain words:**

1. **Exactly one transport:** "WordPress polls Home Assistant's history". The other two
   options appear only under *Alternatives rejected*, each with its reason.
2. **The event id formula:** `ha:{entity_id}:{last_updated}`, with a worked example
   (`ha:sensor.coin:2026-09-10T11:36:14.233147+00:00`).
3. **One latency number:** **65 s** from coins landing to the credit, with how it adds
   up (≤ 2 s machine cycle + ≤ 60 s schedule + ≤ 1.6 s request).
4. **The three answers, each from observation:**
   - *Does a repeated value leave a trace?* — **No** ("A repeated value leaves no trace").
   - *What does the coin counter count?* — the coins paid out **since the last toss**,
     and a toss resets it to 0.
   - *Which relay entity and value mean "closed"?* — `sensor.relay_on`, where **1 means
     closed** and is its normal state. It only follows the relay buttons, and neither
     relay sensor signals a payout.
5. **The unknown, said as such:** no bonus was seen in ten days.

**Must NOT be there — the word "probably".** The step is not done if the decision
guesses:

```bash
awk '/^## 2026-09-18 — Spike: machine events reach WordPress/,/^---$/' docs/DECISIONS.md | grep -ci 'probably'
```

**Expected:** `0`.

---

## 4. The machine's command reference and the domain terms carry the observation

Open `docs/PUSHER-COIN-COMMANDS.txt`.

**Expected:** a block starting `NOTE 2026-09-18: what the machine actually reports`, right
under the 2026-09-16 note near the top. The older lines further down (in Russian) are
**unchanged**; the note adds to them and does not rewrite them.

Open `docs/DOMAIN.md` and find the rows **Bonus** and **Relay** in the terms table.

**Expected:** each row keeps its original sentence and ends with an italic
*Observed 2026-09-18* note. The Relay note says no sensor reports the payout gate. The
Bonus note says no bonus has been seen.

---

## 5. The roadmap item this experiment replaced

Open `docs/ROADMAP.md` and find **Phase 5**, item **6. Documentation walk-through with
Dima**.

**Expected:** it is tagged **`[partial]`** (not `[todo]`, and not `[done]`). It says it
was replaced by the 2026-09-18 experiment, and it ends with **"Still open:"** about the
bonus.

At the bottom of the same file:
- the **Tracking matrix** row **6** says "6 partial (replaced by the `realtime` spike
  2026-09-18 …)";
- the open question "Walk through `PUSHER-COIN-COMMANDS.txt` with Dima" is struck
  through and marked resolved, except for the bonus.

---

## 6. Optional: see the relay's normal state yourself

This only works **while the machine is switched on** and while you still have
`~/.pusher-coin-ha-token`. It only **reads**; it presses nothing.

```bash
curl -sS -m 10 -H @<(printf 'Authorization: Bearer %s\n' "$(cat ~/.pusher-coin-ha-token)") https://developer-it.com/api/states/sensor.relay_on | jq -r .state
```

**Expected while the machine is on:** `1`. That is the relay's normal state, which the
decision says means "closed".
**If it prints `unavailable`:** the machine is off, so this check cannot be done now. It
said `unavailable` when this guide was written, because the machine had been switched
off. Nothing is wrong in that case.

---

## 7. Both check commands still pass

```bash
cd ~/Projects/full-stack/pusher-coin/backend && ddev start && git status --porcelain
bin/check; echo "exit code: $?"
cd ~/Projects/full-stack/pusher-coin/frontend && bin/check; echo "exit code: $?"
```

**Expected:** `git status --porcelain` prints nothing. If it prints
`M wp-config-ddev.php`, run `git checkout -- wp-config-ddev.php`, which fixes the known
DDEV quirk. Both checks end with `exit code: 0`, and the backend summary says
`DDEV checks: passed` (not `SKIPPED`).

---

## 8. What needs your decision (nothing to check, something to act on)

1. **Today the game refuses every toss while the machine is on.** The existing code reads
   the relay's normal `1` as "payout in progress" and answers "machine busy" (HTTP 423).
   That is a bug in the existing (`core`) code; fix it through `/adhoc`, not in this
   feature.
2. **Someone else used the same machine account.** A second "toss a coin" press at
   16:17:04 Kyiv on 2026-09-18 came from the same Home Assistant user as the token, but
   not from the experiment. If it wasn't you, the token is shared more widely than
   intended.
3. **The token file:** `~/.pusher-coin-ha-token` still holds the machine token. Delete it
   when you no longer need it (`rm ~/.pusher-coin-ha-token`). Steps 4 and 5 will need the
   token again.
4. **No bonus or relay crediting yet.** Step 4's plan will follow the decision: credit
   only from the coin counter until a bonus has actually been seen.

If any check in sections 1–7 does not match, report it with `/fix-step <what you saw>`.
