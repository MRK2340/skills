# Cloud Firestore

Firestore is a flexible, scalable NoSQL document database. Data is organized into **collections** of **documents**. Documents contain key-value fields and can contain subcollections.

## Data Model

```
/users/{userId}                    ← document
  name: "Alice"
  email: "alice@example.com"
  /posts/{postId}                  ← subcollection document
    title: "Hello World"
    createdAt: Timestamp

/products/{productId}
  name: "Widget"
  price: 9.99
  tags: ["sale", "featured"]       ← array field
  dimensions: { w: 10, h: 5 }     ← map field
```

## Web SDK (Client-Side)

### Read a Document

```typescript
import { getFirestore, doc, getDoc } from 'firebase/firestore';

const db = getFirestore();

const docRef = doc(db, 'users', userId);
const snap = await getDoc(docRef);

if (snap.exists()) {
  const data = snap.data();        // typed: DocumentData
  console.log(data.name, data.email);
} else {
  console.log('Document not found');
}
```

### Write / Set a Document

```typescript
import { doc, setDoc, updateDoc, serverTimestamp } from 'firebase/firestore';

// setDoc: creates or overwrites the entire document
await setDoc(doc(db, 'users', userId), {
  name: 'Alice',
  email: 'alice@example.com',
  createdAt: serverTimestamp(),
});

// setDoc with merge: only updates provided fields
await setDoc(doc(db, 'users', userId), { name: 'Alicia' }, { merge: true });

// updateDoc: updates specific fields (document must exist)
await updateDoc(doc(db, 'users', userId), {
  name: 'Alicia',
  updatedAt: serverTimestamp(),
});
```

### Add a Document (auto-generated ID)

```typescript
import { collection, addDoc } from 'firebase/firestore';

const docRef = await addDoc(collection(db, 'posts'), {
  title: 'Hello',
  body: 'World',
  createdAt: serverTimestamp(),
});
console.log('New doc ID:', docRef.id);
```

### Delete a Document

```typescript
import { deleteDoc } from 'firebase/firestore';

await deleteDoc(doc(db, 'users', userId));
```

### Field Operations

```typescript
import { updateDoc, arrayUnion, arrayRemove, increment, deleteField } from 'firebase/firestore';

await updateDoc(docRef, {
  tags: arrayUnion('featured'),     // add to array (no duplicates)
  tags: arrayRemove('sale'),        // remove from array
  viewCount: increment(1),          // atomic increment
  oldField: deleteField(),          // remove a field
});
```

### Query a Collection

```typescript
import { collection, query, where, orderBy, limit, getDocs } from 'firebase/firestore';

const q = query(
  collection(db, 'products'),
  where('price', '<', 50),
  where('inStock', '==', true),
  orderBy('price', 'asc'),
  limit(20)
);

const snapshot = await getDocs(q);
snapshot.forEach((doc) => {
  console.log(doc.id, doc.data());
});

// Or as an array
const products = snapshot.docs.map((doc) => ({ id: doc.id, ...doc.data() }));
```

### Query Operators

| Operator | Usage |
|---|---|
| `==` | Equality |
| `!=` | Inequality |
| `<`, `<=`, `>`, `>=` | Comparison |
| `in` | Value in array: `where('status', 'in', ['active', 'pending'])` |
| `not-in` | Value not in array |
| `array-contains` | Array contains value: `where('tags', 'array-contains', 'sale')` |
| `array-contains-any` | Array contains any of: `where('tags', 'array-contains-any', ['sale', 'new'])` |

### Real-Time Listeners

```typescript
import { onSnapshot } from 'firebase/firestore';

// Listen to a document
const unsubscribe = onSnapshot(doc(db, 'users', userId), (snap) => {
  if (snap.exists()) {
    console.log('Updated data:', snap.data());
  }
});

// Listen to a collection query
const q = query(collection(db, 'messages'), orderBy('createdAt', 'desc'), limit(50));
const unsubscribe = onSnapshot(q, (snapshot) => {
  snapshot.docChanges().forEach((change) => {
    if (change.type === 'added') console.log('New:', change.doc.data());
    if (change.type === 'modified') console.log('Modified:', change.doc.data());
    if (change.type === 'removed') console.log('Removed:', change.doc.data());
  });
});

// Call unsubscribe() to stop listening (e.g., in useEffect cleanup)
```

