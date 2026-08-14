# NIFT Movie Night — seat booking

A static, three-page booking system for a college film screening, backed by Firebase
Realtime Database + Auth. No build step: the files are plain HTML and can be served
from GitHub Pages or any static host.

| File | Who uses it | What it does |
|---|---|---|
| `index.html` | Students | Sign up / sign in, pick seats on a BookMyShow-style map, pay via UPI, view tickets |
| `admin.html` | Organisers | Verify payments, cancel bookings, block seats, manage promo codes, event settings, CSV export |
| `scanner.html` | Gate volunteers | Scan ticket QR codes and check guests in |
| `database.rules.json` | — | Realtime Database security rules (**must be deployed**, see below) |

---

## Deploying

### 1. Publish the security rules

**Do this before anything else.** The rules are what enforce access control — the
pages assume they are live.

Firebase Console → your project → Realtime Database → **Rules** → paste the contents
of `database.rules.json` → **Publish**.

Or with the Firebase CLI:

```bash
firebase deploy --only database
```

### 2. Grant organiser and volunteer access

There is no self-service admin signup — roles are assigned by hand in the console,
so nobody can grant themselves access.

In Realtime Database → Data, create a `roles` node keyed by Firebase Auth UID:

```json
{
  "roles": {
    "<organiser-uid>": "admin",
    "<volunteer-uid>": "staff"
  }
}
```

- `admin` — full access to `admin.html` and `scanner.html`
- `staff` — `scanner.html` only (can check guests in, cannot cancel bookings or change settings)

To find a UID: have the person sign in at `admin.html` or `scanner.html`. They'll
see a "not authorised" screen that displays their UID — copy it from there.

Nobody can write to `roles`, including admins. It is console-only by design.

### 3. Migrate existing bookings (one time)

Bookings made before this version stored personal details directly on the
publicly-readable seat map. After deploying the new rules, open `admin.html` →
**Seats** tab → **Migrate legacy bookings**. This moves names, emails, phone
numbers, student IDs and payment proof off `movieSeats` and into `bookings/`,
which only the booker and organisers can read.

The button shows how many seats still need migrating, and is safe to run more than
once. Until it's run, legacy bookings still work — they're flagged `LEGACY` in the
bookings table.

### 4. Configure the event

`admin.html` → **Event & pricing**: set the film title, language, format, venue,
date, show time, and the per-tier seat prices. These drive the student-facing page,
so no code change is needed to run the next show.

---

## Data model

```
roles/<uid>                        "admin" | "staff"          console-only

settings/
  bookingStatus                    "open" | "closed"          public read, admin write
  pricing/{premium,executive,classic}
  event/{title,language,format,venue,date,time}

movieSeats/<seatId>                public read — carries NO personal data
  status                           "held" | "occupied" | "blocked"
  userId                           owner's uid
  ticketId                         links to the booking record
  expiresAt                        hold expiry (held seats only)
  verificationStatus               "pending" | "approved" | "rejected"   (staff/admin only)
  checkedIn                        boolean                               (staff/admin only)

bookings/<uid>/<ticketId>          readable by the booker and organisers only
  bookerName, guestNames[], email, phone, whatsapp, studentId, department,
  utr, screenshotUrl, paidAmount, payeeUsed, promoCode, seats[], createdAt

promoCodes/<CODE>                  readable only if you know the code
  discount                         ₹ off per seat
  limit                            optional max redemptions
  used                             redemption counter (increment enforced by rules)
```

### Seat layout

8 rows (A–H) × 30 seats, split into three blocks of 10 with aisles between, and
grouped into priced tiers:

| Tier | Rows | Position |
|---|---|---|
| Premium | A, B | Farthest from the screen |
| Executive | C, D, E | Middle |
| Classic | F, G, H | Nearest the screen |

Seat IDs are unchanged from earlier versions (`A1` … `H30`), so existing bookings
keep working.

---

## How access control works

The pages are only a UI; every rule below is enforced server-side by
`database.rules.json`, so bypassing the page does not bypass the check.

- **Seat map is public, personal data is not.** `movieSeats` is world-readable so
  visitors can see availability before signing in, but the rules reject any write
  that puts an unexpected field on a seat node — personal data physically cannot
  live there.
- **A student can only touch their own seats.** Writes are limited to claiming a
  free seat, updating a hold they own, or taking over a hold that has expired.
  Once a seat is `occupied` the booker can no longer modify it — only an admin can.
- **Only staff can verify or check in.** `verificationStatus: "approved"` and
  `checkedIn: true` are rejected unless the writer has an `admin` or `staff` role.
  A student writing their own booking may only set `"pending"` and `false`.
- **Promo codes can't be over-redeemed.** The `used` counter may only increase by
  exactly one per write, and never past `limit`.
- **Blocking seats and changing prices are admin-only.**

Seat holds are taken with database transactions, so two people clicking the same
seat at the same moment can't both get it. Holds last 5 minutes; expired ones are
reclaimed automatically by the next booker, and organisers can sweep leftovers from
the **Seats** tab.

---

## Notes and limitations

- **Payment is manual.** Students pay by UPI and upload a screenshot plus the UTR;
  an organiser eyeballs it and approves. There is no payment gateway, so the amount
  a student reports is not machine-verified — always check the proof before
  approving. UPI IDs are hardcoded in `index.html` (`UPI_ACCOUNTS`).
- **Screenshot uploads use an unsigned Cloudinary preset**, which means anyone who
  reads the page source can upload to that Cloudinary account. Rotate the preset
  after the event, or move uploads behind a signed endpoint if this matters.
- **The Firebase `apiKey` in the source is not a secret** — it identifies the
  project, it doesn't grant access. The database rules are the security boundary.
- QR codes are generated in the browser, so ticket IDs are never sent to a
  third-party image service.
