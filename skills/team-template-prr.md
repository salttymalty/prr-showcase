# Production Readiness Review (PRR) Team Template

> 3-gate parallel analysis: know what changed, plan for failure, prove it works.

## Team Composition

### Gate 1 — KNOW (parallel, read-only)

| Role | Agent | Model | Purpose |
|------|-------|-------|---------|
| Blast Radius Mapper | `prr-blast-mapper` | haiku | Trace all affected systems, services, and data stores |
| Change Classifier | `prr-change-classifier` | haiku | Classify change types (schema/config/endpoint/flag) and assign risk tier |
| SLA Scanner | `prr-sla-scanner` | haiku | Map SLA/SLO exposure for changed code paths |

### Gate 2 — PLAN (parallel, blocked by Gate 1)

| Role | Agent | Model | Purpose |
|------|-------|-------|---------|
| Rollback Strategist | `prr-rollback-strategist` | sonnet | Write executable rollback procedures for every change type |
| Observability Auditor | `prr-observability-auditor` | sonnet | Audit logging, metrics, and alerting coverage on changed paths |
| Failure Analyst | `prr-failure-analyst` | sonnet | FMEA — enumerate failure modes with Risk Priority Numbers |

### Gate 3 — PROVE (parallel then sequential, blocked by Gate 2)

| Role | Agent | Model | Purpose |
|------|-------|-------|---------|
| Runbook Generator | `prr-runbook-generator` | sonnet | Synthesize all findings into operator-facing runbook |
| Chaos Writer | `prr-chaos-writer` | sonnet | Write testable chaos scenarios for critical FMEA items |
| Validator | `validator` | sonnet/opus | QA gate — binary PASS/FAIL verdict with veto authority |

## Total: 9 agents (3 + 3 + 3), deployed in 3 waves

## Information Flow

```
              ┌──────────────────────────────────────────────────────────────────┐
              │                        GATE 1: KNOW                              │
              │                                                                  │
              │  ┌─────────────┐  ┌──────────────┐  ┌────────────┐              │
  diff.patch ─┤  │ Blast Radius│  │   Change     │  │    SLA     │              │
  files.txt  ─┤  │   Mapper    │  │  Classifier  │  │  Scanner   │              │
              │  └──────┬──────┘  └──────┬───────┘  └─────┬──────┘              │
              └─────────┼────────────────┼────────────────┼──────────────────────┘
                        │                │                │
                        ▼                ▼                ▼
              blast-radius.md   classification.json   sla-exposure.md
                        │                │                │
              ┌─────────┼────────────────┼────────────────┼──────────────────────┐
              │         │         GATE 2: PLAN            │                      │
              │         ▼                ▼                ▼                      │
              │  ┌─────────────┐  ┌──────────────┐  ┌────────────┐              │
              │  │  Rollback   │  │ Observability│  │  Failure   │              │
              │  │ Strategist  │  │   Auditor    │  │  Analyst   │              │
              │  └──────┬──────┘  └──────┬───────┘  └─────┬──────┘              │
              └─────────┼────────────────┼────────────────┼──────────────────────┘
                        │                │                │
                        ▼                ▼                ▼
              rollback-plan.md   obs-gaps.md        fmea-matrix.md
                        │                │                │
              ┌─────────┼────────────────┼────────────────┼──────────────────────┐
              │         │         GATE 3: PROVE           │                      │
              │         ▼                                 ▼                      │
              │  ┌─────────────┐                   ┌────────────┐               │
              │  │   Runbook   │◄──── all ────────►│   Chaos    │               │
              │  │  Generator  │    outputs        │   Writer   │               │
              │  └──────┬──────┘                   └─────┬──────┘               │
              │         │                                │                      │
              │         ▼                                ▼                      │
              │    runbook.md                    chaos-scenarios.md             │
              │         │                                │                      │
              │         └────────────┬───────────────────┘                      │
              │                      ▼                                          │
              │               ┌────────────┐                                    │
              │               │  Validator  │  ← VETO AUTHORITY                 │
              │               └──────┬─────┘                                    │
              │                      ▼                                          │
              │              prr-verdict.json                                   │
              │              PASS or FAIL                                        │
              └─────────────────────────────────────────────────────────────────┘
```

## Gate Dependencies

| Task | Depends On | Rationale |
|------|-----------|-----------|
| Rollback Strategist | Blast Mapper + Classifier | Can't plan rollback without knowing what changed and how |
| Observability Auditor | Blast Mapper | Need to know which paths to audit |
| Failure Analyst | Blast Mapper + SLA Scanner | Need affected components + SLA context for RPN scoring |
| Runbook Generator | Rollback + Observability + FMEA | Synthesizes all Gate 2 outputs |
| Chaos Writer | FMEA | Scenarios are driven by failure mode analysis |
| Validator | Runbook + Chaos | Reviews all final artifacts for completeness and quality |

## The Separation Principle in Action

```
Gate 1: READ-ONLY              Gate 2: MIXED                  Gate 3: MIXED + VETO
(discover, can't modify)       (plan + audit)                 (synthesize + judge)

├─ blast-mapper (read)         ├─ rollback-strategist (write) ├─ runbook-generator (write)
├─ change-classifier (read)    ├─ observability-auditor (read)├─ chaos-writer (write)
├─ sla-scanner (read)          ├─ failure-analyst (read)      └─ validator (read + VETO)
```

- Gate 1 agents are fully read-only — injection-safe
- Gate 2 has one writer (rollback) and two readers (observability, FMEA)
- Gate 3 has two writers and one reader with veto authority
- The validator cannot fix what it finds — maintaining the separation principle

## Scaling Rules

| Change Size | Mode | Agents | Notes |
|-------------|------|--------|-------|
| < 5 files, simple | Light | 3 | Classifier + Rollback + Validator only |
| 5-20 files, single service | Standard | 9 | Full 3-gate pipeline |
| 20+ files, cross-service | Full | 9 | Same agents, upgraded models (opus for rollback + validator) |

## When to Use

- Before deploying a significant change to production
- When a PR touches infrastructure, schemas, or API contracts
- When the change affects SLA-covered code paths
- When you want to sleep well after a deploy
- "Run a PRR on this branch"
- "Is this safe to deploy?"

## When NOT to Use

- Documentation-only changes
- Test-only changes
- Changes that don't affect production behavior
- Changes behind a feature flag that won't be enabled yet
