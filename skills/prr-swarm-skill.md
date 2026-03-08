---
name: prr-swarm
description: |
  Production Readiness Review swarm. Takes a branch, PR, or diff and runs a 3-gate parallel analysis: blast radius mapping, change classification, SLA scanning, rollback planning, observability auditing, failure mode analysis, runbook generation, and chaos scenario writing. Produces operator-grade artifacts, not reports. Validator holds veto authority.
version: "1.0"
---

# Production Readiness Review (PRR) Swarm

> Every change gets the review it deserves — in parallel, against the actual code, with veto gates.

---

## Instructions

When the user invokes `/prr-swarm <target>`, run the following steps.

### Step 0: Parse the Target

**Accepted inputs:**

| Input | Example | How to Get the Diff |
|-------|---------|-------------------|
| Branch name | `/prr-swarm feature/auth-refactor` | `git diff main...feature/auth-refactor` |
| PR number | `/prr-swarm #142` | `gh pr diff 142` |
| Commit range | `/prr-swarm abc1234..def5678` | `git diff abc1234..def5678` |
| "current" / no arg | `/prr-swarm` | `git diff main...HEAD` (current branch vs main) |

**First action:** Capture the diff and changed file list:

```bash
# Capture diff to a temp file for agents to read
git diff main...HEAD > /tmp/prr-diff.patch
git diff main...HEAD --name-only > /tmp/prr-changed-files.txt
git diff main...HEAD --stat > /tmp/prr-diff-stat.txt
```

If the diff is empty, stop: "No changes detected against main. Nothing to review."

### Step 1: Generate Session ID

```bash
python3 -c "import secrets,datetime as d; print(f'prr-{d.date.today():%Y%m%d}-{secrets.token_hex(2)}')"
```

Format: `prr-YYYYMMDD-XXXX` (e.g., `prr-20260308-a3f7`)

### Step 2: Quick Triage — Is a Full PRR Warranted?

Before deploying 8 agents, do a 30-second triage:

```
Read /tmp/prr-diff-stat.txt
```

| Condition | Action |
|-----------|--------|
| < 5 files changed, all in one directory, no migrations/config/infra | **Light PRR** — skip to Step 5 (light mode) |
| 5-20 files, single service, no schema changes | **Standard PRR** — full 3-gate pipeline |
| 20+ files, multiple services, schema/infra changes | **Full PRR** — all gates + extra scrutiny |
| Only docs/comments/tests changed | **Skip PRR** — "These changes don't affect production behavior. No PRR needed." |

**Tell the user the triage result:**

```
## PRR Triage: [Session ID]

**Target:** [branch/PR]
**Files changed:** [N]
**Change shape:** [light / standard / full / skip]
**Estimated agents:** [N]
**Estimated time:** [rough]

Proceeding with [mode] PRR.
```

### Step 3: Create Team

```
TeamCreate:
  team_name: "prr-[session_id]"
  description: "Production Readiness Review for [target]"
```

### Step 4: Create Tasks with Gate Dependencies

**Gate 1 — KNOW (parallel, no dependencies)**

```
Task 1: "Map blast radius of changes"
  - Agent: prr-blast-mapper (haiku)
  - Input: /tmp/prr-diff.patch, /tmp/prr-changed-files.txt
  - Output: blast-radius.md
  - Acceptance: All changed files traced to L2 dependencies

Task 2: "Classify change types and assign risk tier"
  - Agent: prr-change-classifier (haiku)
  - Input: /tmp/prr-diff.patch, /tmp/prr-changed-files.txt
  - Output: change-classification.json
  - Acceptance: Every changed file classified, risk tier assigned

Task 3: "Scan SLA/SLO exposure"
  - Agent: prr-sla-scanner (haiku)
  - Input: /tmp/prr-diff.patch, /tmp/prr-changed-files.txt
  - Output: sla-exposure.md
  - Acceptance: All endpoints traced to SLAs, gaps flagged
```

**Gate 2 — PLAN (blocked by Gate 1)**

