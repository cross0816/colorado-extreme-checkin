# Learn to Skate — Attendance Check-In

A standalone kiosk app for tracking Learn to Skate attendance. Families scan a
QR code (or type a short code from a printed card) when they arrive, and the
check-in is logged live to Firestore. Staff manage the roster and view
attendance from a separate admin console.

It uses the **same Firebase project** as the CXA Parent Portal
(`cxa-parent-portal`), in two new Firestore collections, so no new Firebase
project setup is required. Staff admin accounts are the same ones already
used for the parent portal (`users/{uid}.role == 'admin'`).

## Pages

- `index.html` — the check-in kiosk. **Staff-only**: requires the same admin
  login as the parent portal before it shows the scanner. Once signed in on a
  device, the session persists (Firebase Auth keeps you logged in), so staff
  sign in once and can leave the device out for families to scan themselves —
  families never see a login screen here, they just scan.
- `signup.html` — public self-registration form, no login required. A family
  fills in their info and skaters (including date of birth) and immediately
  gets their unique code + QR code — this is how a new family gets a code to
  use at the kiosk, without ever needing to log in or touch the kiosk's admin
  login.
- `admin.html` — staff-only console: manage the family/skater roster, generate
  & print QR codes, and view a live attendance log with CSV export. Requires
  the same admin login as the parent portal.

## Data model

**`lts_families/{code}`** — one document per family, keyed by a random
6-character code (also encoded into the printed QR). Created either by staff
via `admin.html` or by the family itself via `signup.html`.
```
{
  familyName: "The Wolitski Family",
  parentName: "Carlos P.",
  contact: "carlos@coloradoextreme.org",
  students: [ { name: "Vaughn Wolitski", dob: "2014-07-22", level: "Basic 3" } ],
  active: true,
  createdAt, updatedAt
}
```

**`lts_attendance/{familyId}_{date}`** — one document per family per calendar
day (student names are copied at check-in time so history stays intact even
if the roster later changes). The document ID is deterministic
(`familyId_date`) rather than a random ID — that's what lets the kiosk check
"did this family already check in today" with a single-document read
(cheap and safe to expose publicly) instead of a broader query (which would
require being logged in).
```
{
  familyId: "7F3K9Q",
  familyName: "The Wolitski Family",
  students: [ { name: "Vaughn Wolitski", level: "Basic 3" } ],
  date: "2026-07-01",       // America/Denver, YYYY-MM-DD
  checkedInAt: <server timestamp>
}
```

A family can only be checked in once per calendar day — scanning again (or a
staff manual check-in for the same day) just shows "Already checked in"
instead of creating a duplicate record.

## Required setup: Firestore rules

The kiosk (`index.html`) is unauthenticated by design (self-check-in, no
login), so it needs limited public access to look up codes and log
check-ins. Add the following to your existing Firestore rules in the
[Firebase console](https://console.firebase.google.com/project/cxa-parent-portal/firestore/rules)
— **alongside**, not replacing, your current rules for the parent portal:

```
function isAdmin() {
  return request.auth != null &&
    exists(/databases/$(database)/documents/users/$(request.auth.uid)) &&
    get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'admin';
}

match /lts_families/{familyId} {
  // Public read so the kiosk can look up a scanned code without logging in.
  allow read: if true;
  // Admins can always create/edit. Families can also self-register via
  // signup.html — but only as a brand-new, tightly-shaped, active record.
  // Editing or deleting an existing family still requires staff.
  allow create: if isAdmin() || (
    request.resource.data.keys().hasOnly(['familyName','parentName','contact','students','active','createdAt']) &&
    request.resource.data.familyName is string && request.resource.data.familyName.size() > 0 &&
    request.resource.data.parentName is string &&
    request.resource.data.contact is string &&
    request.resource.data.students is list &&
    request.resource.data.students.size() > 0 && request.resource.data.students.size() <= 10 &&
    request.resource.data.active == true
  );
  allow update, delete: if isAdmin();
}

match /lts_attendance/{recordId} {
  // recordId is always `${familyId}_${date}` — a single-document read only tells
  // you whether that exact family already checked in on that exact day, which is
  // what the kiosk needs. "list" (browsing/querying all records) stays admin-only.
  allow get: if true;
  allow list: if isAdmin();
  allow create: if
    recordId == request.resource.data.familyId + '_' + request.resource.data.date &&
    request.resource.data.familyId is string &&
    exists(/databases/$(database)/documents/lts_families/$(request.resource.data.familyId)) &&
    request.resource.data.keys().hasOnly(['familyId','familyName','students','date','checkedInAt']);
  allow update, delete: if isAdmin();
}
```

**Note on scope:** the kiosk itself is now behind admin login, but
`signup.html` is intentionally still public (that's the whole point — a new
family registers themselves without staff or a login). That means the public
`read` on `lts_families`, the public (validated) `create` on `lts_families`
for self-signup, and the public `get`/`create` on `lts_attendance` all still
apply — anyone with the URL could technically read family/skater names,
register junk families, or spam check-in writes (the rules above at least
require well-shaped data and a real, existing family code). If that's a
concern for your deployment, consider adding Firebase App Check, or moving
the writes behind a Cloud Function later — that's a bigger change outside
this app's current scope.

## Using it

**Option A — family self-registers:**
1. Point a new family at `signup.html` (link it from your website, or hand out
   the URL). They fill in their info and skaters (name + date of birth), and
   get a unique code + QR code immediately, with print/download/screenshot
   options.
2. From then on they use that code at the kiosk (`index.html`).

**Option B — staff registers them:**
1. Open `admin.html`, sign in with an admin account, go to
   **Roster → Add Family**, enter the family/skater info, and save. A unique
   code is generated automatically.
2. Click **QR Code** on that family's row to view/print/download a card with
   their QR code and the short code as text (for manual entry).
3. Hand out the printed card, or let the family save/screenshot the QR.

**Every visit after that:**
4. **Staff:** sign in once on the front-desk kiosk device (`index.html`) with
   an admin account — the session stays signed in on that device afterward.
5. **Families:** scan their code at the kiosk, or type the short code if
   scanning fails. No login needed on their end.
6. **Staff:** the **Attendance** tab (in `admin.html`) shows who has checked
   in for any given day in real time, with a CSV export for record-keeping.
   Editing or deactivating a family (or fixing a signup typo) is admin-only,
   from the **Roster** tab.

## Deployment

This is its own standalone GitHub Pages site (separate from the CXA Parent
Portal repo), served from the repo root at `lts.coloradoextreme.org`:

- Kiosk: `https://lts.coloradoextreme.org/`
- Sign up: `https://lts.coloradoextreme.org/signup.html`
- Admin: `https://lts.coloradoextreme.org/admin.html`

The `CNAME` file in this repo sets the custom domain, and DNS has a `CNAME`
record for `lts` pointing at `cross0816.github.io`. Camera access requires
HTTPS, which GitHub Pages provides automatically once the custom domain is
verified (check "Enforce HTTPS" under repo Settings → Pages).
