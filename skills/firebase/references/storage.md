# Firebase Cloud Storage

Firebase Storage lets you store and serve user-generated content — images, videos, audio, PDFs, and any binary files. Files are stored in Google Cloud Storage buckets with Firebase Security Rules controlling access.

## Web SDK (Client-Side)

### Initialize

```typescript
import { getStorage } from 'firebase/storage';
import { app } from './firebase';

const storage = getStorage(app);
```

### Upload a File

```typescript
import { getStorage, ref, uploadBytes, uploadBytesResumable, getDownloadURL } from 'firebase/storage';

const storage = getStorage();

// Simple upload (no progress)
const fileRef = ref(storage, `uploads/${userId}/${file.name}`);
await uploadBytes(fileRef, file);
const downloadURL = await getDownloadURL(fileRef);
console.log('File available at:', downloadURL);
```

### Upload with Progress Tracking

```typescript
import { uploadBytesResumable, getDownloadURL } from 'firebase/storage';

function uploadWithProgress(file: File, userId: string): Promise<string> {
  return new Promise((resolve, reject) => {
    const fileRef = ref(storage, `images/${userId}/${Date.now()}_${file.name}`);
    const task = uploadBytesResumable(fileRef, file, {
      contentType: file.type,
    });

    task.on(
      'state_changed',
      (snapshot) => {
        const progress = (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
        console.log(`Upload: ${progress.toFixed(0)}%`);
      },
      (error) => reject(error),
      async () => {
        const url = await getDownloadURL(task.snapshot.ref);
        resolve(url);
      }
    );
  });
}
```

### Upload from a Data URL or Blob

```typescript
import { ref, uploadString } from 'firebase/storage';

// From a base64 data URL (e.g. from canvas.toDataURL())
const dataUrl = 'data:image/png;base64,iVBORw0KGgo...';
await uploadString(ref(storage, 'images/canvas.png'), dataUrl, 'data_url');

// From a Blob
const blob = await fetch(url).then((r) => r.blob());
await uploadBytes(ref(storage, 'files/download.pdf'), blob);
```

### Download / Get Download URL

```typescript
import { ref, getDownloadURL } from 'firebase/storage';

const url = await getDownloadURL(ref(storage, `images/${userId}/avatar.jpg`));
// Use url in an <img src={url} /> or anchor href
```

### List Files

```typescript
import { ref, listAll, list } from 'firebase/storage';

// List all files in a directory
const dirRef = ref(storage, `uploads/${userId}`);
const result = await listAll(dirRef);

result.items.forEach((itemRef) => console.log(itemRef.name, itemRef.fullPath));
result.prefixes.forEach((folderRef) => console.log('Subdirectory:', folderRef.name));

// Paginated list (for large directories)
const page = await list(dirRef, { maxResults: 20, pageToken: nextPageToken });
const nextToken = page.nextPageToken; // undefined if last page
```

### Delete a File

```typescript
import { ref, deleteObject } from 'firebase/storage';

await deleteObject(ref(storage, `uploads/${userId}/old-file.jpg`));
```

### Get File Metadata

```typescript
import { ref, getMetadata, updateMetadata } from 'firebase/storage';

const metadata = await getMetadata(ref(storage, `uploads/${userId}/file.jpg`));
console.log(metadata.contentType, metadata.size, metadata.timeCreated);

// Update metadata (e.g., set cache control)
await updateMetadata(ref(storage, path), {
  cacheControl: 'public, max-age=31536000',
  customMetadata: { uploadedBy: userId },
});
```

### React File Upload Component Example