### Transactions

Use transactions when a write depends on a read — guarantees atomicity.

```typescript
import { runTransaction } from 'firebase/firestore';

await runTransaction(db, async (transaction) => {
  const docRef = doc(db, 'counters', 'pageViews');
  const snap = await transaction.get(docRef);
  const current = snap.exists() ? snap.data().count : 0;
  transaction.set(docRef, { count: current + 1 });
});
```

### Batch Writes

Commit up to 500 writes atomically (no reads).

```typescript
import { writeBatch } from 'firebase/firestore';

const batch = writeBatch(db);

batch.set(doc(db, 'users', 'user1'), { name: 'Alice' });
batch.update(doc(db, 'users', 'user2'), { status: 'active' });
batch.delete(doc(db, 'users', 'user3'));

await batch.commit();
```

### Pagination

```typescript
import { query, orderBy, startAfter, limit, getDocs } from 'firebase/firestore';

let lastDoc: DocumentSnapshot | null = null;

async function loadPage() {
  let q = query(collection(db, 'products'), orderBy('name'), limit(10));
  if (lastDoc) {
    q = query(collection(db, 'products'), orderBy('name'), startAfter(lastDoc), limit(10));
  }
  const snap = await getDocs(q);
  lastDoc = snap.docs[snap.docs.length - 1] ?? null;
  return snap.docs.map((d) => ({ id: d.id, ...d.data() }));
}
```

## Admin SDK (Server-Side)

```typescript
import { getFirestore, FieldValue } from 'firebase-admin/firestore';

const db = getFirestore();

// Read
const snap = await db.collection('users').doc(userId).get();
const data = snap.data();

// Write
await db.collection('users').doc(userId).set({ name: 'Alice' }, { merge: true });

// Query
const snapshot = await db
  .collection('products')
  .where('price', '<', 50)
  .orderBy('price')
  .limit(20)
  .get();

const products = snapshot.docs.map((d) => ({ id: d.id, ...d.data() }));

// Server timestamp
await db.collection('logs').add({
  event: 'login',
  createdAt: FieldValue.serverTimestamp(),
});

// Batch
const batch = db.batch();
batch.set(db.collection('users').doc('u1'), { name: 'Alice' });
batch.delete(db.collection('users').doc('u2'));
await batch.commit();
```

## Security Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users can only read/write their own profile
    match /users/{userId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.auth.uid == userId;
    }

    // Posts are public to read, only author can write
    match /posts/{postId} {
      allow read: if true;
      allow create: if request.auth != null
        && request.resource.data.authorId == request.auth.uid;
      allow update, delete: if request.auth != null
        && resource.data.authorId == request.auth.uid;
    }

    // Validate data shape on create
    match /products/{productId} {
      allow create: if request.auth.token.admin == true
        && request.resource.data.price is number
        && request.resource.data.price > 0
        && request.resource.data.name is string;
    }
  }
}
```

Deploy rules: `firebase deploy --only firestore:rules`

## Indexes

- **Single-field indexes**: created automatically.
- **Composite indexes** (multiple fields in a query): create via the link in the error message, or in `firestore.indexes.json`:

```json
{
  "indexes": [
    {
      "collectionGroup": "products",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "category", "order": "ASCENDING" },
        { "fieldPath": "price", "order": "ASCENDING" }
      ]
    }
  ]
}
```

Deploy: `firebase deploy --only firestore:indexes`

## TypeScript Typing

```typescript
import { DocumentData, FirestoreDataConverter, QueryDocumentSnapshot } from 'firebase/firestore';

interface Product {
  id: string;
  name: string;
  price: number;
  inStock: boolean;
}

const productConverter: FirestoreDataConverter<Product> = {
  toFirestore(product: Product): DocumentData {
    return { name: product.name, price: product.price, inStock: product.inStock };
  },
  fromFirestore(snap: QueryDocumentSnapshot): Product {
    const data = snap.data();
    return { id: snap.id, name: data.name, price: data.price, inStock: data.inStock };
  },
};

// Use the converter
const ref = doc(db, 'products', productId).withConverter(productConverter);
const snap = await getDoc(ref);
const product = snap.data(); // typed as Product
```
