# Quickstart: Mobile View

**Feature**: [spec.md](./spec.md) | **Date**: 2026-10-31

How to run and check the phone layout. Setting up Auth0, running the backend and deploying are
unchanged; see `specs/003-spending-screens/quickstart.md` and
`specs/004-csv-import/quickstart.md`.

Only `hl-budget-frontend` changes. The backend needs no new version.

## 1. Run locally

```sh
# Backend as in 003's quickstart (Postgres + ./gradlew bootRun), then:
cd hl-budget-frontend
pnpm install
pnpm dev            # http://localhost:3000
```

Open http://localhost:3000, sign in with the test user, then open the browser's device
toolbar (Chrome DevTools → Toggle device toolbar) and pick **iPhone 13** (390×844). Reload
once so the phone layout loads from the start.

## 2. Automated gates

```sh
pnpm typecheck && pnpm lint && pnpm test   # constitution gate (unit tests incl. new phone ones)
pnpm e2e --project=chromium                 # existing desktop journeys: must pass unchanged
pnpm e2e --project=phone                    # new phone journeys (e2e/phone/)
```

Expected:
- All three pass.
- `git diff origin/main --stat -- e2e/*.spec.ts` lists no existing desktop spec file (SC-005).
- The phone specs need `E2E_USERNAME` / `E2E_PASSWORD`, as the desktop ones do.

## 3. By-hand phone checks (390px, then a real phone at https://budget.valyalkin.com)

Each step names the user story and requirement it proves.

1. **Month view (US1)**: the current month and year are selected; total and count match; rows
   show "Oct 27"-style dates, category and amount. In the Month dropdown, future months are
   greyed. Pick a past month: the list reloads. Pick December of last year, then switch the
   Year to the current year: the month becomes the current month.
2. **Add (US2)**: tap "Add spending". The amount keypad has a decimal point. Tap Date: the
   calendar is inline, future days are disabled, and Today works. Type "nei" in Merchant and
   pick a past merchant: Category and Account fill in. Clear the amount and tap "Add spending":
   you see "Enter an amount greater than zero." Fill it in and save: you return to the list in
   the spending's month.
3. **Edit and delete (US3)**: tap a row, change the amount, save. Tap it again, then "Delete
   spending": the sheet shows merchant · amount · date. Cancel keeps it. Delete removes it and
   shows "Deleted …".
4. **By category (US4)**: in a month with more than 6 categories, Table lists all with a
   100% total, and Pie shows 6 slices plus "Everything else". The choice stays when the month
   changes. An empty month shows "Nothing to break down in …".
5. **Menu and Settings (US5)**: the user icon opens the sheet. In Settings → Categories, add
   "Coffee"; "coffee" is refused; remove "Coffee". Defaults have no remove button. In Accounts,
   the counts show, removing an account in use is refused, and removing the last account is
   refused. Sign out shows "You're signed out".
6. **Import (US6)**: menu → Import CSV. Picking a `.png` shows the not-CSV error. Pick a file
   with problem rows (generate one with `node e2e/fixtures/make-csv.mjs`, or use a CSV with
   empty fields, a `"Lakeshore Fuel, Hwy 7"` value, `-18.90` and `10/27/2026`). The banner
   counts rows needing attention, `10/27/2026` shows "No date", and Add is disabled. Tap a row:
   "Fix row"; "Save row" refuses until the row is complete; "Remove row" removes at once. When
   all rows are good, "Add N spendings" lands on the latest month with "Added N spendings from
   …".
7. **Back gesture (FR-103)**: on Edit, Settings, Import, Review, the menu and the delete sheet,
   the phone's back gesture closes only that layer (Review → Import CSV). Reload on any screen
   shows the Spendings list.
8. **Touch and zoom (FR-004–FR-007)**: on an iPhone, tapping any input does not zoom. There is
   no sideways scrolling at 320px (DevTools → iPhone SE). The floating button stays visible
   while scrolling and doesn't cover the last row. The bottom button on Edit stays above the
   keyboard.
9. **Rotation (Edge Cases)**: rotate to landscape on a phone wider than 640px. The desktop
   layout shows, with the same month.

## 4. Desktop regression check (SC-005)

Before merging, compare the desktop at 1280px and 1440px against production:

1. Open https://budget.valyalkin.com (current production) and the local branch build side by
   side, signed in as the same user, on the same month.
2. Screenshot the Spendings screen, an open entry row, the Settings window, the Import CSV
   window and the Review import window at both widths.
3. The screenshots must match. Keyboard entry (Tab/Enter autofill, arrows, Escape), hover
   edit/remove and inline delete must behave as before.

## 5. Deploy

As usual: merge the frontend PR → release workflow → hl-helm PR → Argo CD. Then repeat §3 on a
real phone over the home network and over Tailscale. Update the spec README's Features table.
