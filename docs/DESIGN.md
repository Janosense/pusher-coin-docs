# Design — Pusher Coin

<!-- Adoption mode: documented from the code as-is. There are no design files —
     nothing was designed in Claude Design or in chat, so
     docs/features/core/design/ is empty and every Design ref below points at the
     component that defines the screen. Shared design truth for all features:
     tokens, components, the screen list. -->

## Design system source
- **Made in:** documented from code (Adoption). No design tool, no exports, no artboards.
- **Source of truth in code:** two independent systems that share nothing.
  - Player SPA — `frontend/src/assets/main.css`, which imports
    `styles/colors.css` (tokens) and `styles/blocks/*.css` (`body`, `wrapper`,
    `button`, `content`, `view-holder`, `form`). BEM-ish block classes, plus
    scoped `<style>` in each SFC.
  - Admin SPA — `admin/src/assets/main.css`, a single file holding its own token
    set and element resets. No block files, no shared package with the player SPA.
- **Strings:** there is no brief and no artboard, so the code is the only source.
  A string change is a code change.

## Tokens

**Player SPA** (`frontend/src/assets/styles/colors.css`) — dark, gold-on-navy:

| Token | Value | Used for |
|---|---|---|
| `--purple-dark` | `#161b2a` | Page background |
| `--purple` | `#20273d` | Surfaces, panels |
| `--purple-light` | `#8c98a9` | Muted text, labels |
| `--yellow` | `#ffc701` | Primary action, links, accent |
| `--sand` | `#d38d00` | Accent, secondary gold |
| `--black` | `#131620` | Text on gold |
| `--white` | `#ffffff` | Body text |

Type: **Oswald**, 16px base, `line-height: 1`, uppercase for buttons and headers.
Layout: `min-width: 320px`; the side nav takes an 88px body inset from 768px and
100px from 1920px; `.wrapper` pads 20 / 32 / 48px at 0 / 768 / 1440px; forms are
full width, 400px from 1024px, 540px from 1920px. Radius 6px on buttons.
Breakpoints in use: 768, 1024, 1440, 1920.

**Admin SPA** (`admin/src/assets/main.css`) — neutral dark, blue accent:
`--bg #0f1115`, `--surface #161a22`, `--surface-elevated #1e2330`,
`--border #2a2f3d`, `--text #e6e8ee`, `--text-muted #8a8f9d`, `--primary #5b9bff`,
`--primary-hover #76aaff`, `--danger #f87171`, `--success #4ade80`,
`--warning #facc15`. System font stack, 14px base, `color-scheme: dark`.

## Components

| Component | Variants / states | In code |
|---|---|---|
| Button | `.button`, `.button--yellow`; hover widens letter-spacing; `:disabled` | `frontend/src/assets/styles/blocks/button.css` |
| Form | header, item, textfield + label, required marker, error, counter | `frontend/src/assets/styles/blocks/form.css` |
| Modal overlay | open / closed | `frontend/src/components/ModalOverlay.vue` |
| Logout confirm | — | `frontend/src/components/LogoutConfirmModal.vue` |
| App navigation | collapsed / expanded (`NavigationToggle`) | `frontend/src/components/AppNavigation.vue` |
| Room status badge | `available`, `maintenance`, `unavailable` | `frontend/src/components/RoomStatusBadge.vue` |
| Room list / card | loading, empty, loaded | `frontend/src/components/RoomList.vue` |
| Live stream | iframe / `<video>` / HLS; loading, error, offline | `frontend/src/components/LiveStream.vue` |
| Room queue | empty, waiting, you-are-next, your-turn | `frontend/src/components/RoomQueue.vue` |
| Place bet | idle, disabled (not your turn / no coins / **machine out of service** — the relay opened at the venue, which also replaces the phase note with a line saying the turn is kept), submitting | `frontend/src/components/PlaceBet.vue` |
| Room chat | guest read-only, muted, sending, rate-limited | `frontend/src/components/RoomChat.vue` |
| User controls | balance, per-turn winnings, theme-song toggle | `frontend/src/components/UserControls.vue` |
| Next broadcast countdown | scheduled / no upcoming window | `frontend/src/components/NextBroadcastCountdown.vue` |
| Faceless avatar | deterministic tile from the user id | `frontend/src/components/FacelessAvatar.vue` |
| Replenishment balance | coin-quantity input (IMask), price hint, validation | `frontend/src/components/ReplenishmentBalance.vue` |
| Withdrawal request | idle, one-pending lock, submitting | `frontend/src/components/WithdrawalRequest.vue` |
| Sign-in / sign-up form | credentials step, code step, error | `frontend/src/components/SignInForm.vue`, `SignUpForm.vue` |
| Google / Apple button | hidden while unconfigured | `frontend/src/components/GoogleSignInButton.vue`, `AppleSignInButton.vue` |
| Language switcher | present, not wired to an i18n library | `frontend/src/components/LanguageSwitcher.vue` |
| Icons | single-purpose SVG components | `frontend/src/components/icons/` |
| Admin layout | header + nav + slot | `admin/src/components/AdminLayout.vue` |

