---
name: recoup-health-check
description: Diagnostic health check for Recoup agent setups. Tests API connectivity, auth, credit balance, plugin status, and common misconfigurations. Use when something isn't working, when checking quota, or for periodic status checks. Triggers on "check Recoup status", "is Recoup working", "health check", "check my credits", "what's wrong with Recoup", or any troubleshooting request.
---

# Recoup Health Check

Diagnostic skill for troubleshooting and status monitoring. Runs connectivity
tests, validates auth, checks credit balance, and reports plugin status.

```bash
export RECOUP_API_KEY="recoup_sk_..."
export RECOUP_API="https://api.recoupable.com/api"
```

## Decision tree

- **"Is Recoup working?"** → run full diagnostic below
- **"Check my credits"** → jump to Step 3 (credits)
- **"What plugins do I have?"** → jump to Step 4 (plugin inventory)
- **Something specific is broken** → run full diagnostic, focus on the failing step

## Full diagnostic

Run all checks in order. Stop and report at the first failure.

### Step 1 — API connectivity

```bash
# Basic connectivity test (no auth required)
curl -s -o /dev/null -w "%{http_code}" https://api.recoupable.com/api/health
```

- `200` → API is reachable ✅
- Timeout or connection error → network issue (check internet, DNS, firewall)
- `503` → API is down (temporary, retry in a few minutes)

### Step 2 — Authentication

```bash
# Auth test — requires valid API key
curl -s -w "\n%{http_code}" "$RECOUP_API/accounts/me" \
  -H "x-api-key: $RECOUP_API_KEY" | tail -1
```

- `200` → authenticated ✅ (parse response for account details)
- `401` → invalid key (regenerate at developers.recoupable.com/agents)
- `403` → valid key but insufficient permissions

If auth passes, extract and report:
```bash
curl -s "$RECOUP_API/accounts/me" \
  -H "x-api-key: $RECOUP_API_KEY" | jq '{id, name, email, plan}'
```

### Step 3 — Credit balance

```bash
curl -s "$RECOUP_API/credits/balance" \
  -H "x-api-key: $RECOUP_API_KEY" | jq
```

Report: remaining credits, plan limits, and whether the account is near quota.
If credits are low, warn the user.

If the endpoint returns 404, credits may not be enabled for this account tier —
note this but don't treat it as an error.

### Step 4 — Plugin inventory

Check for installed Recoup plugins by scanning for known skill directories:

| Plugin | Check for |
| ------ | --------- |
| Platform | `skills/recoup-getting-started/` or `skills/recoup-health-check/` |
| Research | `skills/recoup-artist-research/` |
| Content | `skills/recoup-content-create/` |
| Catalog Diligence | `skills/diligence-kickoff/` |

Report which are installed and which are missing.

### Step 5 — Functional test

Run a lightweight API call to prove end-to-end functionality:

```bash
# Quick search — minimal payload, fast response
curl -s "$RECOUP_API/research?q=test&type=artists&beta=true" \
  -H "x-api-key: $RECOUP_API_KEY" | jq '.data | length'
```

If this returns a number ≥ 0, the full pipeline works.

## Output format

```
🏥 Recoup Health Check

API:     {✅ Reachable | ❌ Unreachable}
Auth:    {✅ Valid ({email}) | ❌ Failed}
Credits: {✅ {remaining}/{total} | ⚠️ Low ({remaining}) | ℹ️ N/A}
Plugins: {N installed} / 4 available

{If any check failed:}
⚠️ Issue: {description}
   Fix: {specific remediation step}

{If all checks passed:}
✅ All systems healthy
```

## Common issues

| Symptom | Likely cause | Fix |
| ------- | ------------ | --- |
| All calls return 401 | API key rotated or expired | Regenerate at developers.recoupable.com/agents |
| Research works but content fails | Content plugin not installed | Install recoup-content-plugin |
| Intermittent timeouts | Rate limiting | Add 1s delay between calls, check credit balance |
| "No results" on search | Artist name misspelled or not in database | Try broader search terms, check spelling |
| Credits at 0 | Quota exhausted | Contact support or upgrade plan |
