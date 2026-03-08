# Failure Mode and Effects Analysis (FMEA)

**Generated:** 2026-03-08T14:35:12Z
**Session:** prr-20260308-a3f7
**Target:** feature/add-payment-retry-column

---

## Scoring Reference

- **Severity (S):** 1-10. Impact on users/business if failure occurs.
- **Occurrence (O):** 1-10. Likelihood of the failure mode.
- **Detection (D):** 1-10. Likelihood that existing monitoring catches it BEFORE users are affected. (10 = undetectable)
- **RPN = S x O x D.** Higher = more risk.

| RPN Range | Action |
|-----------|--------|
| 0-49 | Accept |
| 50-124 | Monitor — add alerting if not present |
| 125-199 | Mitigate — change code or add safeguards |
| 200+ | **DEPLOY BLOCK** — must resolve before ship |

---

## Failure Modes

### FM-1: Migration locks payments table (PG < 11)

| Factor | Score | Justification |
|--------|-------|---------------|
| Severity | 9 | All payment API calls fail during lock. Revenue impact immediate. |
| Occurrence | 3 | Only if PG version is < 11. Most production deployments are 14+. But not verified. |
| Detection | 8 | No pre-deploy check for PG version. Would appear as mass timeouts after deploy starts. |

**RPN: 216 — DEPLOY BLOCK**

**Mitigation:** Add a pre-deploy check: `SELECT version()` and abort if < 11. Or wrap migration in a transaction with `SET lock_timeout = '5s'` to fail fast instead of blocking.

---

### FM-2: Retry worker N+1 query per batch item

| Factor | Score | Justification |
|--------|-------|---------------|
| Severity | 6 | Worker slows down; retries delayed but not lost. |
| Occurrence | 7 | Happens every 30s cycle. Scales with batch size. |
| Detection | 4 | Worker latency metric exists but no alert threshold set. |

**RPN: 168 — Mitigate**

**Mitigation:** Batch-read retry counts in a single `SELECT ... WHERE id IN (...)` query. This is a code change, not just monitoring.

---

### FM-3: Redis serves stale payment after retry_count update

| Factor | Score | Justification |
|--------|-------|---------------|
| Severity | 4 | Dashboard shows wrong retry count. No revenue impact. |
| Occurrence | 8 | Every retry event. Cache TTL is 5 minutes. |
| Detection | 9 | No cache invalidation monitoring. Users would report stale data. |

**RPN: 288 — DEPLOY BLOCK**

**Mitigation:** Invalidate cache key `payment:{id}` after retry_count update in `repository.go`. Add cache-invalidation log line for monitoring.

---

### FM-4: Retry cap causes legitimate payments to fail permanently

| Factor | Score | Justification |
|--------|-------|---------------|
| Severity | 8 | Customer payment fails. Revenue lost. Support ticket generated. |
| Occurrence | 4 | Affects payments needing > 3 retries. Historically ~2% of retried payments. |
| Detection | 3 | `payment_final_status` metric exists. Alert at > 1% failure rate. |

**RPN: 96 — Monitor**

**Recommendation:** Set alert on `payment_final_status=failed WHERE retry_count >= 3` for first 48 hours. Consider making cap configurable via environment variable.

---

### FM-5: Concurrent retry_count increment causes lost updates

| Factor | Score | Justification |
|--------|-------|---------------|
| Severity | 5 | Retry count underreported. Payment retried more than cap allows. |
| Occurrence | 2 | Requires two retries of same payment in same 30s window. Unlikely but possible under load. |
| Detection | 7 | No monitoring on retry_count accuracy. Would only show as payments exceeding cap. |

**RPN: 70 — Monitor**

**Note:** Code uses `SET retry_count = retry_count + 1` (atomic increment), which is safe. This failure mode is theoretical with current implementation. Would become real if someone refactors to read-then-write pattern.

---

### FM-6: Integration test suite breaks without update

| Factor | Score | Justification |
|--------|-------|---------------|
| Severity | 2 | CI fails. No production impact. Blocks other PRs. |
| Occurrence | 9 | Test references payment struct without retry_count field. Will fail on first CI run. |
| Detection | 1 | CI catches immediately. |

**RPN: 18 — Accept**

**Note:** Expected. Update `tests/integration/payment_flow_test.go` before merge.

---

## Summary Matrix

| ID | Failure Mode | S | O | D | RPN | Action |
|----|-------------|---|---|---|-----|--------|
| FM-1 | Migration table lock (PG<11) | 9 | 3 | 8 | **216** | BLOCK — verify PG version |
| FM-2 | Worker N+1 query | 6 | 7 | 4 | **168** | Mitigate — batch query |
| FM-3 | Stale Redis cache | 4 | 8 | 9 | **288** | BLOCK — add cache invalidation |
| FM-4 | Retry cap permanent failure | 8 | 4 | 3 | 96 | Monitor — alert on failure rate |
| FM-5 | Concurrent increment | 5 | 2 | 7 | 70 | Monitor — already atomic |
| FM-6 | Integration test break | 2 | 9 | 1 | 18 | Accept — update test |

**Critical items (RPN > 200): 2**
**High items (RPN 125-199): 1**
**Moderate items (RPN 50-124): 2**
**Acceptable items (RPN < 50): 1**
