# Chaos Test Scenarios

**Generated:** 2026-03-08T14:38:44Z
**Source:** FMEA matrix from PRR Gate 2
**Total Scenarios:** 4
**Critical Coverage:** 2/2 critical FMEA items have scenarios

---

## Scenario 1: Table Lock During Migration (FM-1, RPN 216)

**Hypothesis:** If the migration is run against PostgreSQL < 11, the ALTER TABLE should fail fast (with lock_timeout) rather than blocking all payment operations for the duration of the rewrite.

### Setup
```bash
# Use a PG10 test instance (docker)
docker run -d --name pg10-test -e POSTGRES_DB=payments_db \
  -p 5433:5432 postgres:10

# Create the payments table with test data
psql -h localhost -p 5433 -U postgres payments_db <<SQL
CREATE TABLE payments (
  id UUID PRIMARY KEY,
  amount INTEGER,
  status TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
INSERT INTO payments SELECT gen_random_uuid(), 1000, 'completed', NOW()
FROM generate_series(1, 100000);
SQL
```

### Trigger
```bash
# Open a long-running transaction that holds a lock
psql -h localhost -p 5433 -U postgres payments_db -c "BEGIN; SELECT * FROM payments LIMIT 1;" &

# In parallel, run the migration with lock_timeout
psql -h localhost -p 5433 -U postgres payments_db <<SQL
SET lock_timeout = '5s';
ALTER TABLE payments ADD COLUMN retry_count INTEGER DEFAULT 0;
SQL
```

### Observe
- **Expected behavior:** Migration fails with `ERROR: canceling statement due to lock timeout`
- **Expected timing:** Fails within 5 seconds
- **Expected log:** PostgreSQL error log shows lock timeout

### Pass Criteria
- [ ] Migration fails with lock_timeout error (not a hang)
- [ ] Existing payment queries continue to work during and after the failed migration
- [ ] No data corruption in payments table

### Fail Criteria
- [ ] Migration hangs indefinitely (no lock_timeout protection)
- [ ] Existing transactions are killed by the migration
- [ ] Payments table is left in an inconsistent state

### Cleanup
```bash
docker stop pg10-test && docker rm pg10-test
```

---

## Scenario 2: Stale Cache After Retry Count Update (FM-3, RPN 288)

**Hypothesis:** After a payment's retry_count is incremented, the cached version in Redis should be invalidated (or updated) so that subsequent reads return the correct count.

### Setup
```bash
# Seed a test payment
PAYMENT_ID="test-payment-$(date +%s)"
curl -s -X POST https://staging-api.example.com/api/v1/payments \
  -H "Content-Type: application/json" \
  -d "{\"id\": \"$PAYMENT_ID\", \"amount\": 1000}"

# Verify it's cached
redis-cli -h redis-staging GET "payment:$PAYMENT_ID" | jq .retry_count
# Expected: 0
```

### Trigger
```bash
# Trigger a retry (simulating a failed payment attempt)
curl -s -X POST "https://staging-api.example.com/api/v1/payments/$PAYMENT_ID/retry"

# Immediately read from the API (which reads from cache)
curl -s "https://staging-api.example.com/api/v1/payments/$PAYMENT_ID" | jq .retry_count
```

### Observe
- **Expected (if fix implemented):** API returns `retry_count: 1`
- **Expected (if bug present):** API returns `retry_count: 0` (stale cache)
- **Check Redis directly:**
  ```bash
  redis-cli -h redis-staging GET "payment:$PAYMENT_ID" | jq .retry_count
  ```
- **Check database directly:**
  ```sql
  SELECT retry_count FROM payments WHERE id = 'test-payment-xxx';
  ```

### Pass Criteria
- [ ] API response shows retry_count = 1 within 1 second of retry
- [ ] Redis value matches database value
- [ ] No 5-minute window of stale data

### Fail Criteria
- [ ] API returns retry_count = 0 after retry (cache not invalidated)
- [ ] Redis and database values disagree for > 5 seconds

