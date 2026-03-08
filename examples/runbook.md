# Production Runbook: Payment Retry Count

**Generated:** 2026-03-08T14:41:22Z
**Change:** feature/add-payment-retry-column
**Risk Tier:** T2 (Service-wide)
**Rollback Complexity:** Medium (ordered rollback required)

---

## TL;DR

**What changed:** Added retry_count tracking to payments. Payments now fail permanently after 3 retries instead of retrying indefinitely. New column in payments table, new field in API response.

**Blast radius:** T2 — payments service, retry worker, Stripe webhook handler, Redis cache, daily reporting.

**Critical risks:** (1) Migration locks table if PostgreSQL < 11. (2) Redis cache not invalidated after retry — stale data served for up to 5 minutes.

**Rollback:** Deploy previous code version first, then optionally drop column. Order matters.

---

## Pre-Deploy Checklist

- [ ] Verify PostgreSQL version is 11+ (FM-1 mitigation)
  ```sql
  SELECT version();
  -- Must show PostgreSQL 11.x or higher
  ```
- [ ] Verify Redis cache TTL for payment keys
  ```bash
  redis-cli -h redis.example.com TTL "payment:any-known-id"
  ```
- [ ] Open monitoring dashboard: payment error rate, worker latency, DB connections
- [ ] Confirm integration tests pass on this branch
- [ ] Notify on-call that a payment service deploy is starting

---

## Deploy Steps

### Step 1: Run migration

```bash
# Apply migration
psql payments_db < migrations/20260308_add_retry_count.sql
```

**Expected result:** `ALTER TABLE` completes in < 1 second (online operation on PG11+).
**If unexpected:** Migration hangs for > 5 seconds → `Ctrl+C` and investigate. Check `pg_stat_activity` for locks.

### Step 2: Deploy new code

```bash
kubectl set image deployment/payment-service \
  payment-service=registry.example.com/payment-service:feature-add-payment-retry-column-latest
```

**Expected result:** Rolling update completes. New pods pass health checks.
**If unexpected:** Pods crash-looping → check logs, initiate code rollback (see Rollback Procedure).

### Step 3: Verify retry worker starts

```bash
kubectl logs -l app=payment-service --tail=20 | grep "retry_worker"
```

**Expected result:** Log line `retry_worker: started with retry_cap=3`.
**If unexpected:** No log line → worker may not have initialized. Check for panic in logs.

---

## Post-Deploy Verification

### Immediate (0-5 minutes)
- [ ] Health check returns 200:
  ```bash
  curl -s https://api.example.com/health | jq .status
  # Expected: "ok"
  ```
- [ ] Error rate stable (< 0.5%):
  Check dashboard → payment.request.error_rate
- [ ] DB connection pool not saturated:
  ```sql
  SELECT count(*) FROM pg_stat_activity WHERE datname = 'payments_db';
  -- Should be < 15 (pool max is 20)
  ```

### Short-term (5-30 minutes)
- [ ] Payment success rate > 99.5%:
  Check dashboard → payment.success_rate
- [ ] Retry worker cycle time < 30s:
  Check dashboard → retry_worker.cycle_duration_seconds
- [ ] No increase in `payment_final_status=failed`:
  ```
  Query: sum(payment_final_status{status="failed"}) by (5m)
  Baseline: ~2-5 per 5min window
  Alert if: > 15 per 5min window
  ```

### Extended (30 min - 2 hours)
- [ ] Daily summary report runs without error (if deploy is before daily report time)
- [ ] Stripe webhook processing latency unchanged
- [ ] Redis cache hit ratio stable

---

## Rollback Triggers

Initiate rollback if ANY of:
- [ ] Payment error rate > 1% for 5 consecutive minutes
- [ ] Payment success rate drops below 99%
- [ ] Retry worker cycle time exceeds 60s (2x normal)
- [ ] DB connection pool hits 20/20 (saturated)
- [ ] Manual decision by on-call

---

## Rollback Procedure

**CRITICAL: Roll back code BEFORE migration. Wrong order causes 500 errors.**

### Step 1: Roll back code
```bash
kubectl rollout undo deployment/payment-service
```
**Verify:** `kubectl rollout status deployment/payment-service`
Expected: `deployment "payment-service" successfully rolled out`

### Step 2: Verify error rate drops
Wait 2 minutes. Check dashboard → payment.request.error_rate
Expected: Returns to baseline (< 0.1%)

### Step 3: (Optional) Roll back migration
Only if abandoning the feature. The column is harmless to old code.
```sql
ALTER TABLE payments DROP COLUMN IF EXISTS retry_count;
```

### Post-Rollback Verification
- [ ] Health check returns 200
- [ ] Error rate at baseline
- [ ] Payment success rate > 99.5%
- [ ] Retry worker running (old behavior: unlimited retries)

---

## Escalation

| Condition | Contact | Method |
|-----------|---------|--------|
| Rollback fails (pods won't start) | Platform team | #platform-oncall Slack |
| Data integrity concern (retry counts wrong) | Database team | #data-oncall Slack |
| Payment success rate < 98% (SLA breach) | Engineering manager + VP Eng | PagerDuty escalation |
| Stripe webhook failures | Payments team lead | Direct call |

---

## Context (for later reading)

### Why this change
Payments could retry indefinitely, causing duplicate charges and Stripe rate limiting. The retry cap limits attempts to 3, with explicit failure after exhaustion. This protects both customers (no duplicate charges) and infrastructure (no infinite loops).

### Key failure modes (from FMEA)
1. **FM-1 (RPN 216):** Migration table lock on PG < 11 — mitigated by pre-deploy version check
2. **FM-3 (RPN 288):** Stale Redis cache after retry — requires cache invalidation (not yet implemented, flagged as deploy blocker)
3. **FM-2 (RPN 168):** N+1 query in retry worker — recommend batch query before next release

### Observability gaps
- No alert threshold on retry worker cycle duration
- No cache invalidation monitoring
- Daily report latency not tracked

### SLA exposure
- Payment processing latency SLO (p99 < 500ms) — LOW risk
- Payment success rate SLO (> 99.5%) — MEDIUM risk for 48 hours post-deploy
- API availability SLO (99.9%) — LOW risk if PG 11+