```tsx
import { useState, useRef } from 'react';
import { getStorage, ref, uploadBytesResumable, getDownloadURL } from 'firebase/storage';

export function FileUpload({ userId }: { userId: string }) {
  const [progress, setProgress] = useState(0);
  const [url, setUrl] = useState('');
  const inputRef = useRef<HTMLInputElement>(null);

  async function handleUpload(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0];
    if (!file) return;

    const storage = getStorage();
    const fileRef = ref(storage, `uploads/${userId}/${file.name}`);
    const task = uploadBytesResumable(fileRef, file);

    task.on('state_changed',
      (snap) => setProgress((snap.bytesTransferred / snap.totalBytes) * 100),
      (err) => console.error(err),
      async () => setUrl(await getDownloadURL(task.snapshot.ref))
    );
  }

  return (
    <div>
      <input ref={inputRef} type="file" onChange={handleUpload} />
      {progress > 0 && <progress value={progress} max={100} />}
      {url && <img src={url} alt="Uploaded file" />}
    </div>
  );
}
```

## Admin SDK (Server-Side)

The Admin SDK uses the `@google-cloud/storage` client under the hood:

```typescript
import { getStorage } from 'firebase-admin/storage';

const bucket = getStorage().bucket(); // default bucket

// Upload a local file
await bucket.upload('/path/to/local/file.pdf', {
  destination: `uploads/${userId}/file.pdf`,
  metadata: { contentType: 'application/pdf' },
});

// Upload from a buffer
const file = bucket.file(`uploads/${userId}/image.png`);
await file.save(buffer, { contentType: 'image/png' });

// Generate a signed URL (time-limited download link)
const [url] = await file.getSignedUrl({
  action: 'read',
  expires: Date.now() + 60 * 60 * 1000, // 1 hour
});

// Download to buffer
const [data] = await file.download();

// Delete
await file.delete();

// List files
const [files] = await bucket.getFiles({ prefix: `uploads/${userId}/` });
files.forEach((f) => console.log(f.name));
```

## Security Rules

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {

    // Users can only access their own uploads
    match /uploads/{userId}/{allPaths=**} {
      allow read: if request.auth != null;
      allow write: if request.auth != null
        && request.auth.uid == userId
        && request.resource.size < 10 * 1024 * 1024  // 10 MB max
        && request.resource.contentType.matches('image/.*');
    }

    // Public read for profile pictures
    match /avatars/{userId} {
      allow read: if true;
      allow write: if request.auth != null
        && request.auth.uid == userId
        && request.resource.size < 2 * 1024 * 1024   // 2 MB max
        && request.resource.contentType.matches('image/(jpeg|png|webp)');
    }
  }
}
```

Deploy: `firebase deploy --only storage`

## Common Patterns

### Image Resizing with Cloud Functions

Use the `@google-cloud/functions-framework` + `sharp` in a Cloud Function triggered by Storage uploads to generate thumbnails:

```typescript
// In Cloud Functions (see cloud-functions.md for trigger setup)
import * as functions from 'firebase-functions';
import { getStorage } from 'firebase-admin/storage';
import sharp from 'sharp';
import path from 'path';

export const generateThumbnail = functions.storage.object().onFinalize(async (object) => {
  const filePath = object.name!;
  if (!filePath.startsWith('uploads/') || filePath.includes('thumb_')) return;

  const bucket = getStorage().bucket(object.bucket);
  const tempPath = `/tmp/${path.basename(filePath)}`;

  await bucket.file(filePath).download({ destination: tempPath });

  const thumbPath = `/tmp/thumb_${path.basename(filePath)}`;
  await sharp(tempPath).resize(200, 200, { fit: 'inside' }).toFile(thumbPath);

  const thumbDest = `uploads/thumbnails/thumb_${path.basename(filePath)}`;
  await bucket.upload(thumbPath, { destination: thumbDest });
});
```

### CORS Configuration (for direct browser access)

If browser fetch to storage URLs fails with CORS errors, configure CORS on the bucket:

Create `cors.json`:
```json
[
  {
    "origin": ["https://your-app.com", "http://localhost:3000"],
    "method": ["GET", "HEAD"],
    "maxAgeSeconds": 3600
  }
]
```

Apply:
```bash
gcloud storage buckets update gs://your-project.appspot.com --cors-file=cors.json
```
