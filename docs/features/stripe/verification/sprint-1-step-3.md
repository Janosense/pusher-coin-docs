# Verification — `stripe` Sprint 1, Step 3: The Stripe client and the Checkout Session

**What this step did.** The server side of "top up" now talks to Stripe instead of
LiqPay. Asking the server for a top-up creates a real Stripe payment page and returns
its address.

**Read this first — one thing is deliberately broken.**

> **The player's top-up button does not work yet, and that is expected.**
> The server now returns a Stripe address, but the browser is still written for
> LiqPay and will throw an error. Rewiring the browser is **Step 5**. Nothing is
> deployed, so no real player is affected. That is why this guide calls the server
> directly instead of clicking the button.

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev start
```

After `ddev start`, check nothing was wiped:

```bash
git status --porcelain
```

**Expected:** no output. If `M wp-config-ddev.php` appears, run
`git checkout -- wp-config-ddev.php`.

---

## 1. Put the Stripe keys in your local config

The server refuses to do anything without them. They live in `wp-config.php`, which is
**not** tracked by git, so they never reach a commit.

```bash
grep -c PC_STRIPE wp-config.php
```

**Expected: `2`.** If it prints `0`, `ddev start` wiped them (a known quirk — see
`docs/LEARNINGS.md`). Add them back:

```bash
KEY=$(stripe config --list | awk -F"'" '/test_mode_api_key/ {print $2; exit}')
printf "\ndefine( 'PC_STRIPE_SECRET_KEY', '%s' );\ndefine( 'PC_STRIPE_WEBHOOK_SECRET', 'whsec_local_placeholder' );\n" "$KEY" >> /dev/null
```

> That command deliberately does nothing — writing a secret from a copy-paste is easy
> to get wrong. Open `wp-config.php` in your editor, and above the line
> `/* That's all, stop editing! Happy publishing. */` add two lines:
>
> ```php
> define( 'PC_STRIPE_SECRET_KEY', 'sk_test_…' );   // from the Stripe CLI or Dashboard
> define( 'PC_STRIPE_WEBHOOK_SECRET', 'whsec_local_placeholder' );
> ```
>
> For this step the webhook secret can be any non-empty text — nothing verifies a
> signature until Step 4. The real secret key must be a genuine `sk_test_…`.

Confirm the server sees them, without printing them:

```bash
ddev wp eval 'echo \PC\Stripe_Client::is_configured() ? "configured, mode=" . \PC\Stripe_Client::mode() : "NOT configured";'
```

**Expected:** `configured, mode=test`.

> **`mode=live` would be a red flag** — it means a real key is in a development
> config. Stop and replace it with a test key.

---

## 2. Run the automatic checks

```bash
cd ~/Projects/full-stack/pusher-coin/backend
bin/check
```

**Expected:** ends with

```
== Summary
php -l:      48 files OK
DDEV checks: passed — stripe-client.php wallet-rollback.php
```

and higher up, **two** success lines:

```
Success: All 39 checks passed.      <- the new Stripe checks
Success: All 53 checks passed.      <- the existing money checks
```

The 39 cover the money conversion at its edges, test-vs-live key detection, the refusal
when configuration is missing, the security-stamp checking, and the saved-id behaviour.

---

## 3. Ask the server for a top-up (the main check)

This is the step's whole promise. Copy the block as-is; it creates a test player,
signs in as them, and asks for 3 coins at 40.00 UAH.

```bash
cd ~/Projects/full-stack/pusher-coin/backend

ddev wp eval '
$u = get_user_by( "login", "stripe-step3-check" );
if ( ! $u ) { $u = get_user_by( "id", wp_create_user( "stripe-step3-check", wp_generate_password( 24 ), "stripe-step3-check@example.com" ) ); }
$u->set_role( "player" );
update_user_meta( $u->ID, \PC\User_Meta_Keys::EMAIL_VERIFIED_AT, current_time( "mysql" ) );
update_user_meta( $u->ID, \PC\User_Meta_Keys::TERMS_ACCEPTED_VERSION, get_option( "pc_terms_current_version" ) );
update_user_meta( $u->ID, \PC\User_Meta_Keys::TERMS_ACCEPTED_AT, current_time( "mysql" ) );
update_user_meta( $u->ID, \PC\User_Meta_Keys::NICKNAME_CHOSEN, "1" );
'

TOKEN=$(ddev wp eval '$u=get_user_by("login","stripe-step3-check");echo \PC\AuthController::issue_access_token($u);' 2>/dev/null | tr -d '\r\n')

