# ShoppingAIv2 IAM Configuration

## Service Accounts

| SA | Email | Role |
|----|-------|------|
| Deploy SA (key we hold) | `firebase-adminsdk-fbsvc@shoppingai-b9b20.iam.gserviceaccount.com` | `cloudfunctions.admin`, `firebase.sdkAdminServiceAgent`, `firebaseauth.admin`, `firebasedatabase.admin`, `iam.serviceAccountTokenCreator` |
| Appspot default SA | `shoppingai-b9b20@appspot.gserviceaccount.com` | Default App Engine SA |

## The IAM Error

```
Missing permissions required for functions deploy.
You must have permission iam.serviceAccounts.ActAs on service account
shoppingai-b9b20@appspot.gserviceaccount.com.
```

**Root cause**: The deploy SA has `serviceAccountTokenCreator` (can impersonate other SAs) but lacks `serviceAccountUser` on the appspot SA.

**Fix**: Grant `Service Account User` (`roles/iam.serviceAccountUser`) on `shoppingai-b9b20@appspot.gserviceaccount.com` to the deploy SA's email (`firebase-adminsdk-fbsvc@shoppingai-b9b20.iam.gserviceaccount.com`).

## How to Apply the Fix

1. Open [console.cloud.google.com/iam-admin/iam?project=shoppingai-b9b20](https://console.cloud.google.com/iam-admin/iam?project=shoppingai-b9b20)
2. Click the appspot SA: `shoppingai-b9b20@appspot.gserviceaccount.com`
3. Click "Add principal" or "Edit" (depending on UI)
4. In "New principals", enter: `firebase-adminsdk-fbsvc@shoppingai-b9b20.iam.gserviceaccount.com`
5. In "Role", select: **Service Account** → **Service Account User**
6. Save

## Verify After Fix

```bash
# From EC2 with the SA key active:
FIREBASE_TOKEN=$(cat ~/.hermes/vault/shoppingAIv2/firebase.token) \
  firebase deploy --only functions:refreshItemPrice --project shoppingai-b9b20
```

## Additional Setup: Cloud Billing API

If deploy fails with:
```
Cloud Billing API has not been used in project 72313916021 before or it is disabled.
Enable it by visiting https://console.developers.google.com/apis/api/cloudbilling.googleapis.com/overview?project=72313916021
```

This is a separate requirement from IAM. Enable it here:
→ [console.developers.google.com/apis/api/cloudbilling.googleapis.com/overview?project=shoppingai-b9b20](https://console.developers.google.com/apis/api/cloudbilling.googleapis.com/overview?project=shoppingai-b9b20)

Click **Enable**. No billing commitment required — just activates the API. Deploy will work immediately after.

## Key Files on EC2

```
~/.hermes/vault/shoppingAIv2/
├── firebase-sa.json          # Deploy SA key (client_email: firebase-adminsdk-fbsvc@...)
├── firebase.token           # CI token (active deploy method)
├── github-token             # GitHub PAT for pushing to Warren's fork
├── firebase-functions.env    # Backup of .env
└── firebase-functions.env.staging  # Backup of .env.staging
```

## Deploy Command

```bash
cd ~/shoppingAIv2
FIREBASE_TOKEN=$(cat ~/.hermes/vault/shoppingAIv2/firebase.token) \
  firebase deploy --only functions:refreshItemPrice \
  --project shoppingai-b9b20
```

Note: `GOOGLE_APPLICATION_CREDENTIALS` + gcloud SA does NOT work for firebase deploy CLI. FIREBASE_TOKEN is the required method.
