# Verification — `stripe` Sprint 1, Step 2: Spike — Stripe on the test keys

**What this step was for.** Not building — finding out. The deliverable is a written
record of what Stripe actually does, so Step 3 builds on observation instead of
assumption. Every script the spike wrote has been thrown away.

**The question it had to answer:** *does Stripe accept UAH?* The sprint says Step 3
must not start on a guess.

**Nothing in the product changed.** No player or operator can see any difference, and
nothing was deployed.

```bash
cd ~/Projects/full-stack/pusher-coin
```

---

## 1. Read the record — it must report, not predict

Open `docs/DECISIONS.md` and scroll to the **last** entry:

> `## 2026-09-17 — Spike: Stripe accepts uah, on a provisional US sandbox; pin 2026-06-24.dahlia and turn Adaptive Pricing off`

**It must state all of these as things that happened:**

| Must be named | What it should say |
|---|---|
| The account | `acct_1TtSrOElMyJqvLDl`, **country US**, "Ask Debt Pros Sandbox sandbox" |
| The UAH answer | **Accepted** — HTTP 200, `"currency": "uah"`, `amount_total` `12000` |
| The pinned API version | `2026-06-24.dahlia`, and why not the newer `2026-08-26.dahlia` |
| A real payment message | A `checkout.session.completed` with `payment_status: paid` |
| The expiry observation | `checkout.session.expired`, `status: expired` |

> **The step fails if the entry hedges.** Search it for the words "should", "probably"
> or "expected to" — a spike that guesses has not done its job:
> ```bash
> grep -nE "should work|probably|expected to|presumably" docs/DECISIONS.md | tail -5
> ```
> **Expected:** no lines from the new entry.

---

## 2. The caveat must be visible, not buried

This is the most important thing to check, because it is the one that could mislead
later.

The account used is a **US sandbox** — not the client's account, which does not exist
yet. Confirm the entry says so plainly, and says the UAH answer must be re-checked on
the client's real account before go-live.

```bash
grep -c "provisional\|not the client's account\|re-verified on the client's real account" docs/DECISIONS.md
```

**Expected:** a number of 2 or more.

---

## 3. Re-run the decisive question yourself (optional, ~1 minute)

The spike's central claim can be re-tested at any time. This creates a payment page and
throws it away; **no money moves — the account is a test sandbox.**

```bash
KEY=$(stripe config --list | awk -F"'" '/test_mode_api_key/ {print $2; exit}')
printf 'header = "Authorization: Bearer %s"\n' "$KEY" | curl -sS -K - \
  https://api.stripe.com/v1/checkout/sessions \
  -d "mode=payment" \
  -d "success_url=https://pusher-coin.ddev.site/account?topup=success" \
  -d "cancel_url=https://pusher-coin.ddev.site/account?topup=cancel" \
  -d "line_items[0][quantity]=1" \
  -d "line_items[0][price_data][currency]=uah" \
  -d "line_items[0][price_data][unit_amount]=12000" \
  -d "line_items[0][price_data][product_data][name]=verification check" \
  | grep -E '"(currency|amount_total|url)"'
```

**Expected:** `"currency": "uah"`, `"amount_total": 12000`, and a
`https://checkout.stripe.com/...` address. That is the whole answer, reproduced.

You can paste that address into a browser to see **"UAH 120.00"** on Stripe's own page.
You do not need to pay it.

> The key is read from the Stripe CLI at the moment you run this. It is never written
> to a file and never stored in the project.

---

## 4. Confirm the account really is US (why the caveat exists)

```bash
KEY=$(stripe config --list | awk -F"'" '/test_mode_api_key/ {print $2; exit}')
printf 'header = "Authorization: Bearer %s"\n' "$KEY" | curl -sS -K - \
  https://api.stripe.com/v1/account | grep -E '"(id|country|default_currency)"'
```

**Expected:** `"country": "US"` and `"default_currency": "usd"` — which is exactly why
the record calls the UAH answer provisional.

---

## 5. The negative check: the temporary recorder must be GONE

During the spike a throwaway listener existed inside the site to capture what Stripe
sends. It must not have survived.

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/payments/stripe/webhook \
  -H 'Content-Type: application/json' -d '{}'
```

**Expected: `HTTP 404`.** The address must NOT answer.

> If this returns `200` with `{"spike":"recorded"}`, the throwaway file was left behind
> and must be removed: `rm backend/wp-content/mu-plugins/zz-stripe-spike.php`.

And the file itself:

```bash
ls backend/wp-content/mu-plugins/ 2>&1
```

**Expected:** "No such file or directory".

---

## 6. The negative check: no keys or card numbers were committed

```bash
cd ~/Projects/full-stack/pusher-coin
git log -p stripe/sprint-1..HEAD | grep -cE "sk_test_[A-Za-z0-9]{10}|whsec_[A-Za-z0-9]{10}|4242424242424242"
```

**Expected: `0`.** No secret material and no card number is in the history.

```bash
for d in . backend frontend admin; do echo "$d: $(git -C $d status --porcelain | wc -l) changes"; done
```

**Expected:** `0 changes` everywhere.

---

## 7. The code repositories were not touched

This step changed documentation only.

```bash
git -C backend log --oneline stripe/sprint-1 -1
git -C frontend log --oneline stripe/sprint-1 -1
git -C admin log --oneline stripe/sprint-1 -1
```

**Expected:** each still shows the Step 1 merge (`merge: sprint 1 step 1 — …`) — no new
commits from this step.

---

## What the spike found that nobody expected

You do not need to verify these; they are recorded so Step 3 and Step 4 can act on
them. Listed here because they are the reason the spike was worth running:

1. **Stripe turned on "Adaptive Pricing" by itself** — it converts the price into the
   buyer's own currency. Dormant for a Ukrainian buyer, but for a buyer elsewhere it
   would change the amount, and Step 4's job is to reject payments whose amount does
   not match. Step 3 must switch it off.
2. **The security stamp has three parts, not two** — code that takes "the last part"
   would check the wrong one.
3. **The six messages from one payment arrive out of order.** Step 4 cannot rely on
   sequence.
4. **Asking Stripe twice returns a stale answer** — correct for avoiding duplicate
   payment pages, wrong as a way to read current status.

## Known and accepted, not faults

- The answer comes from a **US sandbox**, not the client's account. Deliberate and
  recorded; it must be re-checked before go-live.
- That sandbox's test key **expires 2026-10-13**. Step 3's manual verification needs a
  working key before then.
- The spike paid with Stripe's **published test card number** in a `livemode: false`
  sandbox — a reserved number that belongs to nobody and moves no funds. This is noted
  because a standing rule otherwise forbids entering card numbers; the distinction is
  recorded in `docs/LEARNINGS.md` so Step 4 does not have to re-argue it.
- No `LEARNINGS.md` entry was needed for tooling: `stripe listen` accepted the local
  certificate with no workaround.
