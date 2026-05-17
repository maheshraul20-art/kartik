# Society Resident Portal

A Next.js + Firebase mobile-first portal for housing-society residents. Built for the Sunshine Heights demo, designed for any society.

**Stack:** Next.js 14 (App Router) · TypeScript · Tailwind · Firebase Auth (Phone OTP) · Firestore (real-time) · Razorpay (stub).

---

## Quick start

```bash
npm install
cp .env.local.example .env.local
# Fill in NEXT_PUBLIC_FIREBASE_* (see "Firebase setup" below)
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

---

## Firebase setup (one-time, ~10 min)

### 1. Create a Firebase project
- Go to [console.firebase.google.com](https://console.firebase.google.com) → **Add project**
- Name it (e.g. `society-saas-dev`). Disable Google Analytics for now if you want.

### 2. Enable Phone Authentication
- **Authentication → Get started → Sign-in method**
- Enable **Phone** provider
- (Optional, recommended for dev) Scroll to **Phone numbers for testing** and add a test number:
  - Phone: `+91 9999999999`
  - OTP code: `123456`
  - This bypasses real SMS quotas while you develop.

### 3. Create Firestore
- **Build → Firestore Database → Create database**
- Pick region **`asia-south1` (Mumbai)** for lowest latency in India
- Start in **production mode** (we'll deploy our own rules below)

### 4. Register a Web app
- Project settings (gear icon) → **Your apps → Add app → Web (`</>`)**
- App nickname: `Resident Portal`. Don't enable hosting yet.
- Copy the `firebaseConfig` object that appears.

### 5. Add config to `.env.local`
Paste these values from the config object into `.env.local`:

```
NEXT_PUBLIC_FIREBASE_API_KEY=AIza...
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
NEXT_PUBLIC_FIREBASE_PROJECT_ID=your-project
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=...
NEXT_PUBLIC_FIREBASE_APP_ID=1:...:web:...
```

### 6. Add authorised domains
- **Authentication → Settings → Authorized domains**
- Add `localhost` (usually already there) and later your Vercel domain (e.g. `your-app.vercel.app`).

### 7. Generate a service-account key (for the seed script only)
- Project settings → **Service accounts → Generate new private key**
- A JSON file downloads. Paste its **entire contents on one line** into `.env.local`:

```
FIREBASE_ADMIN_SDK_JSON={"type":"service_account","project_id":"...",...}
```

> Tip: if your shell complains about escaped newlines in `private_key`, replace literal `\n` with `\\n` in the value.

### 8. Deploy Firestore rules + indexes
Install Firebase CLI once:
```bash
npm install -g firebase-tools
firebase login
firebase use your-project-id
firebase deploy --only firestore:rules,firestore:indexes
```

### 9. Seed test data
```bash
npm run seed
```

This creates:
- A society `sunshine-heights`
- Flat `B-402`
- Resident `Priya Sharma` (linked to test phone)
- 3 invoices (1 pending, 2 paid)
- 2 complaints
- 4 notices
- 3 visitors (1 awaiting your approval)
- Sets custom auth claims `{ role: 'RESIDENT_OWNER', societyId, flatId }` on the test user

### 10. Sign in
```bash
npm run dev
```
- Enter `9999999999`
- OTP: `123456`
- You land on the home dashboard with live data.

---

## What's wired up

| Feature | Status | Notes |
|---|---|---|
| Phone OTP login | ✅ Live | Firebase Auth + invisible reCAPTCHA |
| Auth-state aware routing | ✅ | `/login` ↔ `/home` redirects via `useAuth` |
| Resident profile | ✅ | Real-time Firestore listener |
| Bills list + detail | ✅ | Real-time, filtered to current user |
| Razorpay checkout | ⚠️ Stubbed | Opens Razorpay Checkout when `NEXT_PUBLIC_RAZORPAY_KEY_ID` is set, but order creation is client-side; production needs a Cloud Function (see below) |
| Complaints list + detail + new | ✅ | `createComplaint` writes to Firestore |
| Notices feed | ✅ | Real-time, society-scoped |
| Visitor approve / deny | ✅ | Updates `approvalStatus` (rules permit residents to flip only this field on their own flat's visitors) |
| Sign out | ✅ | |

## What's NOT wired up yet (intentional)

1. **Razorpay order creation & webhook** — Must be a Cloud Function. Client cannot trust-create orders or verify payments. See `lib/firebase.ts` → wire up `createPaymentOrder` and webhook handler as described in the architecture doc.
2. **Image/document uploads** — Storage rules + upload UI for complaint attachments.
3. **Push notifications** — FCM token registration on login + server-side fan-out on notice/invoice creation.
4. **Resident directory** — Profile screen has stub buttons.

---

## Project structure

```
society-resident-portal/
├── app/
│   ├── layout.tsx                  # Root layout, font setup
│   ├── page.tsx                    # /  → redirects to /login or /home
│   ├── globals.css                 # Tailwind base + utility extras
│   ├── (auth)/                     # Route group with auth-redirect layout
│   │   ├── layout.tsx
│   │   └── login/page.tsx
│   └── (resident)/                 # Route group with auth guard + bottom nav
│       ├── layout.tsx
│       ├── home/page.tsx
│       ├── bills/page.tsx
│       ├── bills/[id]/page.tsx
│       ├── complaints/page.tsx
│       ├── complaints/new/page.tsx
│       ├── complaints/[id]/page.tsx
│       ├── notices/page.tsx
│       ├── visitors/page.tsx
│       └── profile/page.tsx
├── components/
│   ├── ui.tsx                      # StatusPill, Card, IconBadge, PageHeader, EmptyState, Skeleton, Toast
│   └── BottomNav.tsx               # Tab bar with active state
├── lib/
│   ├── firebase.ts                 # Singleton init + phone OTP helpers
│   ├── hooks.ts                    # useAuth, useResident, useBills, useComplaints, useNotices, useVisitors
│   ├── types.ts                    # Resident, Invoice, Complaint, etc.
│   └── utils.ts                    # cn(), inr(), formatDate(), toE164()…
├── scripts/
│   └── seed.mjs                    # Seeds the Firestore + auth state
├── firestore.rules                 # Production-grade security rules
├── firestore.indexes.json          # Composite indexes for the queries used by hooks.ts
├── tailwind.config.ts              # Design tokens
└── .env.local.example
```

---

## Deploying to Vercel

```bash
# First time
npx vercel link
npx vercel env add NEXT_PUBLIC_FIREBASE_API_KEY        # paste value, repeat for all NEXT_PUBLIC_FIREBASE_*
npx vercel deploy --prod
```

Then in Firebase Console → Authentication → Settings → Authorized domains, add your Vercel domain.

---

## Custom claims — how the multi-tenant boundary works

When `seed.mjs` ran, it set:
```js
await auth.setCustomUserClaims(user.uid, {
  role: 'RESIDENT_OWNER',
  societyId: 'sunshine-heights',
  flatId: 'B-402',
});
```

These claims travel in every Firebase ID token. The frontend reads them via `useAuth()`; Firestore Security Rules read them as `request.auth.token.societyId` to enforce that residents can only see data from their own society and visitors from their own flat.

When you build the Society Admin onboarding flow, the **set-claims** action must run server-side (Cloud Function) — never trust the client to pick its own role or society.

---

## Next steps (suggested order)

1. **Cloud Functions** — `createPaymentOrder`, Razorpay webhook, `inviteResident`, `generateMonthlyInvoices` (scheduled)
2. **Society Admin panel** — separate route group `/admin/(society)/*` reusing the same Firestore + auth
3. **Push notifications** — FCM SDK + server-side fan-out
4. **PWA manifest + service worker** so residents can "Add to home screen"

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `auth/captcha-check-failed` on login | Make sure `<div id="recaptcha-container" />` is mounted (it is, in `login/page.tsx`). Hard-reload the page. |
| `auth/billing-not-enabled` | Phone Auth in production requires a paid Blaze plan. For dev, use **test phone numbers** in Firebase Auth settings. |
| Bills/complaints don't show | (a) seed script ran successfully? (b) Firestore rules deployed? (c) custom claims set? (sign out + sign in to refresh the token) |
| `Missing or insufficient permissions` | Confirm `request.auth.token.societyId` matches the path. Run `npm run seed` to re-set claims. |
| Composite index error in console | Click the link in the error — Firestore generates the index for you. Or `firebase deploy --only firestore:indexes`. |

