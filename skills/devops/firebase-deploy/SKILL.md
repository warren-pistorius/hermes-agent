---
name: firebase-deploy
description: Non-interactive Firebase Functions and Hosting deploy from CI/CD environments. Covers auth, IAM, environment scoping, App Distribution, and common failure modes.
trigger: Deploying Firebase functions from a non-interactive environment, uploading APKs to App Distribution, or running firebase CLI in automated pipelines.
---
# Firebase Non-Interactive Deploy

Deploy Firebase functions or hosting from a machine without a human logged in (EC2, CI runner, Cloud Build).

## Two-Layer Architecture

Firebase deploy has **two independent auth problems**:

| Layer | What it needs | How to provide |
|-------|---------------|----------------|
| **Runtime secrets** (functions read this at runtime) | API keys, Stripe keys, config values | `.env` files + `firebase use <project>` to swap envs |
| **Deploy auth** (CLI needs this to upload code) | Identity to GCP | `FIREBASE_TOKEN` (CI token) — no other method works |

**Critical**: `GOOGLE_APPLICATION_CREDENTIALS` and gcloud SA credentials do **NOT** work for `firebase deploy`. Firebase CLI uses its own auth layer, not ADC.

## Deploy Auth Options

### Option 1: CI Token (recommended for most non-interactive setups)

```bash
# Generate: firebase login:ci
# Store vault path: ~/.hermes/vault/{project}/firebase.token
FIREBASE_TOKEN=$(cat ~/.hermes/vault/{project}/firebase.token)
cd ~/shoppingAIv2  # project root (where firebase.json lives)
firebase deploy --only functions:{functionName} --project {projectId}
```

- Generated via `firebase login:ci` on a machine with interactive Firebase login
- Lives in vault — never in plain text or memory files
- Works in any environment (EC2, GitHub Actions, local cron)

### Option 2: gcloud CLI Installation (for GCP tools + SA auth)

```bash
# Install gcloud CLI (use tar.gz, NOT the web installer which returns HTML)
cd /tmp
wget -q https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-496.0.0-linux-x86_64.tar.gz
tar -xf google-cloud-cli-*.tar.gz
./google-cloud-sdk/install.sh --quiet
mv google-cloud-sdk ~/

# Authenticate as SA (for GCP-native tools, NOT for firebase deploy)
~/google-cloud-sdk/bin/gcloud auth activate-service-account \
  --key-file=/path/to/sa-key.json

# Add to PATH permanently
echo 'export PATH="$HOME/google-cloud-sdk/bin:$PATH"' >> ~/.bashrc
```

**Note**: gcloud SA auth does NOT make `firebase deploy` work — Firebase CLI uses its own auth layer, not ADC. Use `FIREBASE_TOKEN` for firebase deploy.

### Option 3: Interactive user login (Mac/dev machines only)
### Option 3: Interactive user login (Mac/dev machines only)
```bash
firebase login  # interactive, requires human
firebase deploy
```

## App Distribution (APK/AAB Upload)

Firebase CLI also supports **Firebase App Distribution** — upload APKs/AABs to testers without needing the Google Play pipeline.

```bash
export FIREBASE_TOKEN=$(cat ~/.hermes/vault/{project}/firebase.token)

# List groups
firebase appdistribution:groups:list --project {projectId}

# Distribute to a group
firebase appdistribution:distribute /path/to/app-release.apk \
  --app 1:72313916021:android:0dfeaeb616aaf27761063c \
  --groups "Internal" \
  --release-notes "Bug fixes and improvements"

# Distribute to testers only (no group)
firebase appdistribution:distribute /path/to/app-release.apk \
  --app 1:72313916021:android:0dfeaeb616aaf27761063c \
  --release-notes "Bug fixes"
```

**ShoppingAIv2 App IDs:**
- Android: `1:72313916021:android:0dfeaeb616aaf27761063c`
- iOS: `1:72313916021:ios:f65bd50c8e7d940561063c`

