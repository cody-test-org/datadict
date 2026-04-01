# SDLC Pipeline — Human Checkpoints

> Defines all human-in-the-loop checkpoints in the agentic SDLC pipeline.
> Agents pause at these gates and write handoff documents before human review.

---

## Checkpoint Overview

```
Phase 0 ──→ [R1] ──→ Phase 1 ──→ [CP1] ──→ Phase 2 ──→ Phase 3 ──→ Phase 4 ──→ [R2] ──→ Phase 5 ──→ [FR] ──→ ✅ Done
             ▲                      ▲                                              ▲                      ▲
         Recommended            MANDATORY                                     Recommended           Recommended
```

| Checkpoint | Location | Type | Blocking |
|------------|----------|------|----------|
| R1 | Phase 0 → Phase 1 | Recommended | No (skippable) |
| CP1 | Phase 1 → Phase 2 | Mandatory | Yes |
| R2 | Phase 4 → Phase 5 | Recommended | Conditional |
| FR | After Phase 5 | Recommended | No (skippable) |

---

## Mandatory Checkpoints

### Checkpoint 1 (CP1): Architecture Review — Phase 1 → Phase 2

- **When**: After the Architecture agent produces the ADR, database schema, and API contract
- **What to Review**:
  - ADR trade-offs and technology choices
  - PostgreSQL schema design (tables, indexes, FTS configuration)
  - API contract completeness and REST conventions
  - Security model and authentication approach
- **Artifacts to Inspect**:
  - `reports/Architecture-Decision-Record.md`
  - `reports/Database-Schema-Design.md`
  - `reports/API-Contract.md`
- **Approval Method**: Review artifacts, then invoke `@phase2-codegen` to signal approval
- **Blocking**: **Yes** — code generation cannot proceed without architecture approval
- **Estimated Review Time**: 30–60 minutes
- **Rejection Protocol**: Add comments to the handoff document and re-invoke `@phase1-architecture` with feedback

---

## Recommended Checkpoints

### Checkpoint R1: PRD Review — Phase 0 → Phase 1

- **When**: After the PRD agent generates the Product Requirements Document
- **What to Review**:
  - Requirements completeness and clarity
  - User story accuracy and acceptance criteria
  - Scope alignment with project goals
  - Priority and phasing decisions
- **Artifacts to Inspect**:
  - `reports/Product-Requirements-Document.md`
- **Blocking**: **Recommended but skippable** — Phase 1 can proceed with generated PRD
- **Estimated Review Time**: 15–30 minutes
- **Skip Criteria**: PRD generated from well-defined input specs with no ambiguity

### Checkpoint R2: Post-Code-Review — Phase 4 → Phase 5

- **When**: After the code review agent produces findings
- **What to Review**:
  - Critical and high-severity findings
  - Security vulnerabilities flagged
  - Performance concerns identified
- **Artifacts to Inspect**:
  - `reports/Code-Review.md`
- **Blocking**: **Conditional** — only blocks if CRITICAL findings exist
- **Estimated Review Time**: 15–45 minutes
- **Auto-Pass Criteria**: No CRITICAL findings AND no HIGH security findings

### Checkpoint FR: Final Review — After Phase 5

- **When**: After documentation is complete and the SDLC pipeline has finished
- **What to Review**:
  - Overall code quality and adherence to standards
  - Test coverage meets threshold (≥80% line, ≥70% branch)
  - Documentation completeness (API docs, README, runbook)
  - All open issues resolved or triaged
- **Artifacts to Inspect**:
  - `reports/Code-Review.md`
  - `reports/Test-Coverage-Report.md`
  - `reports/Documentation-Checklist.md`
  - `reports/Report-Status.md`
- **Blocking**: **No** — recommended but skippable
- **Estimated Review Time**: 30–60 minutes
- **Skip Criteria**: All mandatory checkpoints passed, quality metrics meet thresholds

---

## Checkpoint Protocol

All agents follow this protocol when reaching a checkpoint:

### 1. Agent Writes Handoff Document

```
Agent copies handoffs/HANDOFF-TEMPLATE.md → handoffs/handoff-phaseN-to-phaseM.md
Agent fills in all placeholders with phase-specific data
```

### 2. Agent Updates Pipeline Status

```
Agent updates reports/Report-Status.md:
  - Current phase status → "✅ COMPLETE"
  - Next phase status → "⏸️ AWAITING REVIEW"
  - Checkpoint reference → link to handoff document
```

### 3. Human Reviews Artifacts

- Human reads the handoff document for context
- Human inspects referenced artifacts in `reports/`
- Human checks quality metrics against thresholds

### 4. Human Signals Approval

- **Approve**: Invoke the next phase agent (e.g., `@phase2-codegen`)
- **Reject**: Add feedback to the handoff document and re-invoke the current phase agent
- **Partial Approve**: Invoke next phase with specific constraints noted in handoff

### 5. Next Agent Reads Handoff and Proceeds

```
Next agent reads handoffs/handoff-phaseN-to-phaseM.md
Next agent loads referenced artifacts
Next agent begins phase work with full context
```

---

## Checkpoint Configuration

Checkpoints can be adjusted per-project by modifying this file:

| Setting | Default | Options |
|---------|---------|---------|
| CP1 Blocking | Yes | Yes / No |
| R1 Enabled | Yes | Yes / No |
| R2 Enabled | Yes | Yes / No |
| R2 Auto-Pass | CRITICAL-only | CRITICAL-only / HIGH-and-above / All |
| FR Enabled | Yes | Yes / No |

---

*Part of the Data Dictionary SDLC Framework*