`frontend/src/components/HelloWorld.vue` is Vite scaffold with no importers.

## Screens

| Screen | App | Route | Design ref | States covered |
|---|---|---|---|---|
| Rooms | player | `/` | `frontend/src/views/RoomsView.vue` | loading, empty, loaded |
| Room | player | `/room/:id` (public) | `RoomView.vue` | guest, player-not-in-queue, in-queue, your-turn, maintenance |
| Sign in | player | `/sign-in` | `SignInView.vue` | credentials, code, error |
| Sign up | player | `/sign-up` | `SignUpView.vue` | form, code, error |
| Choose nickname | player | `/choose-nickname` | `ChooseNicknameView.vue` | gate |
| Accept terms | player | `/accept-terms` | `AcceptTermsView.vue` | gate |
| Confirm email | player | `/confirm-email` | `ConfirmEmailView.vue` | pending, confirmed, expired |
| Account | player | `/account` | `AccountView.vue` | default, `?reason=verify-email` |
| History | player | `/history` | `HistoryView.vue` | empty, paginated |
| Support | player | `/support` (public) | `SupportView.vue` | guest + captcha, guest without captcha, signed-in, unverified banner |
| About | player | not routed | `AboutView.vue` | — |
| Admin sign in | admin | `/sign-in` | `admin/src/views/SignInView.vue` | credentials, code, non-admin rejected |
| Room list | admin | `/rooms` | `RoomListView.vue` | list, empty |
| Room form | admin | `/rooms/new`, `/rooms/:id/edit` | `RoomFormView.vue` | create, edit |
| Room schedule | admin | `/rooms/:id/schedule` | `RoomScheduleView.vue` | rules editor, recomputed next window |
| Withdrawals | admin | `/withdrawals` | `WithdrawalsView.vue` | filter tabs, approve dialog, reject-with-reason |
| Top-ups | admin | `/topups` | `TopupsView.vue` | filter tabs, empty, paged |
| Machine | admin | `/machine` | `MachineView.vue` | online, offline, unauthorized, sensor grid |
| Tickets | admin | `/support/tickets` | `TicketsView.vue` | filtered, expanded, status transitions |
| Subjects | admin | `/support/subjects` | `SubjectsView.vue` | list editor, captcha panel (configured / **unconfigured in red**) |
| Chat moderation | admin | `/chat` | `ChatView.vue` | filters, hide, restore, mute |
| Settings | admin | `/settings` | `SettingsView.vue` | pricing, Stripe hint (constants + webhook URL), Stripe status (configured · test / live; **not configured in red**), 4×3 bonus grid, relay coin count |

## Flows

- **Sign-up → play:** Sign up → Confirm email → Choose nickname → Accept terms → Rooms → Room.
- **Top-up:** Room or Account → Replenishment balance → Stripe's hosted Checkout page (off-site) → back to the SPA; the balance moves when the Stripe webhook settles, not on return.
- **Play a turn:** Room → join queue → wait → your turn → Place bet (one toss at a time) → winnings appear in User controls.
- **Withdraw:** Account → Withdrawal request → (operator) Withdrawals → approve or reject.
- **Support:** Support → subject + description (+ captcha for guests) → (operator) Tickets.

## Out of scope

- No design files exist and none are planned retroactively — `docs/features/core/design/` stays empty until a feature brings a designed screen.
- No shared component library between the two SPAs; the duplication is deliberate.
- No responsive or accessibility pass has been done. No i18n: `LanguageSwitcher` is a control with no library behind it.
- The admin SPA has no operations dashboard; its seven sections are separate screens.
