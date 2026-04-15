# Firebase Realtime Database

The Realtime Database is a cloud-hosted JSON tree that syncs data to all connected clients in real time. Unlike Firestore, it has a single large JSON structure with simpler querying but extremely low latency — ideal for chat, presence, live scores, and collaborative editing.

**Prefer Firestore for most new projects.** Use Realtime Database when you need sub-100ms sync, offline support with the legacy SDK, or you're already using it.

## Web SDK (Client-Side)

### Initialize

```typescript
import { getDatabase } from 'firebase/database';
import { app } from './firebase'; // your initialized FirebaseApp

const db = getDatabase(app);
```

### Read Once

```typescript
import { getDatabase, ref, get } from 'firebase/database';

const db = getDatabase();

const snapshot = await get(ref(db, `users/${userId}`));
if (snapshot.exists()) {
  const data = snapshot.val();
  console.log(data);
} else {
  console.log('No data found');
}
```

### Write Data

```typescript
import { ref, set, update, push, remove } from 'firebase/database';

// set: overwrites the entire node
await set(ref(db, `users/${userId}`), {
  name: 'Alice',
  email: 'alice@example.com',
});

// update: merges (like setDoc with merge)
await update(ref(db, `users/${userId}`), { name: 'Alicia' });

// push: adds a child with an auto-generated key (good for lists)
const newRef = push(ref(db, 'messages'));
await set(newRef, { text: 'Hello!', timestamp: Date.now() });
console.log('New message key:', newRef.key);

// remove: deletes a node
await remove(ref(db, `users/${userId}`));
```

### Real-Time Listeners

```typescript
import { ref, onValue, onChildAdded, onChildChanged, onChildRemoved, off } from 'firebase/database';

// Listen to a node
const userRef = ref(db, `users/${userId}`);
const unsubscribe = onValue(userRef, (snapshot) => {
  const data = snapshot.val();
  console.log('User updated:', data);
});

// Stop listening
unsubscribe();

// Listen for new children only (good for chat)
const messagesRef = ref(db, 'messages');
const unsubscribeChild = onChildAdded(messagesRef, (snapshot) => {
  const message = snapshot.val();
  console.log('New message:', message);
});
```

### Querying

```typescript
import { ref, query, orderByChild, limitToLast, equalTo, startAt, endAt, get } from 'firebase/database';

// Get last 50 messages ordered by timestamp
const q = query(
  ref(db, 'messages'),
  orderByChild('timestamp'),
  limitToLast(50)
);
const snapshot = await get(q);
snapshot.forEach((child) => {
  console.log(child.key, child.val());
});

// Filter: users with score >= 100
const q2 = query(
  ref(db, 'users'),
  orderByChild('score'),
  startAt(100)
);
```

**Realtime Database query limitations:**
- You can only order/filter by one field at a time.
- No composite queries. Denormalize data or use Firestore for complex queries.

### Transactions

For atomic counter increments or any read-modify-write operation:

```typescript
import { ref, runTransaction } from 'firebase/database';

await runTransaction(ref(db, `posts/${postId}/likes`), (currentLikes) => {
  return (currentLikes ?? 0) + 1;
});
```

### Presence (Online/Offline Detection)

```typescript
import { ref, onValue, onDisconnect, set, serverTimestamp } from 'firebase/database';
import { getDatabase } from 'firebase/database';

const db = getDatabase();
const connectedRef = ref(db, '.info/connected');

onValue(connectedRef, (snap) => {
  if (snap.val() === true) {
    const userStatusRef = ref(db, `status/${userId}`);

    // When this client disconnects, mark as offline
    onDisconnect(userStatusRef).set({ online: false, lastSeen: serverTimestamp() });

    // Mark as online now
    set(userStatusRef, { online: true });
  }
});
```

## Admin SDK (Server-Side)

```typescript
import { getDatabase } from 'firebase-admin/database';

const db = getDatabase();

// Read
const snap = await db.ref(`users/${userId}`).once('value');
const data = snap.val();

// Write
await db.ref(`users/${userId}`).set({ name: 'Alice' });
await db.ref(`users/${userId}`).update({ name: 'Alicia' });

// Push to list
const newRef = db.ref('messages').push();
await newRef.set({ text: 'Hello', timestamp: Date.now() });

// Delete
await db.ref(`users/${userId}`).remove();

// Transaction
await db.ref(`counters/visits`).transaction((current) => {
  return (current ?? 0) + 1;
});
```

## Security Rules

Realtime Database rules use a JSON structure mirroring the database tree.

```json
{
  "rules": {
    "users": {
      "$userId": {
        ".read": "auth != null && auth.uid === $userId",
        ".write": "auth != null && auth.uid === $userId"
      }
    },
    "messages": {
      ".read": "auth != null",
      "$messageId": {
        ".write": "auth != null",
        ".validate": "newData.hasChildren(['text', 'timestamp']) && newData.child('text').isString()"
      }
    },
    "status": {
      "$userId": {
        ".read": "auth != null",
        ".write": "auth != null && auth.uid === $userId"
      }
    }
  }
}
```

Deploy: `firebase deploy --only database`

## Data Structuring Tips

Flatten data — avoid deeply nested structures. Instead of storing posts inside users, keep separate top-level nodes and use IDs to link them:

```json
{
  "users": {
    "uid1": { "name": "Alice" }
  },
  "posts": {
    "postId1": { "title": "Hello", "authorId": "uid1" }
  },
  "userPosts": {
    "uid1": { "postId1": true }
  }
}
```

This lets you load a user's post IDs without downloading all post data.
