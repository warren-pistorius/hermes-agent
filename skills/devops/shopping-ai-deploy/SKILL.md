---
name: shopping-ai-deploy
description: ShoppingAIv2 full-stack deploy workflow — Firebase Functions + Hosting. Covers Mac (interactive dev) → GitHub → EC2 (automated deploy via vault), Firebase Auth, Functions, Hosting, and APK distribution. Warren's primary project.
version: 1.0.0
platforms: [macos, linux]
metadata:
  hermes:
    tags: [firebase, cloud-functions, hosting, apk, deploy, shoppingai]
    related_skills: [ec2-service-deploy, shoppingAIv2-logs, firebase-deploy]
---

# ShoppingAIv2 Deploy

## Project Overview

ShoppingAIv2 is Warren's shopping list assistant app (Android APK + Firebase backend). The deploy workflow spans two machines:

| Machine | Role |
|---|---|
| Mac | Dev — interactive Firebase auth, building APK |
| EC2 (`3.104.68.19`) | CI/deploy — non-interactive, uses vault tokens |

**Key files (EC2):** `~/shoppingAIv2/`

## Architecture

```
Mac (dev)          GitHub (main)        EC2 (deploy runner)
   │                    │                      │
   ├── git push ──────▶ │                      │
   │                    │                      ├── git pull
   │                    │                      ├── firebase deploy --only functions,hosting
   │                    │                      │   (token from vault)
   │                    │                      │
   │                    │                      └── APK served at http://3.104.68.19/
```

## Credentials & Vault

All secrets stored in `~/.hermes/vault/shoppingAIv2/`:

| File | Purpose |
|---|---|
| `firebase-sa.json` | GCP service account for CI/CD |
| `firebase.token` | Firebase CI token (from `firebase login:ci`) |
| `firebase-functions.env` | Functions runtime env vars |
| `.env.staging` | Staging env vars backup |

**Never put raw secrets in memory or skills — only vault paths.**

## Deploy (from Mac or any machine with repo access)

```bash
cd ~/shoppingAIv2
git push origin main
# CI on EC2 picks it up via a cron job or webhook
```

## Deploy Functions Only (from EC2)

```bash
cd ~/shoppingAIv2
firebase deploy --only functions --token "$(cat ~/.hermes/vault/shoppingAIv2/firebase.token)"
```

## APK Distribution

APK is served at `http://3.104.68.19/` (nginx serves from `/var/www/apk/` or similar). Warren distributes this URL to testers.

APK build:
```bash
cd ~/shoppingAIv2/android
./gradlew assembleDebug
# Output: app/build/outputs/apk/debug/app-debug.apk
```

## Firebase Auth Note

Firebase Auth uses Google Sign-In on Android. The SHA-1 fingerprint must be registered in the Firebase console for the app to work. This is configured on the Mac (where the Android SDK generates the fingerprint) and the result is pasted into Firebase console.

## Related Skills

- `ec2-service-deploy` — generic EC2 + nginx + systemd + SSL pattern
- `firebase-deploy` — Firebase-specific non-interactive CI/CD deploy
- `shoppingAIv2-logs` — Cloud Functions log retrieval and filtering
