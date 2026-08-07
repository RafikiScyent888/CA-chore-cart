# Firebase setup — step by step

This does two things:

1. Makes the parent's phone and the kid's phone show the **same** data
2. Puts a **real login** on it, so only your accounts can read or change it

It's free. Takes about fifteen minutes. You need a Google account.

**Do the steps in order.** Step 2 must come before step 4, or the config you copy
will be missing a line and nothing will work.

---

## Step 1 — Make a Firebase project

1. Go to **https://console.firebase.google.com**
2. Sign in with your Google account
3. Click **Create a project** (or **Add project**)
4. Name it anything — `chore-cart` is fine
5. It offers **Google Analytics**. Turn it **off**. You don't need it.
6. Click **Create project**, wait, then **Continue**

---

## Step 2 — Create the database

This is the step people get wrong. There are two different databases in Firebase
and you need the **Realtime Database**, *not* Firestore.

1. Left sidebar → **Build** → **Realtime Database**

   ⚠️ Not "Firestore Database". If the page says Firestore, go back.
2. Click **Create Database**
3. Pick a location close to you (e.g. `us-central1`). This can't be changed later.
4. When it asks about rules, choose **Start in locked mode**.

   Locked is correct — we write the real rules in step 6. (If you only see "test
   mode", pick that; step 6 replaces it either way.)
5. Click **Enable**

---

## Step 3 — Turn on email logins and create your accounts

1. Left sidebar → **Build** → **Authentication** → **Get started**
2. **Sign-in method** tab → click **Email/Password**
3. Turn on the **first** toggle (Email/Password).

   Leave "Email link (passwordless sign-in)" **off**.
4. **Save**

Now create the accounts. The app never lets anyone sign themselves up — you make
the accounts here, which is what keeps strangers out.

5. Go to the **Users** tab → **Add user**
6. Create one for yourself, e.g. `parent@yourfamily.com` with a strong password

   It does **not** have to be a real email address that exists — but use a real
   one for the parent if you ever want "Forgot password" to work.
7. Click **Add user** again and make one for your kid, e.g. `kid@yourfamily.com`
8. You'll now see both accounts listed, each with a **User UID** — a long string
   like `k3Jd8sPqR2WvXyZ1aBcDeFgHiJk2`.

   **Copy both UIDs somewhere.** You need them in step 6.

   (Hover the row and use the copy icon, or click the ⋮ menu.)

---

## Step 4 — Copy your config

1. Click the **gear icon** ⚙️ (top left) → **Project settings**
2. Scroll to **Your apps**
3. Click the **web icon** — it looks like `</>`

   Not iOS, not Android. The `</>` one.
4. Nickname it `chore-cart-web`
5. **Leave "Also set up Firebase Hosting" unchecked**
6. Click **Register app**
7. It now asks how you want to load the SDK: **"Use npm"** or **"Use a `<script>`
   tag"**.

   Choose **`<script>` tag**. The app already loads Firebase this way, and it
   needs no npm and nothing installed. (Either option shows the same config, so
   npm isn't *wrong* — script tag just matches what the app does.)

8. You'll see a block like this:

   ```html
   <script type="module">
     import { initializeApp } from "https://www.gstatic.com/firebasejs/…/firebase-app.js";

     const firebaseConfig = {
       apiKey: "AIzaSy…",
       authDomain: "chore-cart.firebaseapp.com",
       databaseURL: "https://chore-cart-default-rtdb.firebaseio.com",
       projectId: "chore-cart",
       storageBucket: "chore-cart.appspot.com",
       messagingSenderId: "123456789012",
       appId: "1:123…"
     };

     const app = initializeApp(firebaseConfig);
   </script>
   ```

   **Copy only the braces part** — from the `{` after `firebaseConfig =` down to
   its matching `}`, including both braces.

   ⚠️ **Don't copy the rest.** Not the `<script>` tags, not the `import` line, not
   `initializeApp`. The app already has all of that; pasting it in will break the
   file.

✅ **Check it has a `databaseURL` line.** If it doesn't, you did this before step
2. Create the Realtime Database, then copy the config again.

---

## Step 5 — Paste it into the app

1. Open `index.html` in a plain text editor

   - **Windows:** right-click → Open with → Notepad
   - **Mac:** right-click → Open With → TextEdit
   - Or https://vscode.dev in a browser — free, nothing to install

   ⚠️ **Not Word or Pages.** They replace straight quotes `"` with curly ones `"`
   and the file silently stops working.

2. Search for **`STEP 4`** (Ctrl+F / Cmd+F) — there's a big comment marking the spot
3. Just below it, replace the word `null`:

   ```js
   const FIREBASE_CONFIG = {
     apiKey: "AIzaSyD-xxxxxxxxxxxxxxxxxxxxxxxxx",
     authDomain: "chore-cart.firebaseapp.com",
     databaseURL: "https://chore-cart-default-rtdb.firebaseio.com",
     projectId: "chore-cart",
     storageBucket: "chore-cart.appspot.com",
     messagingSenderId: "123456789012",
     appId: "1:123456789012:web:abc123def456"
   };
   ```

   **Keep the semicolon `;` at the end.** Change nothing else.

4. Save

Leave `FAMILY_ID` alone. It's already filled in and must be identical on every phone.

---

## Step 6 — Write the security rules

This is what stops your kid approving their own chores. Take the two UIDs from
step 3 and keep them handy.

1. **Build → Realtime Database → Rules** tab
2. Delete everything in the box
3. Paste the block below
4. **Replace all four placeholders** — `PARENT_UID` appears twice and `KID_UID`
   appears twice. Use Find & Replace if your browser offers it.

```json
{
  "rules": {
    "families": {
      "$fid": {
        ".read": "auth != null && ($fid === 'fam-219ncuyri2xossyhxr5lsynl') && (auth.uid === 'PARENT_UID' || auth.uid === 'KID_UID')",
        ".write": "auth != null && ($fid === 'fam-219ncuyri2xossyhxr5lsynl') && auth.uid === 'PARENT_UID'",

        "log": {
          "$date": {
            "$chore": {
              ".write": "auth != null && auth.uid === 'KID_UID' && $date.matches(/^[0-9]{4}-[0-9]{2}-[0-9]{2}$/) && root.child('families').child($fid).child('config/chores').hasChild($chore)",

              ".validate": "newData.hasChildren(['at','status','amount','title'])",

              "status": {
                ".validate": "newData.isString() && (auth.uid === 'PARENT_UID' || newData.val() === 'pending' || (newData.val() === 'approved' && root.child('families').child($fid).child('config/settings/requireApproval').val() === false))"
              },

              "amount": {
                ".validate": "newData.isNumber() && (auth.uid === 'PARENT_UID' || newData.val() === root.child('families').child($fid).child('config/chores').child($chore).child('amount').val())"
              },

              "title": { ".validate": "newData.isString() && newData.val().length < 120" },
              "at":    { ".validate": "newData.isString() && newData.val().length < 40" },
              "late":  { ".validate": "newData.isBoolean()" },

              "$other": { ".validate": false }
            }
          }
        }
      }
    }
  }
}
```

5. Click **Publish**

### What each part does

| Rule | Effect |
| --- | --- |
| `.read` | Only your two accounts can see the family at all |
| `.write` at the top | Only the parent can change chores, settings, the PIN or the money |
| `.write` under `log` | The kid may only add or remove **their own completions** |
| `status` validation | The kid can only write `pending` — never `approved` |
| `amount` validation | The kid's entry must match the chore's real price |
| `$other` | Any field the app doesn't recognise is rejected |

The one exception is deliberate: if you switch **"Chores need my approval" off**
in the app, the kid's completions count immediately. The rules allow that,
because you asked for it — and the kid can't flip that switch themselves, since
settings are parent-only.

**Important:** do the setup on **your own phone first**. Whoever signs in first
claims the family as parent.

---

## Step 7 — Test it

1. Open your edited `index.html`
2. You should get a **Sign in** screen. Nothing else is visible — that's correct.
3. Sign in with the parent account
4. Check the **bottom of the page**:

   - **"Synced · parent@yourfamily.com"** → working 🎉
   - **"No access — check the database rules"** → the UIDs in step 6 don't match
   - **"Saved on this device"** → the config didn't parse, see troubleshooting

5. Tick a chore. In Firebase → **Realtime Database → Data**, you should see
   `config`, `ledger` and `log` appear under your family.

Now put the **same file** on the kid's phone and sign in with the *kid* account.
Their phone should show the chore list with **no Parent button at all**. Tick
something there, and it turns up in your Review tab awaiting approval.

---

## Step 8 — Change the PIN and pick a theme

Two things to do once you're signed in, before the phone leaves your hands.

**Change the PIN.** It ships as `1234`, which is the first thing anyone tries —
the app will nag you until you change it.

*Parent → Rules → Parent PIN → Change PIN.* It asks for the current PIN, then
the new one twice.

- Obvious codes are refused: no `1111`, no runs like `1234` or `4321`
- Five wrong tries locks the keypad for a minute, and closing the app doesn't
  reset it, so guessing four digits is slow work
- The PIN is never displayed on screen

The PIN guards the *parent screen*. Your database rules are what guard the
*data* — a kid can't approve their own chores even if they learn the PIN.

**Pick a theme.** The ◐ button in the top bar cycles light, dark, and
follow-your-phone. It's stored per device, so you and your kid can each choose
your own — it isn't part of the family data and doesn't sync.

---

## Troubleshooting

**No sign-in screen at all, footer says "Saved on this device"**
The config didn't parse. Press F12 → Console and look for a red error. Usually a
missing `;`, a missing `}`, or curly quotes from editing in Word.

**"Email or password isn't right"**
Retype it. If you're stuck, Authentication → Users → ⋮ → **Reset password**.

**"Email sign-in isn't switched on in Firebase yet"**
You skipped step 3. Turn on the Email/Password provider.

**"No access — check the database rules"**
You're signed in, but that account isn't in the rules. The UID in your rules must
match the one in Authentication → Users **character for character**. This is the
most common mistake — copy/paste, don't retype.

**"Only a parent can change that" on the kid's phone**
Expected if they somehow reached a parent control — the server refused it. If it
appears when they merely tick a chore, the `log` section of your rules has the
wrong `KID_UID`.

**The kid's phone shows a Parent button**
It signed in before yours did and claimed the family. Fix: in Firebase →
Realtime Database → Data, open `config/parentUid` and set it to the parent's UID.

**One phone shows different data**
That phone has an older copy of the file, or a different `FAMILY_ID`.

---

## What this does and doesn't protect

**Now genuinely private.** No login, no data — full stop. Someone who gets the
file gets a sign-in screen and nothing else. Firebase API keys are public by
design; that's fine, because the rules do the real work.

**Signing out wipes the phone's copy.** Parent → Rules → Sign out clears the
local cache. The data stays safe in the cloud.

**Your kid cannot approve their own chores.** Not through the app, and not by
signing into the Firebase website and editing the database by hand — the rules
reject it at the server. Their account can add a completion marked *pending* and
nothing more. They also can't change a chore's price, alter settings, change the
PIN, give themselves a bonus, delete a penalty, or take over the family.

The parent screen doesn't even appear on their phone.

**What's still worth knowing:** whoever signs in first claims the family as
parent, so do the setup on your own phone before handing anything over. And a
parent who shares their own password gives away everything — the rules identify
people by account, not by device.

**Still don't** put addresses, school details, or anything sensitive in the notes.

---

## Cost

Free. The Spark plan covers 100 simultaneous connections and 1 GB stored. This
app uses a few kilobytes and two phones. No card required, no bill.
