# Rollback Plan

**Generated:** 2026-03-08T14:32:07Z
**Session:** prr-20260308-a3f7
**Target:** feature/add-payment-retry-column

---

## Rollback Order (Critical)

This change has **entangled rollback** — the code depends on the database column.
Rolling back in the wrong order causes a hard failure.

```
CORRECT ORDER:
  1. Rollback code (deploy previous version)
  2. Rollback migration (drop column)

WRONG ORDER:
  1. Rollback migration (drop column)
  2. Code still references retry_count → SELECT fails → 500 errors
```

---

## Rollback Procedure: Code

**When to trigger:** Error rate > 1% on payment endpoints within 30 minutes of deploy.

### Step 1: Deploy previous version

```bash
# Get the last known good commit
git log --oneline -5 main
# Example: abc1234 was the previous deploy

# Deploy it
kubectl set image deployment/payment-service \
  payment-service=registry.example.com/payment-service:abc1234

# Or if using Helm:
helm rollback payment-service 1
```

### Step 2: Verify rollback

```bash
# Check pods are running
kubectl get pods -l app=payment-service

# Check health endpoint
curl -s https://api.example.com/health | jq .

# Check error rate (should drop within 2 minutes)
# Datadog/Grafana: payment.request.error_rate
```

**Expected result:** Error rate returns to baseline within 5 minutes.
**If unexpected:** Pods crash-looping → check logs with `kubectl logs -l app=payment-service --tail=50`

### Step 3: Verify old code doesn't reference retry_count

```bash
# Old code uses sqlc-generated queries that don't SELECT retry_count
# The column existing in DB but not in code is SAFE (Go ignores extra columns)
# No action needed — old code is compatible with new schema
```

**Point of no return:** None for code rollback. Can roll back at any time.

---

## Rollback Procedure: Migration

**When to trigger:** Only if the code has been rolled back AND you want to fully revert.

**Important:** The migration rollback is optional. The old code works fine with the column present (it just ignores it). Only drop the column if you're abandoning the feature entirely.

### Step 1: Verify code is rolled back first

```bash
# Confirm the running version does NOT reference retry_count
kubectl exec -it deployment/payment-service -- \
  grep -r "retry_count" /app/
# Should return empty
```

### Step 2: Drop the column

```sql
-- Run against payments_db
ALTER TABLE payments DROP COLUMN IF EXISTS retry_count;
```

### Step 3: Verify

```sql
-- Confirm column is gone
SELECT column_name FROM information_schema.columns
WHERE table_name = 'payments' AND column_name = 'retry_count';
-- Should return 0 rows
```

**Point of no return:** Dropping the column deletes all retry_count data. If the feature is re-deployed later, all payments start at retry_count=0.

---

## Rollback Procedure: Cache

**When to trigger:** If stale cache entries cause incorrect behavior after code rollback.

```bash
# Flush payment cache keys
redis-cli -h redis.example.com KEYS "payment:*" | xargs redis-cli DEL

# Or if you have too many keys, use SCAN:
redis-cli -h redis.example.com --scan --pattern "payment:*" | \
  xargs -L 100 redis-cli DEL
```

**Verification:**
```bash
# Confirm cache is empty for payments
redis-cli -h redis.example.com DBSIZE
# Compare to expected size without payment keys
```

---

## Blast Radius of Rollback Itself

| Action | Side Effect | Severity |
|--------|------------|----------|
| Code rollback | Payments mid-retry will restart count at 0 on re-deploy | LOW |
| Column drop | Historical retry data lost permanently | MEDIUM |
| Cache flush | Temporary latency spike as cache rebuilds | LOW |

## Decision Matrix

| Situation | Action |
|-----------|--------|
| Error rate spikes immediately after deploy | Roll back code only. Keep column. |
| Retry cap causes too many permanent failures | Increase cap (env var) or roll back code. Keep column. |
| Migration caused table lock / outage | Emergency: roll back code + drop column during maintenance window. |
| Everything works but you want to revert | Roll back code. Column is harmless — drop later or keep for next attempt. |
