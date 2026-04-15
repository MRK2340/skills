# Cloud Functions for Firebase

Cloud Functions lets you run server-side code in response to Firebase events (Firestore writes, Auth events, Storage uploads, HTTP requests, Pub/Sub messages, schedules) without managing infrastructure.

## Setup

```bash
# Initialize in your project (choose TypeScript when prompted)
firebase init functions

# Install dependencies
cd functions && npm install
```

The `functions/` directory structure:

```
functions/
├── src/
│   └── index.ts       ← export your functions here
├── package.json
└── tsconfig.json
```

## HTTP Functions (REST API)

```typescript
import * as functions from 'firebase-functions';
import * as admin from 'firebase-admin';
import express from 'express';
import cors from 'cors';

admin.initializeApp();

const app = express();
app.use(cors({ origin: true }));

app.get('/hello', (req, res) => {
  res.json({ message: 'Hello from Firebase!' });
});

app.post('/users/:id', async (req, res) => {
  try {
    const { id } = req.params;
    await admin.firestore().collection('users').doc(id).update(req.body);
    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: (err as Error).message });
  }
});

export const api = functions.https.onRequest(app);
```

### Callable Functions (recommended for auth'd clients)

Callable functions automatically handle auth, HTTPS, and CORS. The client SDK calls them directly.

```typescript
// functions/src/index.ts
export const processOrder = functions.https.onCall(async (data, context) => {
  // Verify authentication
  if (!context.auth) {
    throw new functions.https.HttpsError('unauthenticated', 'Must be logged in');
  }

  const { orderId, items } = data;
  const uid = context.auth.uid;

  // Business logic...
  await admin.firestore().collection('orders').doc(orderId).set({
    uid,
    items,
    status: 'processing',
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
  });

  return { orderId, status: 'processing' };
});
```

```typescript
// Client-side call
import { getFunctions, httpsCallable } from 'firebase/functions';

const functions = getFunctions();
const processOrder = httpsCallable(functions, 'processOrder');

const result = await processOrder({ orderId: '123', items: [...] });
console.log(result.data); // { orderId: '123', status: 'processing' }
```

## Firestore Triggers

```typescript
import * as functions from 'firebase-functions';
import * as admin from 'firebase-admin';

// Trigger on document create
export const onUserCreated = functions.firestore
  .document('users/{userId}')
  .onCreate(async (snap, context) => {
    const { userId } = context.params;
    const userData = snap.data();

    // Send welcome email, set up defaults, etc.
    await admin.firestore().collection('userProfiles').doc(userId).set({
      createdAt: admin.firestore.FieldValue.serverTimestamp(),
      plan: 'free',
    });
  });

// Trigger on update
export const onOrderUpdated = functions.firestore
  .document('orders/{orderId}')
  .onUpdate(async (change, context) => {
    const before = change.before.data();
    const after = change.after.data();

    if (before.status !== after.status && after.status === 'shipped') {
      // Send shipping notification
      console.log(`Order ${context.params.orderId} shipped`);
    }
  });

// Trigger on delete
export const onPostDeleted = functions.firestore
  .document('posts/{postId}')
  .onDelete(async (snap, context) => {
    const postData = snap.data();
    // Clean up related data
    const batch = admin.firestore().batch();
    const comments = await admin.firestore()
      .collection('comments')
      .where('postId', '==', context.params.postId)
      .get();
    comments.forEach((doc) => batch.delete(doc.ref));
    await batch.commit();
  });

// Trigger on write (create, update, or delete)
export const onDocumentWritten = functions.firestore
  .document('posts/{postId}')
  .onWrite(async (change, context) => {
    if (!change.after.exists) {
      // Document deleted
    } else if (!change.before.exists) {
      // Document created
    } else {
      // Document updated
    }
  });
```

## Authentication Triggers

```typescript
// Trigger when a new user signs up
export const onUserSignUp = functions.auth.user().onCreate(async (user) => {
  await admin.firestore().collection('users').doc(user.uid).set({
    email: user.email,
    displayName: user.displayName ?? '',
    createdAt: admin.firestore.FieldValue.serverTimestamp(),
    plan: 'free',
  });
});

// Trigger when a user is deleted
export const onUserDeleted = functions.auth.user().onDelete(async (user) => {
  // Clean up user data
  await admin.firestore().collection('users').doc(user.uid).delete();
});
```

## Storage Triggers

```typescript
export const onFileUploaded = functions.storage.object().onFinalize(async (object) => {
  const filePath = object.name!;          // e.g., "uploads/userId/image.jpg"
  const contentType = object.contentType!;
  const bucket = object.bucket;

  // Skip non-image files or already-processed files
  if (!contentType.startsWith('image/') || filePath.includes('thumb_')) return;

  // Process the image (see storage.md for full thumbnail example)
  console.log(`Processing upload: ${filePath}`);
});

export const onFileDeleted = functions.storage.object().onDelete(async (object) => {
  console.log(`File deleted: ${object.name}`);
});
```

