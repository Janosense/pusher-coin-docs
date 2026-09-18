# Verification — `stripe` Sprint 2, Step 1: `GET /admin/topups` and `GET /admin/stripe/status`

**What this step did.** It added two new admin-only addresses to the server. Nothing on
any screen has changed yet. The admin **Top-ups** screen and the Stripe badge in
**Settings** are Step 2, and they will read from these two addresses.

- **The top-ups list** (`/admin/topups`) returns every top-up ever made, newest first,
  and can be filtered by status. It is read-only. Old LiqPay-era rows and new Stripe
  rows appear side by side.
- **The Stripe status** (`/admin/stripe/status`) says only whether Stripe is set up
  and whether the keys are `test` or `live`. It never shows a key.

Nothing is deployed. Both addresses reach production only when Sprint 2 is merged into
`main` and `backend` `main` is pushed. So this guide calls your **local** site directly
from the terminal.

---

## 0. Before you start

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev start
git status --porcelain
```

**Expected:** no output from `git status`. If `M wp-config-ddev.php` appears, run
`git checkout -- wp-config-ddev.php` (a known `ddev start` quirk, see
`docs/LEARNINGS.md`).

```bash
git branch --show-current
```

**Expected:** `stripe/sprint-2`.

Check that the Stripe keys are still in your local config. This prints a count, never
the keys:

```bash
grep -c "^define( 'PC_STRIPE_" wp-config.php
```

**Expected:** `2`.

---

## 1. Get two passes: one admin, one player

Every command below uses these, so run this block first, in the same terminal window:

```bash
API=https://pusher-coin.ddev.site/wp-json/pc/v1
ADMIN="Authorization: Bearer $(ddev wp eval 'echo \PC\AuthController::issue_access_token( get_user_by( "id", 1 ) );' 2>/dev/null | tr -d '\r\n')"
PLAYER="Authorization: Bearer $(ddev wp eval 'echo \PC\AuthController::issue_access_token( get_user_by( "id", 50 ) );' 2>/dev/null | tr -d '\r\n')"
SUMMARY='import json,sys; d=json.load(sys.stdin); print("total:", d["total"]); [print(" ", i["id"], i["status"], i["external_ref"]) for i in d["items"]]'
```

- User 1 is your admin account (`tymofii.synianskyi`).
- User 50 is the local test player `stripe-step3-check`, who owns all the local top-ups.
- The passes expire after 15 minutes. If a command later answers `401` when it should
  not, run this block again.

---

## 2. The full list shows old and new top-ups together

```bash
curl -sk -H "$ADMIN" "$API/admin/topups" | python3 -c "$SUMMARY"
```

**Expected:** exactly these four lines under `total: 4`, in this order (newest first):

```
total: 4
  268 failed pc-topup-268
  197 failed None
  95 completed cs_test_a1GDD1iS6t7mCRbd0qYTSxZIPH5R6UQXJ4gQdY3Jis889SXLQah6CNq8tt
  59 pending cs_test_a1rqvK7CJiVyPreIVeH59tAqpyefTmwCC6wbGa82sr8hY4vb6lGWmjIxhc
