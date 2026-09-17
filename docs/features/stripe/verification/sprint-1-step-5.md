# Verification — `stripe` Sprint 1, Step 5: The SPA hand-off, and LiqPay leaves

**What this step did.** Two things. The **top-up button works again** — it has been
deliberately broken since Step 3, and this is the step that reconnects it. And
**LiqPay is gone** from all three applications.

This is the first time you can do the whole thing the way a player will: press a
button, pay, and watch the coins arrive. Nothing here is done by hand on the server.

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev start
git status --porcelain      # if wp-config-ddev.php shows up: git checkout -- wp-config-ddev.php
```

> **`ddev start` deletes settings.** It rewrites both `wp-config-ddev.php` (restore it
> with the command above) **and** `wp-config.php`, which holds the Stripe keys. The
> `wp-config.php` one is **not** in git, so nothing can restore it for you — §2 below
> tells you what to put back. This is a known DDEV quirk, not a fault of this step.

---

## 1. The automatic checks

Three of them, one per application.

```bash
cd ~/Projects/full-stack/pusher-coin/backend  && bin/check
cd ~/Projects/full-stack/pusher-coin/frontend && bin/check
cd ~/Projects/full-stack/pusher-coin/admin    && npm run lint && npm run build
```

**Expected:** all three finish without an error. The backend one ends with:

```
Success: All 39 checks passed.      <- the Stripe client
Success: All 54 checks passed.      <- the webhook
Success: All 53 checks passed.      <- the money checks

== Summary
php -l:      48 files OK
DDEV checks: passed — stripe-client.php stripe-webhook.php wallet-rollback.php
```

Two details worth knowing:

- **48 files, not 50.** Two files were deleted this step. That drop is the point.
- **If you see a big `SKIPPED: DDEV checks did not run` banner, the check proved
  almost nothing** and you should start DDEV and run it again. The deleted code is the
  kind of mistake only a real WordPress start-up can catch.

---

## 2. Set up a live payment

You need **two terminal windows**.

### Terminal A — let Stripe reach your machine

```bash
stripe listen --forward-to https://pusher-coin.ddev.site/wp-json/pc/v1/payments/stripe/webhook
```

It prints a **signing secret** starting `whsec_`. Copy it. Leave this window running.

### Then — put both Stripe keys back

Open `backend/wp-config.php` and check these two lines exist, just above
`/** Include wp-settings.php */`. Add or correct them:

```php
define( 'PC_STRIPE_SECRET_KEY', 'sk_test_…your test key…' );
define( 'PC_STRIPE_WEBHOOK_SECRET', 'whsec_…the value from Terminal A…' );
```

> The webhook secret changes **every time** you restart `stripe listen`. Production
> uses a different one, from the Stripe Dashboard.

### Terminal B — start the player SPA

```bash
cd ~/Projects/full-stack/pusher-coin/frontend
npm run dev
```

---

## 3. The real thing — press the button and get coins

This is the step's whole promise, and it is the first time it can be done from the
screen instead of the command line.

1. Open **http://localhost:5173** and sign in as a player.
2. Go to **Account** and note the **coin balance** you see. Write it down.
3. Open **Replenishment balance** (the top-up panel), choose **2 coins**, leave the
   price at its default, and press **Buy**.

**Expected:** the browser leaves your site and lands on **Stripe's own payment page**,
showing **2 coin(s)** and the matching **UAH** amount.

> **If you get a red error instead and stay on the page, this step has failed** —
> that was the broken behaviour Steps 3 and 4 left behind, and fixing it is what this
> step is for.

4. Pay with Stripe's test card:

| Field | Value |
| --- | --- |
| Card number | `4242 4242 4242 4242` |
| Expiry | any future date, e.g. `12 / 34` |
| CVC | any three digits |
| Name / address | anything |

> Stripe's published test card in a sandbox — it belongs to nobody and moves no real
> money.

**Expected, in order:**

- the browser comes back to **Account** by itself,
- a **green success banner** appears,
- the **coin balance is 2 higher** than what you wrote down,
- Terminal A shows `checkout.session.completed` answered `[200]`.

**You should not have to reload the page by hand.** That is a Definition-of-Done item
for this sprint.

5. Now press **Buy** again, and on Stripe's page use the **back arrow** instead of
   paying.

**Expected:** you are back on **Account** with a **cancel banner**, and the balance is
**unchanged**.

---

## 4. Negative check — the old payment address is dead

```bash
curl -sS -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/payments/liqpay/callback \
  -d 'data=x&signature=y' -w '\nHTTP %{http_code}\n'
```

**Expected: `HTTP 404`** with `{"code":"rest_no_route", …}`.

And the Stripe one must still answer:

```bash
curl -sS -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/payments/stripe/webhook \
  -H 'Content-Type: application/json' -d '{}' -w '\nHTTP %{http_code}\n'
```

**Expected: `HTTP 400`** with `{"code":"missing_required_fields", …}` — it refuses an
unsigned request, which means it is still registered and still guarded.

---

## 5. Negative check — nothing disappeared from history

Old LiqPay-era top-ups must still be visible. Removing the integration must not
remove the record of money people actually spent.

Open **History** in the player SPA.

**Expected:** the older top-up rows are still listed, exactly as before.

> **If any past transaction has vanished, stop and report it.** That is the one thing
> this step was not allowed to do.

---

## 6. Negative check — LiqPay is really gone from the code

Run these **three separate commands**. Each one searches one application.

```bash
cd ~/Projects/full-stack/pusher-coin
git -C backend  grep -niI liqpay -- . ; echo "--- backend done ---"
git -C frontend grep -niI liqpay -- . ; echo "--- frontend done ---"
git -C admin    grep -niI liqpay -- . ; echo "--- admin done ---"
```

> **Do not** use a single search from the project folder. The three applications are
> separate repositories that the top-level folder deliberately ignores, so one search
> from there finds nothing in them and looks like a pass. This nearly happened while
> building this step — `docs/LEARNINGS.md` 2026-09-17.

**Expected:**

- **frontend — nothing.**
- **admin — nothing.**
- **backend — exactly seven lines in three files, and nothing else:**

| File | Why it stays |
| --- | --- |
| `app/utils/install-schema.php` (5 lines) | This is the code that **deletes** the old LiqPay setting from a site that still has it. Remove the name and the setting would be stranded there forever. |
| `app/rest-api/WalletController.php` (1 line) | A note that older top-up records keep their old reference and still display — the thing you checked in §5. |
| `app/stripe/StripeWebhookController.php` (1 line) | Quotes the title of the decision that created the Stripe webhook. |

**Anything else in `backend`, or anything at all in `frontend` or `admin`, is a
leftover** — report it.

---

## When you are done

Stop Terminal A and the dev server with `Ctrl+C`, then:

```bash
cd ~/Projects/full-stack/pusher-coin/backend && ddev stop
```

**If everything above matched, the step passes.** If any check did not match, say which
number failed and what you saw — that goes to `/fix-step`, not a direct patch.

## Known and accepted, not faults

- **The admin Settings screen shows no live Stripe status.** It now names the two
  server settings and the webhook address to register, as static text. The green/red
  "configured" badge is Sprint 2.
- **The production webhook URL is still not registered in the Stripe Dashboard.**
  Required before the first release — it is on the sprint's Definition of Done, not
  this step's.
- **The sandbox test key expires 2026-10-13.**
- **`ddev start` keeps deleting your Stripe keys** from `wp-config.php`. Known quirk.
- **Docs still mention LiqPay**, on purpose: the worklog, the decisions, the roadmap
  and the audit records are history and are meant to stay readable.