## Scheduled Functions (Cron Jobs)

```typescript
// Run every day at midnight UTC
export const dailyCleanup = functions.pubsub
  .schedule('0 0 * * *')
  .timeZone('America/New_York')
  .onRun(async (context) => {
    const cutoff = new Date();
    cutoff.setDate(cutoff.getDate() - 30);

    const oldDocs = await admin.firestore()
      .collection('tempData')
      .where('createdAt', '<', cutoff)
      .get();

    const batch = admin.firestore().batch();
    oldDocs.forEach((doc) => batch.delete(doc.ref));
    await batch.commit();

    console.log(`Cleaned up ${oldDocs.size} old documents`);
  });
```

Cron syntax: `'every 5 minutes'`, `'every 1 hours'`, or standard cron `'0 9 * * 1'` (every Monday at 9am).

## Pub/Sub Triggers

```typescript
export const onMessage = functions.pubsub
  .topic('my-topic')
  .onPublish(async (message) => {
    const data = message.json; // parsed JSON payload
    console.log('Received:', data);
  });
```

## Configuration & Environment Variables

```bash
# Set config (legacy, still widely used)
firebase functions:config:set stripe.key="sk_live_..." sendgrid.key="SG.xxx"

# Access in functions
const stripeKey = functions.config().stripe.key;
```

For 2nd gen functions and newer patterns, use environment variables:

```bash
# Set secrets (stored in Google Secret Manager)
firebase functions:secrets:set STRIPE_SECRET_KEY
```

```typescript
// Access secrets
export const processPayment = functions
  .runWith({ secrets: ['STRIPE_SECRET_KEY'] })
  .https.onCall(async (data, context) => {
    const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2023-10-16' });
    // ...
  });
```

## Function Configuration

```typescript
// Set memory, timeout, and region
export const heavyTask = functions
  .runWith({
    memory: '2GB',
    timeoutSeconds: 540,       // max 540s (9 minutes)
    minInstances: 1,           // keep warm to avoid cold starts
    maxInstances: 10,
  })
  .region('us-east1')
  .https.onRequest(handler);
```

## 2nd Generation Functions (recommended for new projects)

```typescript
import { onRequest, onCall } from 'firebase-functions/v2/https';
import { onDocumentCreated } from 'firebase-functions/v2/firestore';
import { onSchedule } from 'firebase-functions/v2/scheduler';
import { defineSecret } from 'firebase-functions/params';

const stripeKey = defineSecret('STRIPE_SECRET_KEY');

// HTTP function (v2)
export const api = onRequest(
  { region: 'us-central1', memory: '256MiB', timeoutSeconds: 60 },
  (req, res) => {
    res.json({ hello: 'world' });
  }
);

// Callable (v2)
export const processOrder = onCall(
  { secrets: [stripeKey] },
  async (request) => {
    if (!request.auth) throw new HttpsError('unauthenticated', 'Login required');
    // process.env.STRIPE_SECRET_KEY is available here
  }
);

// Firestore trigger (v2)
export const onUserCreated = onDocumentCreated('users/{userId}', async (event) => {
  const userId = event.params.userId;
  const data = event.data?.data();
});

// Scheduled (v2)
export const nightly = onSchedule('0 2 * * *', async (event) => {
  // runs every day at 2am UTC
});
```

## Deploy

```bash
# Deploy all functions
firebase deploy --only functions

# Deploy a specific function
firebase deploy --only functions:api

# Delete a function
firebase functions:delete functionName
```

## Local Testing with Emulators

```bash
firebase emulators:start --only functions,firestore,auth
```

Test HTTP functions at `http://localhost:5001/your-project/us-central1/api`.

Test callable functions from the client — the emulator intercepts calls automatically when you've called `connectFunctionsEmulator`.

## Error Handling

For callable functions, throw `HttpsError` with a standard gRPC code:

```typescript
import { HttpsError } from 'firebase-functions/v2/https';

// Common codes: unauthenticated, permission-denied, not-found,
//               already-exists, resource-exhausted, invalid-argument,
//               internal, unavailable
throw new HttpsError('permission-denied', 'You do not have access to this resource');
```

For background triggers (Firestore, Auth, Storage), log errors and return to retry or skip:

```typescript
try {
  await doWork();
} catch (err) {
  console.error('Function failed:', err);
  // Throw to retry (Firestore/Pub/Sub retries on failure)
  // Return to skip (no retry)
}
```