```

- **#268 has an old LiqPay-style reference (`pc-topup-268`).** Your local database had
  no real LiqPay-era top-up, so this step created this one row as a stand-in. It is
  marked `failed` and moved no coins. Its note reads `local verification fixture`. It
  can stay.
- **#95 and #59 have Stripe references (`cs_test_…`).** They are from Sprint 1's test
  payments.
- **#197 shows `None`, meaning no reference.** It is a top-up where Stripe refused to
  open a payment page, so no reference was ever given. That is correct.

To see every field of one row, run the full answer:

```bash
curl -sk -H "$ADMIN" "$API/admin/topups" | python3 -m json.tool
```

**Expected:** every row has the same 12 fields:

`id`, `user_id`, `user_email`, `user_nickname`, `amount_money`, `amount_coins`,
`unit_price`, `status`, `external_ref`, `notes`, `created_at`, `settled_at`

Money appears **in quotes**, for example `"amount_money": "80.00"` and
`"unit_price": "40.00"`, never as a bare `80.0`. `settled_at` has a date only on the
`completed` row (#95).

---

## 3. The status filter narrows the list

```bash
curl -sk -H "$ADMIN" "$API/admin/topups?status=failed"    | python3 -c "$SUMMARY"
curl -sk -H "$ADMIN" "$API/admin/topups?status=pending"   | python3 -c "$SUMMARY"
curl -sk -H "$ADMIN" "$API/admin/topups?status=completed" | python3 -c "$SUMMARY"
curl -sk -H "$ADMIN" "$API/admin/topups?status=all"       | python3 -c "$SUMMARY"
```

**Expected, in order:**
- `failed` → `total: 2`, rows 268 and 197. The LiqPay-style row and the Stripe-era
  row both appear.
- `pending` → `total: 1`, row 59.
- `completed` → `total: 1`, row 95.
- `all` → `total: 4`, the same list as section 2.

---

## 4. Paging

```bash
curl -sk -H "$ADMIN" "$API/admin/topups?per_page=1&page=2" | python3 -c "$SUMMARY"
```

**Expected:** `total: 4` and exactly one row, `197 failed None`: the second-newest,
shown one per page.

---

## 5. A wrong filter is refused

```bash
curl -sk -H "$ADMIN" -w '\nHTTP %{http_code}\n' "$API/admin/topups?status=refunded"
```

**Expected:**

```
{"code":"invalid_transaction_status","message":"status must be one of: pending, completed, failed, all.","data":{"status":400}}
HTTP 400
```

`refunded` is refused on purpose, because only a withdrawal can ever be `refunded`.
Any made-up word (try `?status=banana`) gives the same answer.

---

## 6. Negative: nobody but an admin can read either address

Without any pass:

```bash
curl -sk -w '\nHTTP %{http_code}\n' "$API/admin/topups"
curl -sk -w '\nHTTP %{http_code}\n' "$API/admin/stripe/status"
```

**Expected:** both answer `rest_forbidden` with `"Authentication required."` and
**`HTTP 401`**.

With the **player's** pass:

```bash
curl -sk -H "$PLAYER" -w '\nHTTP %{http_code}\n' "$API/admin/topups"
curl -sk -H "$PLAYER" -w '\nHTTP %{http_code}\n' "$API/admin/stripe/status"
```

**Expected:** both answer `rest_forbidden` with `"Administrator privileges required."`
and **`HTTP 403`**. No top-up data appears.

---

## 7. Negative: the list cannot change anything, even for an admin

These try to *write* with the admin's pass:

```bash
curl -sk -X POST -H "$ADMIN" -w '\nHTTP %{http_code}\n' "$API/admin/topups"
curl -sk -X POST -H "$ADMIN" -w '\nHTTP %{http_code}\n' "$API/admin/topups/95"
curl -sk -X PUT  -H "$ADMIN" -w '\nHTTP %{http_code}\n' "$API/admin/stripe/status"
```

**Expected:** all three answer `rest_no_route` with **`HTTP 404`**. There is no way to
approve, refund, block or edit a top-up, and no way to change Stripe settings through
the server. That is by design.

---

## 8. The Stripe status, configured

```bash
curl -sk -H "$ADMIN" "$API/admin/stripe/status"; echo
```

**Expected, exactly:**

```
{"configured":true,"mode":"test"}
```

Nothing else is in the answer: no key, no part of a key. To be sure:

```bash
curl -sk -H "$ADMIN" "$API/admin/stripe/status" | grep -c 'sk_\|whsec_'
```

**Expected:** `0`.

---

## 9. The Stripe status with a setting removed, then put back

This temporarily switches off one of the two Stripe settings in your **local**
`wp-config.php`. That file is not tracked by git, so this can never reach a commit. The
commands add and remove `// ` at the start of one line. They never print the key.

**Switch the webhook secret off:**

```bash
sed -i '' "s|^define( 'PC_STRIPE_WEBHOOK_SECRET',|// &|" wp-config.php
grep -c "^// define( 'PC_STRIPE_WEBHOOK_SECRET'," wp-config.php
curl -sk -H "$ADMIN" "$API/admin/stripe/status"; echo
```

**Expected:** the `grep` prints `1`, and then:

```
{"configured":false,"mode":null}
```

`mode` is `null` even though the secret key itself is still there. With only one of the
two settings present, the server cannot take a payment, so it reports no mode at all.

**Put it back. Do not skip this:**

```bash
sed -i '' "s|^// \(define( 'PC_STRIPE_WEBHOOK_SECRET',\)|\1|" wp-config.php
grep -c "^define( 'PC_STRIPE_" wp-config.php
curl -sk -H "$ADMIN" "$API/admin/stripe/status"; echo
```

**Expected:** the `grep` prints `2`, and the answer is back to
`{"configured":true,"mode":"test"}`.

*Optional:* the same works with the other setting. Use `PC_STRIPE_SECRET_KEY` in place
of `PC_STRIPE_WEBHOOK_SECRET` in the three commands above: switch it off, check the
answer, then put it back. The answer is again `{"configured":false,"mode":null}`, then
`{"configured":true,"mode":"test"}` once restored. Always finish with
`grep -c "^define( 'PC_STRIPE_" wp-config.php` printing `2`.

---

## Done when

- [ ] §2 lists four top-ups, including `pc-topup-268` next to two `cs_test_…` rows
- [ ] §3 `?status=failed` shows exactly 268 and 197
- [ ] §5 a wrong filter answers `HTTP 400`
- [ ] §6 no pass → `401`; player pass → `403`, on both addresses
- [ ] §7 every write attempt → `404`
- [ ] §8 status reads `{"configured":true,"mode":"test"}` with no key in it
- [ ] §9 with one setting removed it reads `{"configured":false,"mode":null}`, and it is
      back to `test` after restoring (`grep` prints `2`)

If any of these fails, report it with `/fix-step <what failed>`.
