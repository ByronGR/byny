# Byny Rental

Static Firebase-backed operations app for Byny Rental.

## Deploy to Vercel

This project does not need a build step or API routes. Deploy the folder as a static project:

```bash
vercel
vercel --prod
```

## Firebase

### One document per record (v4.0)

Business data lives in per-record collections:

```text
bikes/{id}   drivers/{id}   payments/{id}   expenses/{id}   retiros/{id}
graceDays/{id}   contracts/{id}   appointments/{id}   renterHistory/{id}
config/settings
```

Until v4.0 the entire business was a single document, `byny/data`. Every save
rewrote the whole thing, so two people saving at the same time overwrote each
other — the same failure mode that destroyed the data in August, just from a
different direction. Writes are now per record: `save()` diffs the in-memory `DB`
against the last known server state and writes only what changed, in a batch.

**`byny/data` is deliberately left in place** as the migration's backup, and the
rules still allow it. Do not delete it.

**Publish `firestore.rules` before deploying** a build that expects collections.
If the rules are missing the app detects `permission-denied`, falls back to the
legacy document, and shows a banner — it degrades rather than breaking — but that
fallback still has the concurrent-write problem.

The migration runs once, automatically, and only after *every* collection has
returned a snapshot. An empty collection and one that has not loaded yet look
identical, and confusing the two is exactly what caused the original data loss.

The app is admin-only. There is one login (Firebase Auth, LOCAL persistence so the
session lasts) and four tabs: **Inicio**, **Motos**, **Dinero**, **Ajustes**. Motos and
Dinero each carry sub-tabs; everything else was collapsed into them.

The driver-facing portal was removed in v3.0. It never worked from a driver's own phone —
Firestore rules require auth, only an admin login triggered a data load, so `DB.drivers`
was always empty there and the cédula was never found. Driver *management* (the admin
screens) is unchanged, under Motos → Conductores.

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

## Importing

`Importar JSON` **adds** to what is already loaded; it does not replace it. Replacing is
still possible but takes two deliberate confirmations, and either way a full copy of the
prior state is written to a `byny_rescate_import_*` key in localStorage first.

The old behaviour was a silent full replace, which cost two hand-entered bikes on
2026-08-16: importing a one-bike file deleted everything else. Merging is by `id`; an
incoming record whose id already exists is given a new one rather than dropped.

## Files and receipts

Receipt photos and driver documents go to **Firebase Storage**; only the URL is kept in
the Firestore document. This is not cosmetic: a Firestore document is capped at **1 MB**,
and a single phone photo stored as base64 exhausts it, at which point *every* save fails.
Before v3.0 the payment upload looked up an element id that did not exist
(`pay-receipt` instead of `pay-receipt-inp`) so it silently never ran, and expenses had no
Storage path at all — both wrote base64 straight into the document. If you touch this code,
keep images out of the document.

## Known issues (still open)

- **Drivers are matched to bikes by first-name substring** (`b.renter.includes(fname)`).
  Two drivers sharing a first name, or a name contained in another, resolve to the wrong bike.
- The repository is public. Nothing sensitive is in it any more (the Bancolombia account
  number and payment QR left with the driver portal in v3.0), but private is still the
  safer default.
- Any authenticated user has full read/write on all business data, including personal
  cédulas and the owner's retiros.
