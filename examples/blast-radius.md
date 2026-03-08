# Blast Radius Analysis

**Generated:** 2026-03-08T14:22:31Z
**Session:** prr-20260308-a3f7
**Target:** feature/add-payment-retry-column

---

## Changed Files (L0 — Direct)

| File | Change Type | Lines |
|------|------------|-------|
| `migrations/20260308_add_retry_count.sql` | Schema Migration | +12 |
| `internal/payments/model.go` | Struct Field Added | +8, -2 |
| `internal/payments/service.go` | Business Logic | +47, -11 |
| `internal/payments/repository.go` | Query Modified | +15, -8 |
| `api/handlers/payment.go` | Response Shape | +6, -3 |
| `api/openapi/payments.yaml` | API Spec | +9, -1 |

## L1 Dependencies (Direct Callers/Callees)

| File | Relationship | Risk |
|------|-------------|------|
| `internal/payments/retry_worker.go` | Calls `service.ProcessRetry()` | HIGH — retry logic changes affect worker behavior |
| `internal/billing/invoice.go` | Calls `repository.GetPayment()` | MEDIUM — query result shape changes |
| `internal/webhooks/stripe.go` | Calls `service.HandleWebhook()` | HIGH — webhook handler references retry count |
| `cmd/worker/main.go` | Starts retry worker | LOW — no interface change |
| `api/middleware/auth.go` | Called by handler | LOW — no interface change |

## L2 Dependencies (Transitive)

| File | Path | Risk |
|------|------|------|
| `internal/notifications/email.go` | via `billing/invoice.go` → `payments/service.go` | LOW — notification triggered by invoice, not payment model |
| `internal/reporting/daily_summary.go` | via `repository.GetPayments()` | MEDIUM — daily report queries payments table directly |
| `tests/integration/payment_flow_test.go` | Tests full payment flow | HIGH — will break if not updated |

## Data Surface

| Store | Tables/Keys Affected | Access Pattern |
|-------|---------------------|----------------|
| PostgreSQL `payments_db` | `payments` table — new column `retry_count INTEGER DEFAULT 0` | Read/Write — every payment creation and retry |
| Redis `payment-cache` | Key pattern `payment:{id}` | Read — cached payment structs will be stale until TTL |

## External Surface

| Service | Integration Point | Impact |
|---------|-------------------|--------|
| Stripe API | Webhook handler reads retry count to decide re-attempt | Logic change — not an API change |
| Client mobile app | `GET /api/v1/payments/:id` response includes new field | Additive — non-breaking if clients ignore unknown fields |
| Internal dashboard | Reads from `reporting/daily_summary` | Will show retry_count=0 for historical records |

## Summary

**Blast Radius Tier: T2 (Service-wide)**
- 6 files changed directly
- 8 L1 dependencies, 3 at HIGH risk
- 3 L2 dependencies
- 1 database migration (schema change)
- 1 cache invalidation concern
- 1 external integration affected (Stripe webhook logic)
