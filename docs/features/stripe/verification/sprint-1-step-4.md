# Verification — `stripe` Sprint 1, Step 4: The webhook and settlement

**What this step did.** This is the one that makes the feature work. Before it, a
player could pay Stripe and get nothing back. Now Stripe's confirmation reaches the
server and **the coins appear in the wallet**.

**Still deliberately broken:** the player's top-up *button*. The browser side is
Step 5. Everything below drives the server directly, which exercises exactly the same
path a real player will.

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev start
git status --porcelain      # expect no output; if wp-config-ddev.php changed: git checkout -- wp-config-ddev.php
```

---

## 1. The automatic checks

```bash
cd ~/Projects/full-stack/pusher-coin/backend
bin/check
```

**Expected:** three success lines and a clean summary —

```
Success: All 39 checks passed.      <- the Stripe client
Success: All 54 checks passed.      <- the webhook (new in this step)
Success: All 53 checks passed.      <- the money checks

== Summary
php -l:      50 files OK
DDEV checks: passed — stripe-client.php stripe-webhook.php wallet-rollback.php
```

The 54 include the ones worth knowing about: a payment credits **once**; the same
message twice changes nothing; a forged, stale or missing signature is refused; an
amount **one kopiyka short** is refused; an expiry notice arriving after payment
leaves the paid record alone; and a deliberately broken database is asked to retry and
then settles correctly on that retry.

---

## 2. The real thing — a live payment credits a wallet

This is the step's whole promise. You need two terminal windows.

### Terminal A — let Stripe reach your machine

```bash
stripe listen --forward-to https://pusher-coin.ddev.site/wp-json/pc/v1/payments/stripe/webhook
```

It prints a line containing a **signing secret** starting `whsec_`. Copy it.

Leave this window running.

### Then — tell the server that secret

Open `backend/wp-config.php` and set the webhook secret line to the value you just
copied:

```php
define( 'PC_STRIPE_WEBHOOK_SECRET', 'whsec_…the value from Terminal A…' );
```

> This secret is **per `stripe listen` session** — it changes each time you start it.
> Production uses a different one, registered in the Stripe Dashboard.

### Terminal B — create a top-up and note the balance

```bash
cd ~/Projects/full-stack/pusher-coin/backend

PLAYER=$(ddev wp eval '$u=get_user_by("login","stripe-step3-check");echo $u->ID;' 2>/dev/null | tr -d '\r\n')
echo "coins BEFORE: $(ddev wp eval "echo \PC\Wallet_Service::get_wallet($PLAYER)['balance_coins'];" 2>/dev/null)"

TOKEN=$(ddev wp eval '$u=get_user_by("login","stripe-step3-check");echo \PC\AuthController::issue_access_token($u);' 2>/dev/null | tr -d '\r\n')
curl -sS -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/wallet/topup \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"coin_qty":2,"unit_price":"40.00"}'
```

> If the player does not exist, create it with the block in §3 of
> `sprint-1-step-3.md` first.

**Write down the "coins BEFORE" number.** Copy the `checkout_url` from the reply.

### Pay it

Open that `checkout_url` in a browser and pay with Stripe's test card:

| Field | Value |
| --- | --- |
| Card number | `4242 4242 4242 4242` |
| Expiry | any future date, e.g. `12 / 34` |
| CVC | any three digits, e.g. `123` |
| Name / address | anything |

> This is Stripe's published test card in a sandbox — it belongs to nobody and moves
> no real money.

**In Terminal A** you should see, within a second or two:

```
--> checkout.session.completed [evt_…]
<--  [200] POST https://pusher-coin.ddev.site/wp-json/pc/v1/payments/stripe/webhook
```

Several other events also arrive and are answered `[200]` — that is correct, they are
acknowledged and ignored.

### The check that matters

```bash
echo "coins AFTER: $(ddev wp eval "echo \PC\Wallet_Service::get_wallet($PLAYER)['balance_coins'];" 2>/dev/null)"
ddev wp db query "SELECT id, amount_money, amount_coins, status, settled_at FROM wp_pc_transactions ORDER BY id DESC LIMIT 1"
```

**Expected:**

- **coins AFTER = coins BEFORE + 2**
- the row shows `status` **`completed`** and a real `settled_at` timestamp

That is the feature working end to end.

---

## 3. Negative check — the same confirmation twice credits nothing twice

This is a sprint completion requirement, so do it properly.

Find the event id from Terminal A (the `evt_…` next to `checkout.session.completed`),
then:

```bash
stripe events resend evt_…paste it here…
```

Then check again:

```bash
echo "coins after resend: $(ddev wp eval "echo \PC\Wallet_Service::get_wallet($PLAYER)['balance_coins'];" 2>/dev/null)"
ddev wp db query "SELECT event_type, created_at FROM wp_pc_auth_audit_log WHERE event_type LIKE 'stripe_webhook_%' ORDER BY id DESC LIMIT 3"
```

**Expected:**

- **the balance is unchanged** — still `BEFORE + 2`
- the newest audit row is **`stripe_webhook_already_settled`**

> **If the balance went up again, that is a serious defect** — stop and report it.

---

## 4. Negative check — a chargeback changes nothing

```bash
BEFORE_D=$(ddev wp eval "echo \PC\Wallet_Service::get_wallet($PLAYER)['balance_coins'];" 2>/dev/null)
stripe trigger charge.dispute.created
sleep 3
echo "before: $BEFORE_D  after: $(ddev wp eval "echo \PC\Wallet_Service::get_wallet($PLAYER)['balance_coins'];" 2>/dev/null)"
```

**Expected:** the two numbers are **the same**, and Terminal A shows the dispute event
answered `[200]`. Disputes and refunds are explicitly out of scope for v1 — the
product acknowledges them and does nothing.

---

## 5. Negative check — an unsigned request is refused

```bash
curl -sS -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/payments/stripe/webhook \
  -H 'Content-Type: application/json' -d '{"type":"checkout.session.completed"}' \
  -w '\nHTTP %{http_code}\n'