### Cleanup
```bash
# Delete test payment
psql payments_db -c "DELETE FROM payments WHERE id = '$PAYMENT_ID'"
redis-cli -h redis-staging DEL "payment:$PAYMENT_ID"
```

---

## Scenario 3: Retry Worker Under Load (FM-2, RPN 168)

**Hypothesis:** If the retry worker batch contains 100+ payments, the N+1 query pattern should not cause the worker cycle to exceed its 30-second interval.

### Setup
```bash
# Seed 200 payments in "pending_retry" status
for i in $(seq 1 200); do
  psql payments_db -c "INSERT INTO payments (id, amount, status, retry_count)
    VALUES ('load-test-$i', 1000, 'pending_retry', 1);"
done
```

### Trigger
```bash
# Force a retry worker cycle
curl -s -X POST https://staging-api.example.com/admin/retry-worker/trigger

# Monitor cycle duration
kubectl logs -l app=payment-service --tail=5 -f | grep "cycle_duration"
```

### Observe
- **Expected metric:** `retry_worker.cycle_duration_seconds` should be < 30s
- **Expected DB connections:** Connection pool usage should not exceed 80% (16/20)
- **Expected behavior:** All 200 payments processed in a single cycle

### Pass Criteria
- [ ] Worker cycle completes in < 30 seconds with 200 items
- [ ] DB connection pool stays below 80% during cycle
- [ ] No query timeouts in PostgreSQL logs

### Fail Criteria
- [ ] Cycle duration exceeds 30 seconds (next tick fires before current completes)
- [ ] Connection pool saturates (20/20)
- [ ] Query timeouts appear in logs

### Cleanup
```bash
psql payments_db -c "DELETE FROM payments WHERE id LIKE 'load-test-%'"
```

---

## Scenario 4: Retry Cap Exhaustion (FM-4, RPN 96)

**Hypothesis:** When a payment reaches retry_count = 3, it should be marked as permanently failed with a clear error message, not silently dropped.

### Setup
```bash
# Create a payment that will always fail (invalid card token)
PAYMENT_ID="cap-test-$(date +%s)"
psql payments_db -c "INSERT INTO payments (id, amount, status, retry_count, card_token)
  VALUES ('$PAYMENT_ID', 1000, 'pending_retry', 2, 'tok_invalid');"
```

### Trigger
```bash
# Trigger retry worker — this payment has retry_count=2, so this is attempt 3 (the last)
curl -s -X POST https://staging-api.example.com/admin/retry-worker/trigger

# Check the payment status
sleep 5
curl -s "https://staging-api.example.com/api/v1/payments/$PAYMENT_ID" | jq '{status, retry_count, failure_reason}'
```

### Observe
- **Expected status:** `failed`
- **Expected retry_count:** `3`
- **Expected failure_reason:** Non-empty string explaining why
- **Expected metric:** `payment_final_status{status="failed"}` incremented by 1
- **Expected log:** `payment permanently failed after 3 retries: $PAYMENT_ID`

### Pass Criteria
- [ ] Payment status is "failed" (not "pending_retry")
- [ ] failure_reason field is populated (not null/empty)
- [ ] Metric counter incremented
- [ ] Log line present with payment ID
- [ ] No further retry attempts for this payment

### Fail Criteria
- [ ] Payment stuck in "pending_retry" after 3 retries
- [ ] failure_reason is empty (operator can't diagnose)
- [ ] Payment retried a 4th time despite cap

### Cleanup
```bash
psql payments_db -c "DELETE FROM payments WHERE id = '$PAYMENT_ID'"
```

---

## Test Environment Requirements
- Staging environment with payment service deployed
- PostgreSQL 10 Docker image (for Scenario 1 only)
- Redis access (staging)
- `kubectl` access to staging cluster
- Admin endpoint access for retry worker trigger

## Notes for Validator
- Scenario 1 requires a PG10 Docker instance — cannot run in production or standard staging
- Scenario 2 may pass even without the fix if cache TTL happens to expire during the test window; run 3 times in rapid succession to confirm
- Scenario 3 load numbers (200) are conservative; production batches may be larger
