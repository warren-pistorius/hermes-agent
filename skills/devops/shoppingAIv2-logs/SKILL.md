---
name: shoppingAIv2-logs
description: Use when checking Firebase Cloud Functions logs for ShoppingAIv2. Retrieves and filters function execution logs to diagnose errors or verify behavior.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [firebase, logs, shoppingAIv2, debugging]
    related_skills: [shoppingAIv2-deploy]
---

# ShoppingAIv2 Logs

## Overview

Retrieves and filters Firebase Cloud Functions logs for ShoppingAIv2. Use after deploys to verify success, or when Warren reports an error to diagnose the issue.

## When to Use

- User says "check logs", "any errors", "what happened"
- After a deploy to verify functions are running
- When Warren reports a bug or app error
- Periodic health check on Cloud Functions

## EC2 Command (this agent)

```bash
cd ~/shoppingAIv2/firebase-functions && \
FIREBASE_TOKEN="$(cat ~/.hermes/vault/shoppingAIv2/firebase.token)" \
firebase functions:log --project shoppingai-b9b20
```

### Filter by Function
```bash
# Single function
firebase functions:log --project shoppingai-b9b20 --only refreshItemPrice

# Multiple functions
firebase functions:log --project shoppingai-b9b20 --only analyzeGroceryItem,analyzeGroceryItemCredits
```

### Filter by Severity
```bash
# Only errors
firebase functions:log --project shoppingai-b9b20 --only refreshItemPrice --level error

# Only warnings
firebase functions:log --project shoppingai-b9b20 --only analyzeGroceryItem --level warn
```

### Recent Logs Only
```bash
# Last 50 lines (default)
firebase functions:log --project shoppingai-b9b20 | tail -50

# Last N entries
firebase functions:log --project shoppingai-b9b20 | head -100
```

## Common Function Names

| Function | Purpose |
|---|---|
| `analyzeGroceryItem` | AI price analysis |
| `analyzeGroceryItemCredits` | AI price analysis with credit deduction |
| `refreshItemPrice` | Web search price refresh |
| `stripeWebhook` | Payment webhook handler |
| `createPaymentIntent` | Stripe payment intent |
| `processAIQueue` | Async AI queue processor |

## Reading Log Output

Log format:
```
2026-05-16T12:22:08.934793Z ? refreshItemPrice: Error message here
2026-05-16T12:22:08.934797Z ? refreshItemPrice:   at function_name (file.js:145)
```

Key fields:
- `??` = info, `W?` = warning, `??` = error
- Timestamp is UTC
- Stack traces follow the error line

## Common Errors

| Error Pattern | Likely Cause | Fix |
|---|---|---|
| `404` from Anthropic/OpenAI | API key wrong or model deprecated | Check API key in `.env`, verify model ID |
| `401 Unauthorized` | Firebase token expired | Regenerate token on Mac |
| `Exceeded limit` | AI quota exceeded | Check `quotaUsage` in RTDB |
| `TypeError: undefined` | Missing parameter | Check function input data |
| `ETIMEDOUT` | Network timeout | Retry, check API status |

## Diagnostics Workflow

1. **Get recent logs for the relevant function:**
   ```bash
   firebase functions:log --project shoppingai-b9b20 --only FUNCTION_NAME | tail -50
   ```

2. **Filter for errors only:**
   ```bash
   firebase functions:log --project shoppingai-b9b20 | grep -i error | tail -20
   ```

3. **Check a specific time window** (last hour):
   ```bash
   firebase functions:log --project shoppingai-b9b20 --only FUNCTION_NAME | grep "$(date -u -v-1H '+%Y-%m-%d')"
   ```

## Mac Equivalent

Warren can run the same commands on his Mac from the `~/shoppingAIv2/firebase-functions/` directory without the `FIREBASE_TOKEN` prefix (his CLI is authenticated interactively).

## Verification Checklist

- [ ] Identify the correct function name
- [ ] Retrieve relevant log entries
- [ ] Filter for errors if investigating a bug
- [ ] Note timestamp, error type, and stack trace
- [ ] Report findings to Warren with severity assessment
