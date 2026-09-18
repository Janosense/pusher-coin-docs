# Verification — `stripe` Sprint 2, Step 2: The Top-ups screen and the Stripe badge in Settings

**What this step did.** The admin app now has a **Top-ups** screen and a live Stripe
status line in **Settings**.

- **Top-ups** lists every top-up. It has filter tabs and page controls, and it is
  read-only: nothing on it can change a top-up.
- **Settings** now says whether Stripe is set up, and whether the keys are `test` or
  `live`. It turns red when Stripe is not configured.

Both read from the two server addresses Step 1 added.

**What the agent could not check.** The agent confirmed that `/topups` exists and sends
a signed-out visitor to the sign-in page. It did **not** see the screens signed in:
that would have meant putting an access token into the browser, which it does not do.
Everything from §2 onwards is checked here by you for the first time.

Nothing is deployed. The admin app runs only on your computer, against your local site.

---

## 0. Before you start

```bash
cd ~/Projects/full-stack/pusher-coin/backend
ddev start
git status --porcelain
git branch --show-current
```

**Expected:** no output from `git status`. If `M wp-config-ddev.php` appears, run
`git checkout -- wp-config-ddev.php`. The branch is `main`.

```bash
cd ~/Projects/full-stack/pusher-coin/admin
git branch --show-current
```

**Expected:** `main`.

Start the admin app. If it is already running from an earlier session, stop it with
`Ctrl+C` and start it again, so it serves `main`:

```bash
npm run dev
```

**Expected:** a line with `http://localhost:5174/`. Leave this terminal open.

---

## 1. Negative: a signed-out visitor cannot reach Top-ups

Open a **private / incognito** browser window and go to:

```
http://localhost:5174/topups
```

**Expected:** you land on the admin **sign-in** form, and the address bar reads
`http://localhost:5174/sign-in?redirect=/topups`. No top-up data is shown.

Close the private window.

---

## 2. Sign in as the admin

In a normal browser window, open `http://localhost:5174/sign-in`. Sign in with your
admin account (user `tymofii.synianskyi`) and its password.

The app asks for a **6-digit code**. Locally, emails are caught by Mailpit. Open
`https://pusher-coin.ddev.site:8026`, open the newest email, and type the code in.

**Expected:** you are signed in and see the admin header.

---

## 3. The menu has Top-ups

**Expected:** the menu along the top reads, in this order:

**Rooms · Withdrawals · Top-ups · Machine · Support · Chat · Settings**

Click **Top-ups**.

---

## 4. The full list: old and new top-ups side by side

**Expected:**
- **Heading:** `Top-ups`.
- **Tabs on the right:** `All`, `Pending`, `Completed`, `Failed`. **All** is highlighted
  with a blue outline.
- **Columns:** `PLAYER`, `AMOUNT (UAH)`, `COINS × PRICE`, `STATUS`, `REFERENCE`,
  `CREATED`, `SETTLED`, `NOTES`.
- **Exactly 4 rows**, newest first. All belong to the local test player
  `stripe-step3-check` (email `stripe-step3-check@example.com` under the name):

| # | Amount | Coins × price | Status (pill colour) | Reference | Settled | Notes |
|---|---|---|---|---|---|---|
| 1 | ₴80.00 | 2 × ₴40.00 | FAILED (red) | `pc-topup-268` | — | local verification fixture |
| 2 | ₴80.00 | 2 × ₴40.00 | FAILED (red) | — | — | Stripe: stripe_call_failed |
| 3 | ₴80.00 | 2 × ₴40.00 | COMPLETED (green) | `cs_test_a1GDD1iS6t7…` (long, wraps onto a second line) | a date and time | — |
| 4 | ₴120.00 | 3 × ₴40.00 | PENDING (yellow) | `cs_test_a1rqvK7CJiV…` (long, wraps) | — | — |

