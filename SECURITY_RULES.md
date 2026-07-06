# Firestore Security Rules — Audio Clarity (Final, Production-Ready)

```
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {

    match /users/{userId} {
      allow read: if request.auth != null && request.auth.uid == userId;

      // First-time profile creation must start as premium: false.
      allow create: if request.auth != null
                    && request.auth.uid == userId
                    && request.resource.data.premium == false;

      // Client may update displayName/photoURL/lastLogin, but the
      // premium field must stay exactly what it already was — it
      // can only change via the Admin SDK (Cloud Function), which
      // bypasses these rules entirely.
      allow update: if request.auth != null
                    && request.auth.uid == userId
                    && request.resource.data.premium == resource.data.premium;

      // Nobody deletes their own profile from the client. If you
      // ever need account deletion, do it via a Cloud Function so
      // you can also clean up usage/{uid} and any billing records
      // in the same transaction.
      allow delete: if false;
    }

    match /usage/{userId} {
      allow read: if request.auth != null && request.auth.uid == userId;

      // First write must start the counter at 0.
      allow create: if request.auth != null
                    && request.auth.uid == userId
                    && request.resource.data.freeUsed == 0;

      // freeUsed may only stay the same or increase — a user can
      // update deviceFingerprint/lastUsed freely, but can never roll
      // freeUsed back down to reset their own trial.
      allow update: if request.auth != null
                    && request.auth.uid == userId
                    && request.resource.data.freeUsed >= resource.data.freeUsed;

      allow delete: if false;
    }
  }
}
```

## Why each piece is there

- **`premium` locked on both create and update** — a user can create their
  own profile (must start `false`) and later update other fields, but the
  `premium` value itself can never move on a client-issued write. Only a
  Cloud Function using the Admin SDK (which isn't subject to these rules
  at all) can flip it to `true`, after your Razorpay payment is verified
  server-side.
- **`freeUsed` is monotonic** — `request.resource.data.freeUsed >=
  resource.data.freeUsed` means the value can go up (or stay the same when
  only `deviceFingerprint`/`lastUsed` are touched) but never down. A user
  editing their own document in devtools can't reset their trial count.
- **`delete: if false` on both collections** — closes the last obvious
  loophole: deleting your `usage/{uid}` doc and letting the app recreate it
  fresh would otherwise reset `freeUsed` back to 0. With delete blocked,
  that path doesn't exist.

## Deployment

Paste this into **Firebase console → Firestore Database → Rules**, or if
you use the CLI:

```
firebase deploy --only firestore:rules
```

No app code changes are needed for this — these rules run entirely on
Firebase's servers and apply to the existing `users/{uid}` and
`usage/{uid}` structure already used in `audio_clarity_tool.html`.

## Final architecture this locks in

```
Firestore
├─ users/{uid}
│    premium, displayName, email, photoURL, createdAt, lastLogin
└─ usage/{uid}
     freeUsed, lastUsed, deviceFingerprint
```

- User can read/create/update their own profile — never delete it, never
  touch `premium` themselves.
- `premium` flips to `true` only via a Cloud Function, after Razorpay
  payment verification.
- `freeUsed` only ever increases.
- Nothing is deletable from the client.

This is a solid, production-appropriate rule set for this data model.