```
Task 4: "Write rollback procedures"
  - Agent: prr-rollback-strategist (sonnet)
  - Input: blast-radius.md + change-classification.json
  - Output: rollback-plan.md + any rollback scripts
  - blockedBy: [Task 1, Task 2]
  - Acceptance: Every change type has a specific rollback procedure with verification steps

Task 5: "Audit observability coverage"
  - Agent: prr-observability-auditor (sonnet)
  - Input: blast-radius.md
  - Output: observability-gaps.md
  - blockedBy: [Task 1]
  - Acceptance: Coverage percentage calculated, dark paths flagged, recommendations specific

Task 6: "Run failure mode analysis (FMEA)"
  - Agent: prr-failure-analyst (sonnet)
  - Input: blast-radius.md + sla-exposure.md
  - Output: fmea-matrix.md
  - blockedBy: [Task 1, Task 3]
  - Acceptance: All critical paths have RPN scores, no unscored failure modes
```

**Gate 3 — PROVE (blocked by Gate 2)**

```
Task 7: "Generate operator runbook"
  - Agent: prr-runbook-generator (sonnet)
  - Input: ALL gate 1 + gate 2 outputs
  - Output: runbook.md
  - blockedBy: [Task 4, Task 5, Task 6]
  - Acceptance: Passes the 3am rule — executable by on-call who's never seen this code

Task 8: "Write chaos test scenarios"
  - Agent: prr-chaos-writer (sonnet)
  - Input: fmea-matrix.md
  - Output: chaos-scenarios.md
  - blockedBy: [Task 6]
  - Acceptance: Every critical/high FMEA item has a testable scenario with pass/fail criteria

Task 9: "QA gate — final PRR verdict"
  - Agent: validator (sonnet) — uses existing validator role with PRR acceptance criteria
  - Input: ALL outputs from all gates
  - Output: prr-verdict.json
  - blockedBy: [Task 7, Task 8]
  - Acceptance: Binary PASS/FAIL with specific findings
```

### Step 4b: PRR-Specific Validator Criteria

When spawning the validator for Task 9, include these acceptance criteria in addition to the standard validator checks:

```markdown
## PRR Acceptance Criteria

### Completeness
- [ ] Every changed file appears in blast-radius.md
- [ ] Every changed file is classified in change-classification.json
- [ ] Every change type has a rollback procedure
- [ ] Every critical FMEA item (RPN > 200) has a chaos scenario
- [ ] Runbook covers deploy, verify, rollback, and escalation

### Quality
- [ ] Rollback procedures have verification steps (not just commands)
- [ ] FMEA scores are justified (not just assigned)
- [ ] Runbook passes the 3am rule (executable without context)
- [ ] Chaos scenarios have pass/fail criteria (not just descriptions)
- [ ] No placeholder content in any output

### Consistency
- [ ] Risk tier in classification matches risk discussed in FMEA
- [ ] Rollback procedures match the change types identified
- [ ] Runbook references match actual output file contents
- [ ] SLA exposure findings are reflected in FMEA scoring

### Verdict
- **PASS:** All completeness checks pass, no blocking quality issues
- **FAIL:** Any completeness gap, or quality issues that would mislead an operator
```

### Step 5: Deploy Agents

#### Light Mode (< 5 files, simple change)

Skip team creation. Run 3 agents sequentially as single `Task` calls:

1. **Classifier** (haiku) — classify the change, assign tier
2. **Rollback Strategist** (sonnet) — write rollback procedure
3. **Validator** (sonnet) — review both outputs

Output: A lightweight `prr-light.md` combining classification + rollback + verdict.

#### Standard Mode (5-20 files)

Deploy the full 3-gate pipeline:

**Gate 1 — spawn 3 agents in parallel:**
```
Task(prr-blast-mapper, haiku, isolation: worktree)
Task(prr-change-classifier, haiku, isolation: worktree)
Task(prr-sla-scanner, haiku, isolation: worktree)
```

Wait for all Gate 1 tasks to complete.

**Commit checkpoint:** "Gate 1 complete — blast radius mapped, changes classified, SLAs scanned. Committing findings before Gate 2."

**Gate 2 — spawn 3 agents in parallel:**
```
Task(prr-rollback-strategist, sonnet, team_name: prr-{session_id})
Task(prr-observability-auditor, sonnet, isolation: worktree)
Task(prr-failure-analyst, sonnet, isolation: worktree)
```

Wait for all Gate 2 tasks to complete.