- **Row 1 is the LiqPay-era stand-in.** Your local database had no real LiqPay top-up,
  so Step 1 created this one row with an old-style `pc-topup-…` reference. It moved no
  coins. It sits next to Stripe's `cs_test_…` references, which is the point of the
  screen.
- Amounts show exactly two decimals (`₴80.00`), never `₴80` or `₴80.0`.
- The **Created** column shows a date and time on every row, in your computer's local
  time.

---

## 5. The tabs change the list

Click each tab in turn:

| Click | Expected highlighted tab | Expected rows |
|---|---|---|
| **Failed** | Failed | exactly 2: row 1 (`pc-topup-268`) and row 2 (no reference) |
| **Pending** | Pending | exactly 1: the `₴120.00` row |
| **Completed** | Completed | exactly 1: the `₴80.00` row with a Settled date |
| **All** | All | all 4 again |

Only one tab is highlighted at a time: the one you clicked last.

---

## 6. Negative: nothing on the screen can change a top-up

On **All**, look along every row.

**Expected:**
- There is no **Approve**, **Refund**, **Reject**, **Block** or any other button in
  any row, and no actions column.
- Clicking anywhere on a row does nothing. No dialog opens and the page does not change.
- The only buttons on the page are the four tabs.
- There are **no** page controls (`Prev`, `Page 1 of 1`, `Next`) under the table.
  They appear only when a tab has more than 50 top-ups, and you have 4.

---

## 7. The other screens still work

This step added to the shared menu and page router, and to the Settings screen. Check
that nothing else moved:
- Click **Withdrawals**. It opens as before, with its own tabs and list.
- Click **Settings**. The **Coin pricing** and **Bonus mapping** sections still show
  their values.

---

## 8. Settings: Stripe reads configured, on test keys

Still on **Settings**, scroll to the **Stripe** section.

**Expected:**
- Directly under the heading `Stripe` there is a **green** line reading
  **`Configured · test`**.
- Below it are the two paragraphs about `wp-config.php` and the webhook address, as
  before.
- No key and no part of a key appears anywhere on the page.

---

## 9. Settings: red when a Stripe setting is missing, then back

This temporarily switches off one Stripe setting in your **local** `wp-config.php`.
That file is not tracked by git, so this can never reach a commit. The commands add and
remove `// ` at the start of one line, and never print the key.

In a **new terminal** (leave `npm run dev` running):

```bash
cd ~/Projects/full-stack/pusher-coin/backend
sed -i '' "s|^define( 'PC_STRIPE_WEBHOOK_SECRET',|// &|" wp-config.php
grep -c "^// define( 'PC_STRIPE_WEBHOOK_SECRET'," wp-config.php
```

**Expected:** `1`.

Go back to the browser and **reload** the Settings page.

**Expected:** the line under `Stripe` is now **red** and reads **`Not configured`**.

**Put it back. Do not skip this:**

```bash
sed -i '' "s|^// \(define( 'PC_STRIPE_WEBHOOK_SECRET',\)|\1|" wp-config.php
grep -c "^define( 'PC_STRIPE_" wp-config.php
```

**Expected:** `2`.

**Reload** Settings again. **Expected:** green **`Configured · test`**.

---

## Done when

- [ ] §1 a signed-out visitor to `/topups` lands on sign-in
- [ ] §3 the menu shows **Top-ups** right after **Withdrawals**
- [ ] §4 four rows: `pc-topup-268` next to two `cs_test_…` references, amounts like `₴80.00`
- [ ] §5 each tab highlights itself; **Failed** shows exactly 2 rows
- [ ] §6 no button or action anywhere in the rows, and no page controls with 4 rows
- [ ] §7 Withdrawals and Settings' other sections still work
- [ ] §8 Settings shows a green `Configured · test`
- [ ] §9 with the webhook secret switched off it turns red `Not configured`, and green again once restored

If any of these fails, report it with `/fix-step <what failed>`.
