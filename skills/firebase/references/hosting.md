# Firebase Hosting

Firebase Hosting delivers static assets (HTML, CSS, JS, images) and single-page apps from a global CDN with automatic SSL, custom domains, and preview deployments. It can also route requests to Cloud Functions or Cloud Run for dynamic content.

## Quick Deploy

```bash
# One-time setup (if not already done)
firebase init hosting

# Build your app, then deploy
npm run build
firebase deploy --only hosting
```

Your site is live at `https://your-project.web.app` and `https://your-project.firebaseapp.com`.

## firebase.json Configuration

```json
{
  "hosting": {
    "public": "dist",           // folder containing built files (dist, build, out, .next/out, etc.)
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**"
    ],
    "cleanUrls": true,          // serve /about instead of /about.html
    "trailingSlash": false,
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"   // SPA fallback: all routes → index.html
      }
    ]
  }
}
```

### SPA Routing

For React, Vue, Angular, and other SPAs, add the catch-all rewrite so client-side routing works:

```json
"rewrites": [
  { "source": "**", "destination": "/index.html" }
]
```

### Rewrites to Cloud Functions / Cloud Run

```json
"rewrites": [
  { "source": "/api/**", "function": "api" },
  { "source": "/ssr/**", "run": { "serviceId": "ssr-service", "region": "us-central1" } }
]
```

### Headers (Cache Control, Security)

```json
"headers": [
  {
    "source": "**/*.@(js|css|woff2)",
    "headers": [{ "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }]
  },
  {
    "source": "**",
    "headers": [
      { "key": "X-Frame-Options", "value": "SAMEORIGIN" },
      { "key": "X-Content-Type-Options", "value": "nosniff" },
      { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" }
    ]
  }
]
```

### Redirects

```json
"redirects": [
  {
    "source": "/old-path",
    "destination": "/new-path",
    "type": 301
  },
  {
    "source": "/blog/:slug",
    "destination": "/posts/:slug",
    "type": 302
  }
]
```

## Custom Domains

1. In the Firebase console: **Hosting → Add custom domain**
2. Add the provided DNS records (A records or CNAME) to your domain registrar
3. Firebase provisions an SSL certificate automatically (can take up to 24h for DNS propagation)

## Preview Channels (Staging Deployments)

Deploy to a temporary URL for testing without affecting production:

```bash
firebase hosting:channel:deploy staging
# → https://your-project--staging-abc123.web.app

# List channels
firebase hosting:channel:list

# Delete a channel
firebase hosting:channel:delete staging
```

### GitHub Actions Integration

Auto-deploy preview channels on PRs and production on merge:

```bash
firebase init hosting:github
```

This creates `.github/workflows/firebase-hosting-*.yml` files automatically.

Example workflow (manually written):

```yaml
# .github/workflows/deploy.yml
name: Deploy to Firebase Hosting
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci && npm run build

      - name: Deploy to production (main only)
        if: github.ref == 'refs/heads/main'
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          channelId: live
          projectId: your-project

      - name: Deploy preview (PRs)
        if: github.event_name == 'pull_request'
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          projectId: your-project
```

## Rollback

```bash
# List deploy history
firebase hosting:releases:list

# Roll back to the previous release
firebase hosting:rollback
```

## Multiple Sites (Monorepo)

Host multiple apps in one Firebase project:

```json
{
  "hosting": [
    {
      "target": "app",
      "public": "packages/app/dist",
      "rewrites": [{ "source": "**", "destination": "/index.html" }]
    },
    {
      "target": "admin",
      "public": "packages/admin/dist",
      "rewrites": [{ "source": "**", "destination": "/index.html" }]
    }
  ]
}
```

Set targets in `.firebaserc`:

```json
{
  "projects": { "default": "your-project" },
  "targets": {
    "your-project": {
      "hosting": {
        "app": ["your-project-app"],
        "admin": ["your-project-admin"]
      }
    }
  }
}
```

Deploy a specific target:

```bash
firebase deploy --only hosting:app
```

## Next.js / SSR with Firebase

For Next.js with static export (`output: 'export'`), Firebase Hosting works directly. For server-side rendering, use one of:

1. **Cloud Functions**: Use `next-on-firebase` or a custom function that runs the Next.js server.
2. **Cloud Run**: Containerize the Next.js app and point Hosting rewrites at Cloud Run.
3. **Firebase App Hosting** (newer, recommended for Next.js): Managed hosting that handles SSR, streaming, and ISR natively.

```bash
# Firebase App Hosting (Next.js, Angular, etc.)
firebase init apphosting
```

This creates a `apphosting.yaml` and connects to your GitHub repo for CI/CD.
