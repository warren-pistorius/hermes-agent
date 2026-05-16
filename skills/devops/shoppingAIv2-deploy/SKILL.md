---
name: shoppingAIv2-deploy
description: Use when deploying ShoppingAIv2 Firebase Cloud Functions. Handles both EC2 (vault-based) and Mac (interactive auth) deploy workflows.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [firebase, deploy, shoppingAIv2, cloud-functions]
    related_skills: [shoppingAIv2-logs, shoppingAIv2-build-android]
---

# ShoppingAIv2 Deploy

## Overview

Deploys Firebase Cloud Functions for the ShoppingAIv2 (Shopping Mate) app. Two deployment environments exist: **EC2** (vault-based auth, automated) and **Mac** (interactive Firebase auth).

## When to Use

- User says "deploy ShoppingAIv2" or "push functions"
- After pulling changes that modify Cloud Functions
- After updating `.env` secrets for Firebase Functions

## ⚠️ Environment Confirmation (Critical)

**ALWAYS confirm target environment before deploying.** Stripe live keys in staging = accidentally charging real money. Stripe test keys in production = payments fail silently.

### Required Step: Ask Before Deploying
```
Is this a STAGING or PRODUCTION deploy?

- STAGING → Stripe TEST keys (safe, no real charges)
- PRODUCTION → Stripe LIVE keys (real payments enabled)

Default: staging (safer). Only proceed to production on explicit request.
```

If user says "deploy", "push", "ship it" without specifying → assume **staging** and confirm.

### Environment Key Differences

| | Staging | Production |
|--|--|--|
| Stripe keys | `sk_test_...` (test mode) | `sk_live_...` (real charges) |
| GA4 | Test property | Live property |
| `firebase use` alias | `staging` | `production` |
| `.env` files loaded | `.env` + `.env.staging` | `.env` + `.env.production` |

## EC2 Deploy (this agent — Warren's EC2 box)

### Prerequisites
- `.env` file exists in `firebase-functions/` (loaded automatically by Firebase CLI)
- Stripe keys depend on target environment (staging vs production)
- Active Firebase project: `shoppingai-b9b20`

### Deploy Command
```bash
cd ~/shoppingAIv2/firebase-functions
firebase use staging  # loads .env + .env.staging (Stripe test keys)
firebase deploy --only functions --project shoppingai-b9b20
```

Or for production:
```bash
firebase use production  # loads .env + .env.production (Stripe live keys)
firebase deploy --only functions --project shoppingai-b9b20
```

Note: No `FIREBASE_TOKEN` needed — secrets come from `.env` files, not `functions.config()`.

### Deploy Specific Functions
```bash
# Single function
firebase deploy --only functions:refreshItemPrice --project shoppingai-b9b20

# Multiple functions
firebase deploy --only functions:analyzeGroceryItem,functions:refreshItemPrice --project shoppingai-b9b20
```

### Verify
Check deployment success in output: `✔ functions[functionName] Successful update operation`

---

## Mac Deploy (Warren's machine)

### Prerequisites
- Firebase CLI authenticated: `firebase login`
- Active project: `shoppingai-b9b20`
- `.env` file exists in `firebase-functions/` (get from 1Password or create from `.env.example`)

### Deploy Command
```bash
cd ~/shoppingAIv2/firebase-functions
firebase deploy --only functions --project shoppingai-b9b20
```

### Create .env from Template
If `.env` is missing on Mac:
```bash
cd ~/shoppingAIv2/firebase-functions
cp .env.example .env
# Then fill in real values from 1Password:
# ANTHROPIC_API_KEY=...
# OPENAI_API_KEY=...
# OPENROUTER_API_KEY=...
# STRIPE_SECRET_KEY=...  (test or live depending on firebase use alias)
# STRIPE_WEBHOOK_SECRET=...
```

### Environment Aliases
```bash
firebase use staging    # loads .env + .env.staging (Stripe test keys)
firebase use production # loads .env + .env.production (Stripe live keys)
```

---

## After Any Deploy

1. Check function logs for errors:
   ```bash
   firebase functions:log --project shoppingai-b9b20 --only functionName
   ```

2. Test the function directly via the app or:
   ```bash
   firebase functions:log --project shoppingai-b9b20 --only refreshItemPrice
   ```

---

## Common Pitfalls

1. **Token expired** — If EC2 deploy fails with auth error, Warren needs to regenerate:
   ```bash
   firebase login:ci
   # Copy token to EC2: ~/.hermes/vault/shoppingAIv2/firebase.token
   ```

2. **Wrong project** — Confirm active project:
   ```bash
   firebase use  # shows current project
   ```

3. **Node version mismatch** — Functions use Node.js 20. EC2 should have this; Mac may need:
   ```bash
   nvm use 20
   ```

4. **functions.config() deprecation** — If deploy fails with config errors, `.env` migration may be incomplete. Verify `.env` exists and has all required keys.

## Deploy Checklist

- [ ] **Confirm environment: STAGING or PRODUCTION** (ask user if not specified)
- [ ] Pull latest from `origin/main`
- [ ] Verify correct `.env` + `.env.{environment}` has all required keys
- [ ] Run deploy command for target environment
- [ ] Verify `✔ functions[...] Successful update operation` in output
- [ ] Check function logs for runtime errors
- [ ] Report deploy status to Warren
