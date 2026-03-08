# SLA/SLO Exposure Analysis

**Generated:** 2026-03-08T14:23:45Z
**Session:** prr-20260308-a3f7
**Target:** feature/add-payment-retry-column

---

## Affected SLAs

### Payment Processing Latency — SLO: p99 < 500ms

| Code Path | Current p99 | Risk | Detail |
|-----------|-------------|------|--------|
| `POST /api/v1/payments` | 320ms | LOW | No change to creation path |
| `POST /api/v1/payments/:id/retry` | 280ms | MEDIUM | New conditional logic adds branching; retry_count INCREMENT is an additional DB write |
| `retry_worker.ProcessBatch()` | 1200ms (batch) | HIGH | Worker now checks retry_count before each retry — adds N queries per batch where N = batch size |

**Recommendation:** The retry worker should batch-read retry counts in a single query rather than N+1 per batch item. Current implementation will degrade linearly with batch size.

### Payment Success Rate — SLO: > 99.5% of valid payments succeed

| Scenario | Impact | Detail |
|----------|--------|--------|
| Retry cap at 3 | MEDIUM | Payments that previously retried indefinitely will now fail after 3 attempts. Expected to reduce success rate by 0.1-0.3% initially. |
| Historical payments | NONE | Existing payments have retry_count=0, so they get fresh retry budget. |

**Recommendation:** Monitor `payment_final_status=failed` metric closely for 48 hours post-deploy. If failure rate increases >0.5%, consider raising the cap to 5.

### API Availability — SLO: 99.9% uptime

| Risk | Detail |
|------|--------|
| LOW | Migration is online (ADD COLUMN with DEFAULT on PG11+). No downtime expected. |
| MEDIUM | If PostgreSQL version is <11, migration locks `payments` table. During lock, all payment API calls will queue/timeout. |

**Recommendation:** Verify PostgreSQL version before deploy. If <11, schedule migration during maintenance window.

## Implicit SLAs (Detected from Code)

| Source | Implied SLA | Detail |
|--------|------------|--------|
| `service.go:47` — `context.WithTimeout(ctx, 5*time.Second)` | 5s hard timeout on retry processing | New retry logic must complete within existing timeout |
| `repository.go:23` — `pgxpool` with `MaxConns: 20` | 20 concurrent DB connections max | Migration + retry worker + API all share this pool |
| `retry_worker.go:15` — `time.NewTicker(30 * time.Second)` | Worker runs every 30s | Retry count check adds latency per cycle; if cycle >30s, ticks accumulate |

## Dark Paths (No SLA Coverage)

| Path | Why It Matters |
|------|---------------|
| Redis cache invalidation after retry | Stale payment objects served from cache after retry_count changes; no TTL-based SLA |
| Daily reporting query | `daily_summary.go` queries payments table directly; no latency SLA, but a table lock would delay the report |

## Summary

- **2 explicit SLOs** at risk (latency, success rate)
- **1 conditional risk** (PG version-dependent table lock)
- **3 implicit SLAs** detected from code (timeouts, pool size, ticker interval)
- **2 dark paths** with no monitoring coverage
