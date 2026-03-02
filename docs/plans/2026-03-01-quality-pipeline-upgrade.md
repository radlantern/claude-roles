# Quality Pipeline Upgrade

**Date:** 2026-03-01
**Status:** Design validated — pending test

## Problem

The Coder goes straight from a task spec to TDD implementation. If the design is
wrong, tests get written against the wrong design and the full review cycle runs
before anyone catches it. Design decisions are the most expensive thing to fix late.

## Changes

### 1. Critical risk tier (new)

A fourth tier above High for money movement and gambling outcome determination.

| Tier | Applies To |
|------|-----------|
| **Critical** | Balance mutations, bet settlement, payout calculations, gambling outcome determination, withdrawals |
| **High** | Auth, session, state machines, audit trail |
| **Medium** | Repositories, validation, read-only services |
| **Low** | Config, docs, migrations, test helpers |

### 2. Design Agent (new role: `design-agent.md`)

Sits between Tasker analysis and Coder dispatch for Critical and High risk tasks.

**Produces:** 2-3 competing designs, each with trade-offs and a recommendation.

**Tasker:** selects one design, or escalates to human if it cannot decide.

**Coder:** receives Task Assignment + Approved Design Spec. Deviations from the
approved design are flagged as interface violations in the Completion Report.

**For Critical risk only — Design Agent also spawns (in parallel):**
- Financial invariants sub-agent: validates the design enforces financial invariants
  (balance never negative, idempotency designed in, no double-payout race)
- Schema & data model sub-agent: validates FK indexes, NOT NULL migration strategy,
  idempotency constraints at the DB level

**Cap:** 2 design sub-agents. Further concerns flagged as risks in the Design Spec,
not additional agents.

**When it applies:**

| Risk | Design Agent |
|------|-------------|
| Critical | Mandatory + 2 sub-agents |
| High | Mandatory, no sub-agents |
| Medium | Optional — Tasker judgment |
| Low | Skip |

### 3. Three-model review panel (update to `tasker.md`)

Replace the existing dual-reviewer (Claude + Codex) with a three-model panel:

- **Reviewer A:** Claude subagent (reads `reviewer.md`)
- **Reviewer B:** Codex CLI
- **Reviewer C:** Gemini CLI

All three dispatched in parallel.

### 4. Focused sub-agents within Reviewer A (update to `reviewer.md`)

**Tier 1 — Mandatory, based on code content:**

| Code contains | Focused agent spawned |
|---------------|----------------------|
| SQL / ORM / migrations | DB & query agent |
| Balance, amount, payout | Financial integrity agent |
| Auth, tokens, session | Auth & permissions agent |
| Goroutines, mutexes, shared state | Concurrency agent |

**Tier 2 — Triggered, based on broad review findings:**

For each dimension scoring < 4/5 or producing a Critical/High finding, spawn one
focused agent to deep-dive that specific area. Cap: 3 triggered agents.

**Hard cap: 5 focused agents total** (Tier 1 + Tier 2 combined). If the code
touches enough things to exceed 5, the change is too large and should be split.

### 5. Updated verdict standards

| | Critical | High | Medium | Low |
|--|---------|------|--------|-----|
| Reviewer consensus | 3/3 | 2/3 | 1 reviewer | Self |
| Quality threshold | 23/25 | 21/25 | 20/25 | — |
| MEDIUM findings | Human sign-off to defer | Logged | Logged | — |
| Security Linter | Gates review (FAIL blocks) | Mandatory | Mandatory | — |
| Design Agent | Mandatory + sub-agents | Mandatory | Optional | Skip |

### 6. Security Linter gates Critical review

For Critical risk, the Security Linter runs before the three-model panel. Any FAIL
blocks the review entirely — no point running three model reviewers on code with a
confirmed SQL injection or integer overflow.

## Full pipeline

```
Human Request
     ↓
[Tasker] — classify risk, find references, edge cases, state machine analysis
     ↓
     ├── Low ──────────────────────────────────────────────► Coder
     │
     ├── Medium ─── Design Agent (optional) ──────────────► Coder
     │
     └── High / Critical
              └── Design Agent
                    ├── Design A (trade-offs)
                    ├── Design B (trade-offs)
                    └── Design C (optional)
                  + [Critical only, parallel]:
                    ├── Financial invariants sub-agent
                    └── Schema & data model sub-agent
                              ↓
                    Tasker selects design
                    (or escalates to human)
                              ↓
                            Coder
                (Task Assignment + Approved Design Spec)
                              ↓
                       Completion Report
                              ↓
         [Critical] Security Linter ── FAIL → blocks review
                              ↓ PASS
         ┌─────────────────────────────────────────────┐
         │ Reviewer A: Claude + focused sub-agents     │
         │ Reviewer B: Codex CLI                       │
         │ Reviewer C: Gemini CLI                      │
         └─────────────────────────────────────────────┘
                              ↓
                  Tasker merges verdicts
                              ↓
         APPROVED ✅  ITERATE (CRITICAL/HIGH only, max 3)  REJECT ❌
```

## Escalation

The Tasker is the only agent that escalates to the human. All other agents report
to their parent:

```
Human
  └── Tasker
        ├── Design Agent
        │     ├── Financial invariants sub-agent
        │     └── Schema sub-agent
        ├── Coder
        ├── Security Linter
        └── Reviewer A (Claude)
              └── Focused sub-agents (up to 5)
        └── Reviewer B (Codex CLI)
        └── Reviewer C (Gemini CLI)
```

## Implementation plan

New files needed:
- `design-agent.md` — Design Agent role

Files to update:
- `tasker.md` — Critical tier, Design Agent dispatch, Gemini as Reviewer C,
  updated verdict thresholds
- `reviewer.md` — Focused sub-agent rules (Tier 1 triggers, Tier 2 triggers,
  hard cap of 5)
- `README.md` — Gemini CLI setup, design-agent.md in role table

## Testing approach

Before rewriting all role files, test the Design Agent in isolation:

1. Pick one upcoming Critical or High risk task in evenplay-mono
2. Manually invoke: "Read `.claude/roles/design-agent.md` and produce 2-3 designs for: [task]"
3. Evaluate: does the Design Agent catch design problems the Coder would have made?
4. If yes, proceed with full role file updates