curl -sS -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/wallet/topup \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"coin_qty":3,"unit_price":"40.00"}' -w '\nHTTP %{http_code}\n'
```

**Expected: `HTTP 200`** and a reply shaped like this:

```json
{
  "transaction_id": 59,
  "external_ref": "cs_test_a1rqvK7C…",
  "amount": "120.00",
  "checkout_url": "https://checkout.stripe.com/c/pay/cs_test_…"
}
```

Check three things:

1. `amount` is **`"120.00"`** — 3 coins at 40.00.
2. `external_ref` starts with **`cs_test_`** (a real Stripe session; `cs_live_` would
   mean a live key).
3. `checkout_url` points at **checkout.stripe.com**.

**Now open that `checkout_url` in a browser.** You should see Stripe's own payment page
showing:

- **UAH 120.00**
- the line **"Pusher Coin top-up — 3 coin(s) @ 40.00 UAH"**

> **Do not pay it.** Nothing settles a payment until Step 4, so paying now would take
> a test card charge that no wallet ever receives. Just look and close the tab.

---

## 4. The money record exists and is waiting

```bash
ddev wp db query "SELECT id, amount_money, amount_coins, status, settled_at, LEFT(external_ref,12) AS ref FROM wp_pc_transactions ORDER BY id DESC LIMIT 1"
```

**Expected:** one row with `amount_money` `120.00`, `amount_coins` `3`, `status`
**`pending`**, `settled_at` **`NULL`**, `ref` starting `cs_test_`.

`pending` with no settlement date is exactly right: the money is not credited until
Stripe confirms the payment, which is Step 4's job.

---

## 5. Negative check — a half-configured server writes nothing

This is the most important check in the guide. A server missing a key must refuse
**before** creating a record it could never complete.

In `wp-config.php`, comment out the webhook secret line by putting `//` in front:

```php
//define( 'PC_STRIPE_WEBHOOK_SECRET', '…' );
```

Then count the records, try a top-up, and count again:

```bash
ddev wp db query "SELECT COUNT(*) AS before_count FROM wp_pc_transactions"

TOKEN=$(ddev wp eval '$u=get_user_by("login","stripe-step3-check");echo \PC\AuthController::issue_access_token($u);' 2>/dev/null | tr -d '\r\n')
curl -sS -X POST https://pusher-coin.ddev.site/wp-json/pc/v1/wallet/topup \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"coin_qty":3,"unit_price":"40.00"}' -w '\nHTTP %{http_code}\n'

ddev wp db query "SELECT COUNT(*) AS after_count FROM wp_pc_transactions"
```

**Expected:**

- `HTTP 500` with `{"code":"stripe_not_configured", …}`
- **`before_count` and `after_count` are the same number.**

> **If a new record appeared, that is a defect** — the server would be leaving orphaned
> pending charges whenever it is misconfigured.

Now remove the `//` to restore the line.

---

## 6. Negative check — the old LiqPay setting is gone

```bash
ddev wp option get pc_liqpay_public_key
```

**Expected:** an error saying the option could not be found. It must **not** print a
value (or an empty line).

```bash
ddev wp option get pc_db_version
```

**Expected:** `1.9.0`.

You can also confirm the upgrade removes it from an older install:

```bash
ddev wp eval '
add_option("pc_liqpay_public_key","leftover");
update_option("pc_db_version","1.8.0");
\PC\Install_Schema::maybe_install();
echo ( false === get_option("pc_liqpay_public_key", false) ? "removed" : "STILL THERE" ) . ", version " . get_option("pc_db_version") . "\n";
'
```

**Expected:** `removed, version 1.9.0`.

---

## 7. Negative check — no secrets were committed

```bash
cd ~/Projects/full-stack/pusher-coin
git -C backend ls-files wp-config.php | wc -l
```

**Expected: `0`** — the file holding your keys is not tracked by git at all.

```bash
git -C backend log -p stripe/sprint-1..HEAD | grep -c "$(stripe config --list | awk -F"'" '/test_mode_api_key/ {print $2; exit}')"
```

**Expected: `0`** — your actual key appears in no commit.

---

## 8. The other two apps were not touched

```bash
git -C frontend log --oneline stripe/sprint-1 -1
git -C admin log --oneline stripe/sprint-1 -1
```

**Expected:** both still show the Step 2 merge — this step changed `backend/` and
documentation only.

---

## When you are done

```bash
cd ~/Projects/full-stack/pusher-coin/backend && ddev stop
```

**If everything above matched, the step passes.** If any check did not match, say which
number failed and what you saw — that goes to `/fix-step`, not a direct patch.

## Known and accepted, not faults

- **The player's top-up button throws an error.** The browser side is Step 5. Verified
  here by calling the server directly.
- **Nothing can be paid successfully yet.** The webhook that credits a wallet is Step 4.
  A payment made now would take money in the Stripe sandbox and credit nothing.
- **`docs/DATA-MODEL.md` still says payments complete "through the LiqPay callback"** —
  because they still do. Step 4 ships the replacement and owns that rewording.
- **The test player `stripe-step3-check` and one `pending` row stay in your local
  database.** Local only, never deployed. Remove them if you like:
  `ddev wp user delete stripe-step3-check --yes`.
- **`ddev start` will drop the two Stripe lines from `wp-config.php`.** A known DDEV
  quirk already recorded in `docs/LEARNINGS.md`.
