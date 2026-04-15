---
name: firebase
description: Build, deploy, and manage Firebase applications. TRIGGER when the user mentions Firebase, Firestore, Firebase Auth, Firebase Storage, Firebase Hosting, Cloud Functions for Firebase, Firebase Realtime Database, or any Firebase SDK. Use this skill whenever the user wants to add Firebase to a project, query Firestore, handle authentication with Firebase, upload files to Firebase Storage, deploy with Firebase Hosting, or write Cloud Functions — even if they just say "add Firebase" or "connect to Firestore" without further detail.
---

# Firebase

This skill covers the full Firebase platform — from project setup to production deployment. Firebase is Google's app development platform that provides backend services (auth, database, storage, hosting, serverless functions) with SDKs for web, iOS, Android, and server environments.

## Quick Service Reference

| Service | Use For | Reference File |
|---|---|---|
| Firebase Auth | User sign-in (email, Google, phone, etc.) | `references/authentication.md` |
| Firestore | Flexible NoSQL document database | `references/firestore.md` |
| Realtime Database | Low-latency synced JSON database | `references/realtime-database.md` |
| Cloud Storage | File uploads (images, videos, docs) | `references/storage.md` |
| Firebase Hosting | Static site + SPA deployment | `references/hosting.md` |
| Cloud Functions | Serverless backend triggers & APIs | `references/cloud-functions.md` |

## Getting Started

If the user needs initial project setup, SDK installation, or Firebase config, read `references/setup.md` first.

## How to Use This Skill

1. **Identify the service(s)** the user needs from the table above.
2. **Read `references/setup.md`** if the project isn't already configured.
3. **Read the relevant reference file(s)** for the target service(s).
4. **Detect the language/framework** (see below) and use the appropriate SDK patterns.

## Language & Framework Detection

| Project Signals | Environment | SDK |
|---|---|---|
| `package.json` with React/Next/Vue/Svelte | Web (client) | `firebase` npm package |
| `package.json` with `firebase-admin` | Node.js server | `firebase-admin` npm package |
| `*.py` + `firebase-admin` import | Python server | `firebase-admin` PyPI package |
| `go.mod` + `firebase.google.com/go` | Go server | Firebase Admin Go SDK |
| `AndroidManifest.xml` / `*.kt` / `*.java` | Android | Firebase Android SDK |
| `*.swift` / `*.xcodeproj` | iOS | Firebase Apple SDK |
| `functions/` directory | Cloud Functions | `firebase-functions` npm package |

If the language is ambiguous, check for a `firebase.json` or `.firebaserc` to confirm this is an existing Firebase project, then ask the user which environment they're targeting.

## Key Principles

- **Security rules matter**: Firestore and Storage both require Security Rules. Always write rules alongside data access code — never leave the default open rules in production.
- **Use the Admin SDK server-side**: The client SDK is for browsers/mobile apps. For server code (Cloud Functions, scripts, backends), always use `firebase-admin`.
- **Emulators first**: The Firebase Local Emulator Suite (`firebase emulators:start`) lets you develop without hitting production. Recommend it for any non-trivial project.
- **Modular SDK (v9+)**: The modern Firebase Web SDK uses tree-shakeable imports (`import { getFirestore } from 'firebase/firestore'`). Prefer modular imports over the older namespaced style.

## Common Pitfalls

- Don't expose the service account key in client-side code — use it only in server/Cloud Functions environments.
- Firestore queries require composite indexes for multi-field filters; the error message will include a direct link to create the index.
- Cloud Functions cold starts add latency — use `runWith({ minInstances: 1 })` sparingly for latency-sensitive functions.
- Firebase Hosting serves static files only; dynamic server-side rendering requires pairing Hosting with Cloud Functions or Cloud Run.
- The client SDK `firebase/auth` `currentUser` may be `null` on initial load before the auth state is restored — always use `onAuthStateChanged` to wait for the resolved state.