**Commit checkpoint:** "Gate 2 complete — rollback planned, observability audited, failure modes analyzed. Committing before Gate 3."

**Gate 3 — spawn 2 agents, then validator:**
```
Task(prr-runbook-generator, sonnet, team_name: prr-{session_id})
Task(prr-chaos-writer, sonnet, team_name: prr-{session_id})
```

Wait for both. Then:
```
Task(validator, sonnet, team_name: prr-{session_id})
```

#### Full Mode (20+ files, cross-service)

Same as Standard, but:
- Gate 1 agents get `maxTurns: 35` (more to trace)
- Gate 2 rollback strategist runs with `model: opus` (complex rollback needs judgment)
- Gate 3 validator runs with `model: opus` (high stakes need best judgment)

### Step 6: Collect and Present Results

After the validator returns its verdict:

```markdown
## PRR Complete: [Session ID]

**Target:** [branch/PR]
**Verdict:** PASS / FAIL
**Risk Tier:** [T1-T4]
**Agents deployed:** [N]
**Duration:** [time]

### Gate Summary
| Gate | Status | Key Finding |
|------|--------|-------------|
| 1: KNOW | complete | [blast radius tier, change types found] |
| 2: PLAN | complete | [rollback complexity, FMEA critical count] |
| 3: PROVE | [pass/fail] | [validator verdict summary] |

### Artifacts Produced
- `blast-radius.md` — dependency map of affected systems
- `change-classification.json` — change types and risk tiers
- `sla-exposure.md` — SLA/SLO impact analysis
- `rollback-plan.md` — executable rollback procedures
- `observability-gaps.md` — monitoring coverage analysis
- `fmea-matrix.md` — failure mode enumeration with RPN scores
- `runbook.md` — operator-facing deployment guide
- `chaos-scenarios.md` — testable failure scenarios

### [If FAIL] Blocking Issues
1. [Issue from validator]
2. [Issue from validator]

### [If PASS] Notes for Human Review
- [Areas the validator flagged for human judgment]
```

### Step 7: Artifact Location

All PRR artifacts are written to a session directory:

```
~/.claude/prr/{session_id}/
├── blast-radius.md
├── change-classification.json
├── sla-exposure.md
├── rollback-plan.md
├── observability-gaps.md
├── fmea-matrix.md
├── runbook.md
├── chaos-scenarios.md
└── prr-verdict.json
```

After presenting results, prompt: "PRR artifacts are at `~/.claude/prr/{session_id}/`. Want to commit these to the repo?"

### Step 8: Session Manifest

Write provenance to `~/.claude/provenance/sessions/{session_id}.yaml` following the standard swarm manifest format. Tag with `prr`, domain name, and risk tier.

---

## Agent Prompt Templates

When spawning each agent, include in the prompt:

1. The agent's role file content (from `~/.claude/agents/prr-*.md`)
2. The project's CLAUDE.md (if it exists in the target repo)
3. The specific task description with acceptance criteria
4. Paths to input files (diff, changed files list, upstream gate outputs)
5. The output directory: `~/.claude/prr/{session_id}/`

**Critical:** Each agent must write its output to the session directory, not to the working directory. This keeps PRR artifacts separate from the codebase.

---

## Anti-Patterns

- **Never skip the triage step** — deploying 8 agents for a README fix is wasteful
- **Never run Gate 2 before Gate 1 completes** — Gate 2 agents consume Gate 1 findings
- **Never run the validator before Gates 2+3 complete** — the validator needs all outputs
- **Never let the rollback strategist self-certify** — the validator reviews rollback plans too
- **Never produce a PRR without a runbook** — the runbook is the primary deliverable; everything else feeds into it
- **Never skip commit checkpoints between gates** — context compaction is real; commit early, commit often
- **Never run this on a dirty working tree** — `git stash` or commit first

## Lessons Learned

| Session | Lesson | Change Made |
|---------|--------|-------------|
| (initial) | First deployment — no field data yet | Will update after first real PRR run |

---

## Example Invocations

```
/prr-swarm                          # Review current branch vs main
/prr-swarm feature/auth-refactor    # Review specific branch
/prr-swarm #142                     # Review specific PR
/prr-swarm abc1234..def5678         # Review commit range
```
