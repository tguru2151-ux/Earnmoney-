# Earn Money — Firebase Edition

Real Google Sign-In + Email/Password auth, Firestore database, and a
password-protected Admin Panel. Green + white theme. No ads anywhere.

## 1. Local setup

```bash
npm install
npm run dev
```

Open the printed `localhost` URL. Google Sign-In **will not work on
localhost by default** until you complete step 3 below (Authorized
domains) — `localhost` is usually already whitelisted automatically by
Firebase, so it should work immediately for local testing.

## 2. Firestore security rules

In the Firebase Console → **Firestore Database → Rules**, paste the
contents of `firestore.rules` (included in this project) and click
**Publish**. Without this, all reads/writes will be blocked by the
default "production mode" rules and the app will appear broken.

Read the security note at the bottom of `firestore.rules` — the admin
panel currently uses a simple in-app password, not real Firebase admin
roles. Fine for testing; upgrade before handling real payouts.

## 3. Enable Google Sign-In for your deployed domain

1. Firebase Console → **Authentication → Sign-in method** → make sure
   **Google** and **Email/Password** are both enabled (you already did
   this).
2. Firebase Console → **Authentication → Settings → Authorized
   domains** → click **Add domain** → add the domain you deploy to
   (e.g. `earn-money-yourname.vercel.app`). Without this, Google
   Sign-In will fail with an `auth/unauthorized-domain` error on your
   live site.

## 4. Deploy (Vercel — free)

1. Push this folder to a GitHub repo (or drag-and-drop deploy via the
   Vercel dashboard).
2. In Vercel: **New Project → Import your repo**.
3. Build settings are auto-detected (Vite). Just click **Deploy**.
4. Once deployed, copy your `*.vercel.app` URL and add it to Firebase
   **Authorized domains** (step 3 above).

Netlify works the same way — build command `npm run build`, publish
directory `dist`.

## 5. Admin Panel

On the login screen, tap **"Admin Login"** at the bottom, then enter
the password:

```
@Nikhil001
```

You'll see total users, total referrals, total withdrawals, a
searchable user list (name, email, coins, balance, refer info,
withdrawal totals), and an **Accept / Reject** action on every pending
withdrawal. Rejecting automatically refunds the coins & balance to
that user.

## 6. Data model (Firestore collections)

- `users/{email}` — Coins, BalanceINR, ReferCode, ReferredBy,
  TotalEarned, TotalWithdrawn, ReferCount, ReferComplete,
  WithdrawToday, LastDailyClaim, etc.
- `history/{autoId}` — Email, Type, Amount, Date, Status
- `offers/{autoId}` — Title, Coins, Link, Description (auto-seeded on
  first run if empty — edit directly in the Firebase console to add
  your own offers)
- `surveys/{autoId}` — same shape as offers
- `withdrawals/{autoId}` — Email, Type, AmountINR, UPIorEmail, Date,
  Status

## 7. Changing the admin password

Open `src/App.jsx`, find this line near the top:

```js
const ADMIN_PASSWORD = "@Nikhil001";
```

Change the value and redeploy.
