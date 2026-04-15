# Firebase Project Setup

## Prerequisites

- Node.js 18+ (for CLI and Cloud Functions)
- A Google account
- Firebase project created at [console.firebase.google.com](https://console.firebase.google.com)

## Install the Firebase CLI

```bash
npm install -g firebase-tools
firebase login
```

## Initialize a Project

```bash
firebase init
```

Select the services you want (Firestore, Hosting, Functions, etc.). This creates:
- `firebase.json` — CLI configuration (hosting, functions, emulators)
- `.firebaserc` — project aliases
- `firestore.rules` / `storage.rules` — security rules
- `firestore.indexes.json` — Firestore composite indexes

## Web SDK Setup (Client-Side)

### Install

```bash
npm install firebase
```

### Initialize (modular v9+ API)

Create a `src/firebase.ts` (or `firebase.js`) config module:

```typescript
import { initializeApp } from 'firebase/app';
import { getFirestore } from 'firebase/firestore';
import { getAuth } from 'firebase/auth';
import { getStorage } from 'firebase/storage';

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,        // or import.meta.env.VITE_*
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID,
};

const app = initializeApp(firebaseConfig);

export const db = getFirestore(app);
export const auth = getAuth(app);
export const storage = getStorage(app);
```

Find your config in the Firebase console: **Project Settings → Your apps → SDK setup and configuration**.

### Environment Variables

Never commit the Firebase config to source control if it contains sensitive keys. For web apps, the `apiKey` is safe to expose (it identifies the project, not a secret), but restricting it to specific domains/IPs in the Google Cloud Console is good practice.

Use a `.env.local` file (git-ignored) for local development:

```
NEXT_PUBLIC_FIREBASE_API_KEY=AIza...
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
NEXT_PUBLIC_FIREBASE_PROJECT_ID=your-project
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=123456789
NEXT_PUBLIC_FIREBASE_APP_ID=1:123:web:abc
```

## Admin SDK Setup (Server-Side)

Use the Admin SDK for server environments: Cloud Functions, Next.js API routes, Express backends, scripts.

### Install

```bash
npm install firebase-admin
```

### Initialize

```typescript
import { initializeApp, getApps, cert } from 'firebase-admin/app';
import { getFirestore } from 'firebase-admin/firestore';
import { getAuth } from 'firebase-admin/auth';

// Avoid re-initializing in hot-reload environments (Next.js, etc.)
if (!getApps().length) {
  initializeApp({
    credential: cert({
      projectId: process.env.FIREBASE_PROJECT_ID,
      clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
      privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, '\n'),
    }),
  });
}

export const adminDb = getFirestore();
export const adminAuth = getAuth();
```

### Service Account Key

Download from: **Firebase Console → Project Settings → Service Accounts → Generate new private key**.

Store the values in environment variables (never commit the JSON file):

```
FIREBASE_PROJECT_ID=your-project
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-xxx@your-project.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nMII...\n-----END PRIVATE KEY-----\n"
```

In Cloud Functions, the Admin SDK auto-initializes with `initializeApp()` (no credentials needed) — the runtime provides them automatically.

## Local Emulator Suite

Run Firebase services locally without touching production:

```bash
firebase emulators:start
```

Connect the Web SDK to emulators in development:

```typescript
import { connectFirestoreEmulator } from 'firebase/firestore';
import { connectAuthEmulator } from 'firebase/auth';
import { connectStorageEmulator } from 'firebase/storage';

if (process.env.NODE_ENV === 'development') {
  connectFirestoreEmulator(db, 'localhost', 8080);
  connectAuthEmulator(auth, 'http://localhost:9099');
  connectStorageEmulator(storage, 'localhost', 9199);
}
```

The emulator UI is available at `http://localhost:4000`.
