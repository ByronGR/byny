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