```

**Expected: `HTTP 400`** with `{"code":"missing_required_fields", …}`.

And with a **forged** signature:

```bash
curl -sS -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/payments/stripe/webhook \
  -H 'Content-Type: application/json' \
  -H "Stripe-Signature: t=$(date +%s),v1=0000000000000000000000000000000000000000000000000000000000000000" \
  -d '{"type":"checkout.session.completed"}' -w '\nHTTP %{http_code}\n'
```

**Expected: `HTTP 401`** with `{"code":"stripe_signature_invalid", …}`.

> **Anyone able to credit a wallet by posting to this address without a valid
> signature would be a critical defect.** These two checks are what rule it out.

---

## 6. Negative check — an abandoned payment does not stay "pending" forever

Create a top-up but do **not** pay it, then expire it from Stripe's side:

```bash
TOKEN=$(ddev wp eval '$u=get_user_by("login","stripe-step3-check");echo \PC\AuthController::issue_access_token($u);' 2>/dev/null | tr -d '\r\n')
REF=$(curl -sS -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/wallet/topup \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"coin_qty":1,"unit_price":"40.00"}' | sed -n 's/.*"external_ref":"\([^"]*\)".*/\1/p')

KEY=$(stripe config --list | awk -F"'" '/test_mode_api_key/ {print $2; exit}')
printf 'header = "Authorization: Bearer %s"\n' "$KEY" | curl -sS -K - -X POST \
  "https://api.stripe.com/v1/checkout/sessions/$REF/expire" > /dev/null
sleep 3

ddev wp db query "SELECT status, notes FROM wp_pc_transactions WHERE external_ref='$REF'"
```

**Expected:** `status` is **`failed`** and `notes` mentions
`checkout.session.expired`. The balance must **not** have changed.

---

## When you are done

Stop Terminal A with `Ctrl+C`, then:

```bash
cd ~/Projects/full-stack/pusher-coin/backend && ddev stop
```

**If everything above matched, the step passes.** If any check did not match, say which
number failed and what you saw — that goes to `/fix-step`, not a direct patch.

## Known and accepted, not faults

- **The player's top-up button still throws an error.** Step 5 rewires the browser.
- **Several events arrive per payment and most are "ignored".** Correct — one card
  payment produces six or so events, and only the session one settles anything.
- **The signing secret changes every time you restart `stripe listen`.** Per-session by
  design; production registers its own in the Stripe Dashboard.
- **`ddev start` will drop the Stripe lines from `wp-config.php`** — a known DDEV quirk
  in `docs/LEARNINGS.md`.
- **The production webhook URL is not registered yet.** That is a sprint-completion
  item, required before the first release, not part of this step.
- **A rolled-back settlement answers 500 on purpose**, so Stripe retries —
  `DECISIONS.md` 2026-09-17. It is the one failure the handler does not swallow.