**Warren's deploy-android.js script** (`~/shoppingAIv2/scripts/deploy-android.js`) builds locally then calls App Distribution automatically — it also uses the same `npx firebase-tools appdistribution:distribute` pattern with the `FIREBASE_TOKEN` inherited from the shell environment.

**Note:** EAS Build cannot run on EC2 (needs Expo credentials + signing keys on a developer machine). EC2 App Distribution flow: build locally on Mac → copy APK to EC2 or trigger EAS cloud build → use this script to upload to App Distribution.

## IAM Permissions (Common Failure Mode)

If deploy fails with:
```
Missing permissions required for functions deploy.
You must have permission iam.serviceAccounts.ActAs on service account {projectId}@appspot.gserviceaccount.com.
```

**The fix is NOT adding roles to your deploy SA's key itself.** You need:

1. The SA doing deploy (identified by the JSON key's `client_email`) needs `roles/iam.serviceAccountUser` on the **appspot default SA**: `{projectId}@appspot.gserviceaccount.com`

2. To add this permission:
   - Go to [console.cloud.google.com/iam-admin/iam?project={projectId}](https://console.cloud.google.com/iam-admin/iam?project={projectId})
   - Find `{projectId}@appspot.gserviceaccount.com` in the principals list
   - Click Edit → Add role → **Service Account User** (`roles/iam.serviceAccountUser`)
   - The deploy SA's key email is NOT the place to add roles

**Why this trips people up**: The error says "ActAs on appspot SA" — people add the role to their deploy SA, but the role needed is `serviceAccountUser` **on the target (appspot) SA**, not on the deploy SA itself.

## Runtime Config: dotenv + firebase use

For ShoppingAIv2, the project structure uses dotenv for runtime secrets:

```bash
# Project structure
shoppingAIv2/
├── firebase-functions/
│   ├── .env           # Shared secrets: API keys, haiku model, GA4
│   ├── .env.staging   # Stripe test keys
│   └── .env.production # Stripe live keys
└── firebase.json      # Must be at project root

# Deploy with .env loaded
cd ~/shoppingAIv2
firebase use staging   # loads .env + .env.staging
firebase deploy --only functions:refreshItemPrice --project shoppingai-b9b20
```

## Environment Switching

```bash
firebase use production   # swaps to production aliases + .env.production
firebase use staging      # swaps to staging aliases + .env + .env.staging
firebase use default      # swap back
```

**⚠️ Confirm target environment before any deploy involving Stripe.** The wrong Stripe key in the wrong environment = either accidentally charging real money (test keys in production) or payments failing silently (live keys in staging). Always ask:

> "Is this STAGING or PRODUCTION?"
> - STAGING → Stripe TEST keys (`sk_test_...`)
> - PRODUCTION → Stripe LIVE keys (`sk_live_...`)

If the user doesn't specify, assume **staging** and confirm. Never deploy to production on an ambiguous request.

## Quick Reference

```bash
# Full deploy (all functions)
firebase deploy --project shoppingai-b9b20

# Single function
firebase deploy --only functions:refreshItemPrice --project shoppingai-b9b20

# With CI token
FIREBASE_TOKEN=$(cat ~/.hermes/vault/shoppingAIv2/firebase.token) firebase deploy ...

# Verify logged-in account
firebase projects:list   # shows active project/auth
```

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `firebase: command not found` | Firebase CLI not installed | `npm install -g firebase-tools` |
| `Error: Not in a Firebase app directory` | Wrong working directory | `cd` to project root (where firebase.json lives) |
| `Cloud Billing API has not been used in project` | Fresh GCP project, billing API not enabled | Visit `console.developers.google.com/apis/api/cloudbilling.googleapis.com/overview?project={projectId}` and click Enable. No billing commitment required — just a one-time API activation. |
| `Missing permissions...ActAs on appspot` | Wrong IAM target | Add `serviceAccountUser` to appspot SA, not deploy SA |
| `Failed to authenticate` | No auth method available | Use `FIREBASE_TOKEN` env var |
| `You must run firebase login first` | Token expired or not set | Re-generate with `firebase login:ci` |
