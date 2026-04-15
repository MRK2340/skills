# Firebase Authentication

Firebase Auth provides sign-in with email/password, phone, Google, GitHub, Apple, and more — with session management handled automatically.

## Enable Providers

In the Firebase console: **Authentication → Sign-in method → Enable** the providers you want.

## Web SDK (Client-Side)

### Listen for Auth State

Always use `onAuthStateChanged` — never rely on `auth.currentUser` synchronously on page load.

```typescript
import { getAuth, onAuthStateChanged } from 'firebase/auth';

const auth = getAuth();

onAuthStateChanged(auth, (user) => {
  if (user) {
    console.log('Signed in as', user.uid, user.email);
  } else {
    console.log('Signed out');
  }
});
```

### Email / Password

```typescript
import {
  getAuth,
  createUserWithEmailAndPassword,
  signInWithEmailAndPassword,
  signOut,
  sendPasswordResetEmail,
} from 'firebase/auth';

const auth = getAuth();

// Register
const { user } = await createUserWithEmailAndPassword(auth, email, password);

// Sign in
const { user } = await signInWithEmailAndPassword(auth, email, password);

// Sign out
await signOut(auth);

// Password reset
await sendPasswordResetEmail(auth, email);
```

### Google Sign-In (Popup)

```typescript
import { getAuth, GoogleAuthProvider, signInWithPopup } from 'firebase/auth';

const auth = getAuth();
const provider = new GoogleAuthProvider();
// Optional: force account selection each time
provider.setCustomParameters({ prompt: 'select_account' });

const { user } = await signInWithPopup(auth, provider);
```

### Google Sign-In (Redirect — better for mobile)

```typescript
import { getAuth, GoogleAuthProvider, signInWithRedirect, getRedirectResult } from 'firebase/auth';

const auth = getAuth();

// Initiate redirect
await signInWithRedirect(auth, new GoogleAuthProvider());

// On page load, check for redirect result
const result = await getRedirectResult(auth);
if (result) {
  const user = result.user;
}
```

### GitHub Sign-In

```typescript
import { GithubAuthProvider, signInWithPopup } from 'firebase/auth';

const provider = new GithubAuthProvider();
provider.addScope('read:user');
const { user } = await signInWithPopup(auth, provider);
```

### Phone Number (SMS)

```typescript
import { getAuth, RecaptchaVerifier, signInWithPhoneNumber } from 'firebase/auth';

const auth = getAuth();

// Set up invisible reCAPTCHA
const appVerifier = new RecaptchaVerifier(auth, 'sign-in-button', { size: 'invisible' });

// Send SMS
const confirmationResult = await signInWithPhoneNumber(auth, '+15551234567', appVerifier);

// Verify code entered by user
const { user } = await confirmationResult.confirm(smsCode);
```

### Update User Profile

```typescript
import { updateProfile, updateEmail, updatePassword } from 'firebase/auth';

await updateProfile(auth.currentUser, {
  displayName: 'Jane Doe',
  photoURL: 'https://example.com/photo.jpg',
});

await updateEmail(auth.currentUser, 'new@example.com');
await updatePassword(auth.currentUser, 'newPassword123');
```

### Get ID Token (for sending to your backend)

```typescript
const token = await auth.currentUser.getIdToken();
// Include in request headers: Authorization: Bearer <token>
```

## Admin SDK (Server-Side)

### Verify an ID Token

```typescript
import { getAuth } from 'firebase-admin/auth';

async function verifyToken(idToken: string) {
  const decodedToken = await getAuth().verifyIdToken(idToken);
  return decodedToken.uid; // guaranteed to be the authentic user ID
}
```

### Create a Custom Token

```typescript
const customToken = await getAuth().createCustomToken(uid, {
  premium: true,  // custom claims accessible in Security Rules
});
// Send to client, then: signInWithCustomToken(auth, customToken)
```

### Manage Users

```typescript
import { getAuth } from 'firebase-admin/auth';

const adminAuth = getAuth();

// Get user by UID
const user = await adminAuth.getUser(uid);

// Get user by email
const user = await adminAuth.getUserByEmail('user@example.com');

// Create user
const newUser = await adminAuth.createUser({
  email: 'user@example.com',
  password: 'password123',
  displayName: 'Jane Doe',
});

// Set custom claims (for roles/permissions)
await adminAuth.setCustomUserClaims(uid, { admin: true });

// Disable account
await adminAuth.updateUser(uid, { disabled: true });

// Delete user
await adminAuth.deleteUser(uid);
```

### List Users

```typescript
let nextPageToken: string | undefined;
do {
  const result = await adminAuth.listUsers(1000, nextPageToken);
  result.users.forEach((user) => console.log(user.uid, user.email));
  nextPageToken = result.pageToken;
} while (nextPageToken);
```

## React Hook Example

```typescript
import { useState, useEffect } from 'react';
import { getAuth, onAuthStateChanged, User } from 'firebase/auth';

export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const auth = getAuth();
    const unsubscribe = onAuthStateChanged(auth, (user) => {
      setUser(user);
      setLoading(false);
    });
    return unsubscribe; // cleanup on unmount
  }, []);

  return { user, loading };
}
```

## Security Rules with Auth

In `firestore.rules`, access authenticated user data:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Only authenticated users can read/write their own documents
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // Only admins (custom claim) can access admin collection
    match /admin/{document=**} {
      allow read, write: if request.auth.token.admin == true;
    }
  }
}
```

## Error Handling

Common auth error codes:

| Code | Meaning |
|---|---|
| `auth/user-not-found` | No user with that email |
| `auth/wrong-password` | Incorrect password |
| `auth/email-already-in-use` | Email taken at registration |
| `auth/weak-password` | Password too short (< 6 chars) |
| `auth/invalid-email` | Malformed email address |
| `auth/too-many-requests` | Account temporarily locked |
| `auth/popup-blocked` | Browser blocked the sign-in popup |
| `auth/network-request-failed` | Network error |

```typescript
import { FirebaseError } from 'firebase/app';

try {
  await signInWithEmailAndPassword(auth, email, password);
} catch (err) {
  if (err instanceof FirebaseError) {
    switch (err.code) {
      case 'auth/user-not-found':
      case 'auth/wrong-password':
        // Don't distinguish between these for security
        throw new Error('Invalid email or password');
      case 'auth/too-many-requests':
        throw new Error('Too many attempts. Try again later.');
      default:
        throw err;
    }
  }
}
```
