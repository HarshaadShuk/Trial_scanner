# Coupon Verifier — Setup Guide

A QR scanner that installs like an app on Android, checks a scanned code against your Google Sheet, and marks the coupon as used — live, with no server to host.

It has two parts:
1. **The backend** — a small script that lives *inside* your Google Sheet (Google Apps Script). It's the only thing allowed to read/write your sheet.
2. **The app** — `index.html` + friends, hosted anywhere, opened on your phone, and installed to the home screen.

Total setup time: about 10 minutes, once.

---

## Part 1 — Prepare your Google Sheet

Your sheet needs a header row (row 1) with at least these two exact column names:

- `QR Token` — the unique code that's encoded in each coupon's QR image.
- `Coupon Used?` — leave this blank for unused coupons. The app fills it in with `USED`.

Optional: add a `Used At` column — the app will auto-fill it with a timestamp when a coupon is verified.

If your QR codes don't exist yet, put a random string in every row's `QR Token` cell (e.g. `CPN-0001`, `CPN-0002`, ...) — that string is exactly what gets encoded into each coupon's printed/emailed QR image later.

---

## Part 2 — Deploy the backend (Apps Script)

1. Open your Google Sheet.
2. Go to **Extensions → Apps Script**. A new tab opens with a code editor.
3. Delete anything in the default `Code.gs` file, and paste in the entire contents of **`apps-script/Code.gs`** from this project.
4. Check the three settings near the top of the file match your sheet:
   ```js
   var SHEET_NAME = '';              // leave blank for the first sheet, or name it e.g. 'Sheet1'
   var TOKEN_COLUMN_NAME = 'QR Token';
   var USED_COLUMN_NAME = 'Coupon Used?';
   ```
5. Click **Deploy → New deployment**.
6. Click the gear icon next to "Select type" → choose **Web app**.
7. Fill in:
   - **Execute as:** `Me`
   - **Who has access:** `Anyone`  
     (This does *not* make your sheet public — it just means the scanner app can call this one script. The script only ever exposes verify/used-or-not, never your raw data.)
8. Click **Deploy**. The first time, Google will ask you to authorize the script — click through **Advanced → Go to [project name] (unsafe)** (this warning is normal for your own scripts) and allow access.
9. Copy the **Web app URL** it gives you — it looks like:
   ```
   https://script.google.com/macros/s/AKfycb.../exec
   ```
   Keep this tab open, you'll paste this URL into the app next.

> **If you ever edit `Code.gs` again**, you must do **Deploy → Manage deployments → ✏️ Edit → New version → Deploy** for the changes to go live. Saving the file alone is not enough.

---

## Part 3 — Host the app files

The app is just static files (`index.html`, `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png`). It needs to be served over **https** (required for camera access) from *some* URL. Easiest free options:

- **GitHub Pages** — create a repo, upload these 5 files to the root, enable Pages in repo Settings. You'll get a URL like `https://yourname.github.io/coupon-verifier/`.
- **Netlify Drop** — go to [app.netlify.com/drop](https://app.netlify.com/drop) and drag the folder in. Instant URL, no account needed for a quick test (create a free account to keep it permanent).
- **Vercel / Firebase Hosting** — also work fine if you already use them.

Do **not** try to open `index.html` directly from your phone's file manager (`file://...`) — the camera API and service worker both require `https://`.

---

The Apps Script URL is already baked into `index.html` — there's no setup screen, it opens straight to the camera.

> If you ever redeploy the backend and get a *new* URL, open `index.html`, find the line starting with `const SCRIPT_URL` near the top of the `<script>` block, and update it there.

## Part 4 — Install on Android

1. On your Android phone, open the hosted URL in **Chrome**.
2. Allow camera access when Chrome asks.
3. Tap Chrome's **⋮ menu → Add to Home screen → Install**. This adds a real app icon and opens full-screen, no browser bar, like a native app.

That's it — you now have an installable Android app that scans a coupon, checks it live against your sheet, and marks it used, with no APK, no Play Store, and no server bill.

---

## How it behaves

| Scan result | What happens |
|---|---|
| Code matches a row, `Coupon Used?` is empty/false | Shows **green "VERIFIED"**, marks that row `USED` (and timestamps it, if you added a `Used At` column) |
| Code matches a row, `Coupon Used?` already filled in | Shows **red "ALREADY USED"** — does *not* touch the sheet again |
| Code doesn't match any row | Shows **orange "NOT FOUND"** |
| No internet / script unreachable | Shows **"CONNECTION ERROR"** — nothing is written until it can reach the sheet |

A running session tally (Verified / Used / Unknown) shows at the top — it resets when the app is reloaded, it's just for the person scanning to keep count during an event.

Two people scanning the same coupon at the exact same instant can't both mark it used — the backend uses a lock so only one wins the race; the other correctly sees "already used."

---

## Troubleshooting

- **"Could not find columns..." error** — your header row's spelling must match exactly: `QR Token` and `Coupon Used?` (case-sensitive, including the `?`). Fix the sheet or update the constants in `Code.gs` and redeploy (see the "New version" note above).
- **Camera doesn't open** — must be https, and you must tap "Allow" on the permission prompt. If you accidentally denied it, reset site permissions in Chrome (site info icon → Permissions).
- **Everything says "CONNECTION ERROR"** — double check the Web app URL was pasted in full (ends in `/exec`), and that the deployment's "Who has access" is set to Anyone.
- **Want to reset which coupons are used** — just clear the `Coupon Used?` column in the sheet manually; the app always reads live.
