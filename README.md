# Byny Rental

Static Firebase-backed operations app for Byny Rental.

## Deploy to Vercel

This project does not need a build step or API routes. Deploy the folder as a static project:

```bash
vercel
vercel --prod
```

## Firebase

The app stores business data in Firestore at:

```text
byny/data
```

Use `firestore.rules` and `storage.rules` as the starting security rules in Firebase. They require Firebase Auth for reading and writing business data.

Important: the current driver portal is client-side and cédula-based. If drivers need to access the app from their own phones without an admin Firebase session, the next step should be proper driver authentication or separate driver-safe Firestore documents.

## Sync safety rules — do not remove

In August 2026 a full set of real business data was destroyed by this app. The cause:
the file shipped a block of demo data that ran whenever `DB.bikes` was empty. On a device
that had never opened the app, Firebase had not answered yet, so the demo data was created,
stamped with the *current* time, and — because sync resolved conflicts by "newest timestamp
wins" — uploaded over the real data on the server.

Three rules now prevent this. Do not undo them:

1. **No seed data, ever.** The app must start empty. Never reintroduce a "create sample
   data if empty" block — an empty database and a not-yet-loaded database look identical.
2. **`hasSyncedOnce` gates all uploads.** A device may not write to Firestore until
   Firestore has answered it at least once. Local saves before that are kept on the device
   only, and `_meta.fromSyncedSession` records whether a save was made on top of server
   data. Only saves with that flag may overwrite the server.
3. **Shrinking snapshots are never silent.** If the server sends fewer records than the
   device holds, a full copy is stashed under a `byny_rescate_*` key in localStorage and the
   user is warned before anything is replaced.

## Pico y placa

Medellín rotates the restricted digits every semester. The table lives in
**Ajustes → Pico y placa** and is stored in `settings.pypDays` (day → digits, the way the
Alcaldía publishes it). `PYP_DAYS_DEFAULT` in `index.html` is only the fallback and holds
the rotation in force from 3 August 2026.

Update it in the app when the city rotates — not in the code. Restriction days are
non-billable, so a stale table silently overcharges and undercharges drivers.

## Known issues (still open)

- **Drivers cannot use the portal from their own phones.** Firestore rules require auth,
  and only an admin login triggers a data load, so `DB.drivers` is empty on a driver's
  device and the cédula is never found. Their submissions also cannot reach Firestore.
  Needs real driver authentication.
- **Drivers are matched to bikes by first-name substring** (`b.renter.includes(fname)`).
  Two drivers sharing a first name, or a name contained in another, resolve to the wrong bike.
- **The Bancolombia account number is hardcoded in `index.html`** and this repository is
  public. Consider making the repo private.
- Any authenticated user has full read/write on all business data, including personal
  cédulas and the owner's retiros.
