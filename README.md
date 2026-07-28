# CA Chore Cart

Chores and pocket money, tracked on the kid's phone. The parent controls
everything behind a PIN.

One `index.html`. No build step, no npm, no app store. Open it in a browser and
it works.

---

## Getting it onto a phone

**Fastest way — no hosting at all:** email or AirDrop `index.html` to the phone
and open it. Everything saves on that device.

**Proper way — GitHub Pages:**

1. Repo → **Settings** → **Pages**
2. Source: **Deploy from a branch**, branch `main`, folder `/ (root)`
3. Wait a minute, then open the URL Pages gives you
4. On the phone: **Share** → **Add to Home Screen**

It then launches fullscreen with no browser chrome, like a normal app.

The parent PIN starts as **1234**. Change it in Parent → Rules.

---

## Using it

Tap **Parent** (top right), enter the PIN, and you get four tabs:

| Tab | What's there |
| --- | --- |
| **Review** | Approve or send back each finished chore. Post a note that shows on the kid's screen. |
| **Chores** | Add, edit and retire chores — pay, due time, which days, and whether it's required. |
| **Money** | Balance owed, one-tap payout, and manual add/subtract with a reason. |
| **Rules** | Streak rewards, milestone bonuses, deadline grace, late-pay policy, PIN, kid's name. |

The kid's screen shows their balance, a 7-day punch strip for the streak, and
today's chores as tear-off stubs.

### How streaks work

A day counts once **every chore marked "required" for that day** is approved.

- Days with nothing scheduled are **skipped** — a rest day won't break a streak.
- Bonuses pay out on a schedule you set: something small every N days, something
  bigger at each milestone. Set N to 0 to switch a reward off.
- **Breaking a streak doesn't claw back money already earned.** Bonuses that were
  reached stay banked; only the ones never reached are lost.

### Retiring vs deleting

"Retire" hides a chore going forward but keeps it in past records, so old
streaks and totals don't silently rewrite themselves. Retired chores can be
restored.

---

## Syncing the parent's phone with the kid's phone

Out of the box each device keeps its own copy in `localStorage`. To have both
phones see the same data, add Firebase:

1. [console.firebase.google.com](https://console.firebase.google.com) → new project
2. Build → **Realtime Database** → Create
3. Project settings → add a **Web app** → copy the config object
4. In `index.html`, near the top of the `<script>`:

```js
const FIREBASE_CONFIG = {
  apiKey: "...",
  authDomain: "....firebaseapp.com",
  databaseURL: "https://....firebasedatabase.app",
  projectId: "...",
  appId: "..."
};
const FAMILY_ID = "family-1";
```

That's the only change needed — the sync code is already there and switches on
by itself. The footer changes from "Saved on this device" to "Synced across
devices" once it connects. If Firebase can't be reached the app falls back to
local storage instead of breaking.

---

## Security — please read

There is **no authentication**. Anyone with the URL can read and write the
family's data, and the PIN is client-side JavaScript — a speed bump for a
nine-year-old, not real security.

That's fine for chore data. Don't put anything sensitive in it.

To make it real, turn on Firebase **Anonymous Auth** and add database rules
scoped to the family ID, so an unauthenticated request can't read the tree.

---

## Data model

Everything lives in one JSON object.

```
chores[]      { id, title, amount, due, days[], required, createdAt, archivedAt }
log           { "2026-07-28": { choreId: { at, late, status, amount, title } } }
adjustments[] { id, at, amount, reason }
payouts[]     { id, at, amount }
settings      { requireApproval, graceMinutes, latePayPct,
                bonusEvery, bonusAmount, mileEvery, mileAmount }
```

Balance and streak are **never stored** — they're recomputed from the log on
every render, so the two can't drift apart. Each completion snapshots the chore's
title and pay at the moment it was done, which means changing a chore's price
later doesn't rewrite history.
