# 📷 Wedding QR Code Check-in App

A Progressive Web App (PWA) for real-time guest check-in at wedding venues. Ushers scan personalised QR codes from guests' e-invitations; the app verifies entry, tracks check-ins live via Firebase, and flags duplicates instantly.

Built for a Nigerian traditional wedding with 104 guests across 19 tables.

**Live demo:** [https://oiseomon.github.io/checkin-app](https://oiseomon.github.io/checkin-app)

---

## Features

- **QR code scanning** using the device camera (powered by [html5-qrcode](https://github.com/mebjas/html5-qrcode))
- **Real-time sync** via Firebase Realtime Database — all devices update instantly
- **Usher mode** (default) — scan and check in guests, see immediate result
- **Admin mode** (`?admin=1`) — full guest list, manual check-in/out, live stats, Excel export
- **Couple/Single detection** — each guest record stores how many people are allowed (1 or 2)
- **Duplicate prevention** — scanning an already-checked-in QR code shows an alert instead of granting entry
- **Offline-capable** — PWA with service worker caching
- **Excel export** — download full check-in report as `.xlsx` (SheetJS)
- **No app install required** — runs in any modern mobile browser

---

## How It Works

### QR Code Format

Each guest's personalised PNG invite contains an embedded QR code. The QR code encodes a unique token in the format:

```
G_XXXXXX_NNN
```

Where:
- `G_` — prefix identifying it as a guest token
- `XXXXXX` — 6-character random alphanumeric ID
- `NNN` — 3-digit guest sequence number

Example: `G_XU7XC8_076`

This token is the key used to look up the guest in Firebase.

### Check-in Flow

```
Usher opens app → Camera activates → Guest shows QR code on phone
  → App decodes token (e.g. G_XU7XC8_076)
  → Looks up token in Firebase
      ├─ Not found       → ❌ "Invalid QR code"
      ├─ Already checked in → ⚠️ "Already checked in at HH:MM"
      └─ Valid & new     → ✅ "Welcome, [Name]! Table [N] · Party of [1 or 2]"
                           → Firebase updated: checkedIn=true, checkinTime=timestamp
```

### Security Model

QR codes are **single-use by design** — not by encoding, but by state:

- The first scan sets `checkedIn: true` in Firebase
- All subsequent scans of the same QR code return "Already checked in"
- An usher at the door rejects anyone presenting a flagged QR code

> The QR code image can still be forwarded, but only the **first person to present it** gets checked in. The system relies on ushers acting on the screen result.

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Hosting | GitHub Pages |
| Database | [Firebase Realtime Database](https://firebase.google.com/products/realtime-database) |
| QR scanning | [html5-qrcode](https://github.com/mebjas/html5-qrcode) v2.3.8 |
| Excel export | [SheetJS](https://sheetjs.com) |
| Offline support | Service Worker (PWA) |
| UI | Vanilla HTML/CSS/JS |

---

## Project Structure

```
checkin-app/
├── index.html             # Main app (usher + admin mode)
├── manifest.json          # PWA manifest
├── sw.js                  # Service worker
├── guests.sample.json     # Example Firebase data format (safe to commit)
└── README.md
```

Your actual guest data lives in Firebase — nothing sensitive is stored in this repo.

---

## Setup

### 1. Firebase

1. Go to [Firebase Console](https://console.firebase.google.com) → Create a project
2. Enable **Realtime Database** → Start in test mode
3. In `index.html`, update the Firebase config block:

```js
const firebaseConfig = {
  apiKey:            "YOUR_API_KEY",
  authDomain:        "YOUR_PROJECT.firebaseapp.com",
  databaseURL:       "https://YOUR_PROJECT-default-rtdb.firebaseio.com",
  projectId:         "YOUR_PROJECT",
  storageBucket:     "YOUR_PROJECT.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId:             "YOUR_APP_ID"
};
```

### 2. Guest Data Format

Each guest is stored in Firebase under `/guests/{token}`:

```json
{
  "G_SAMPLE_001": {
    "name": "John Doe",
    "token": "G_SAMPLE_001",
    "allowed": 1,
    "table": "1",
    "checkedIn": false,
    "checkinTime": null
  },
  "G_SAMPLE_002": {
    "name": "Mr and Mrs Sample",
    "token": "G_SAMPLE_002",
    "allowed": 2,
    "table": "2",
    "checkedIn": false,
    "checkinTime": null
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Guest display name |
| `token` | string | Unique QR code value (matches what's embedded in the PNG) |
| `allowed` | number | `1` = single, `2` = couple |
| `table` | string | Table number |
| `checkedIn` | boolean | `false` until scanned at the door |
| `checkinTime` | string / null | ISO timestamp set on first scan |

**To load your guest data into Firebase:**
1. Open Firebase Console → Realtime Database
2. Click the three-dot menu → **Import JSON**
3. Upload your `firebase_guests.json`

### 3. Deploy

The app is a single HTML file — deploy anywhere static files are served.

**GitHub Pages (free):**
```bash
git clone https://github.com/Oiseomon/checkin-app.git
cd checkin-app
# Push your changes
git push origin main
# Enable GitHub Pages: Settings → Pages → Deploy from main branch
```

The app will be live at `https://YOUR_USERNAME.github.io/checkin-app`.

---

## Usage

### Usher Mode (default)

Open `https://YOUR_USERNAME.github.io/checkin-app` on any mobile device.

1. Allow camera access when prompted
2. Point the camera at the guest's QR code
3. The result appears immediately:
   - ✅ **Green** — valid guest, name and table displayed, party size shown
   - ⚠️ **Yellow** — already checked in (show time of first scan)
   - ❌ **Red** — QR code not recognised

### Admin Mode

Open `https://YOUR_USERNAME.github.io/checkin-app?admin=1`

- See all guests and their check-in status in real time
- Manually check in or undo a check-in
- View live stats (total checked in, remaining, couples vs singles)
- Export the full check-in list to Excel (`.xlsx`)

---

## Generating QR-Coded Invite PNGs

This app is the **receiving end** of a two-part system. To generate the personalised PNG invitations with embedded QR codes, see the companion project:

**[Wedding E-Invite Sender](https://github.com/Oiseomon/wedding-einvite-sender)** — generates invite PNGs, sends them via WhatsApp (automated and manual helpers).

The QR token embedded in each PNG must match the `token` field in Firebase.

---

## Security & Privacy

**No guest data is stored in this repository.** All personal data (names, check-in times, table assignments) lives in your Firebase project, which you control.

Firebase security rules (recommended for production):

```json
{
  "rules": {
    "guests": {
      ".read": true,
      ".write": true
    }
  }
}
```

> For a more secure setup, restrict write access to authenticated admin users only and give ushers read + limited write (checkedIn field only).

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Camera doesn't activate | Allow camera permission in browser settings; use HTTPS (required for camera API) |
| QR code not recognised | Ensure the token in the QR matches the `token` field in Firebase exactly |
| Firebase data not loading | Check the `databaseURL` in the config; verify Realtime Database rules allow reads |
| Already checked in shows for a new guest | The token was scanned before — use Admin mode to reset `checkedIn` to `false` |
| App doesn't work offline | Open it once with internet connection first to cache the service worker |
| Excel export is empty | Switch to Admin mode (`?admin=1`) to use the export function |

---

## Related Project

The invite PNGs scanned by this app are generated and distributed by the **[Wedding E-Invite Sender](https://github.com/Oiseomon/wedding-einvite-sender)** — a Node.js automation tool with mobile and PC manual helpers for WhatsApp distribution.

---

## License

MIT — free to use and adapt for your own events.
