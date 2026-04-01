---
name: prd-generation
description: >-
  Domain knowledge for generating Product Requirements Documents (PRDs) from project
  descriptions. Covers user story formats, acceptance criteria patterns, requirement
  categorization, gap analysis techniques, and PRD structure. Use when discovering
  requirements and generating PRDs.
---

# PRD Generation Skill

## Purpose

This skill provides structured templates and techniques for generating Product Requirements
Documents (PRDs) from project descriptions, stakeholder input, and existing system analysis.
It ensures consistent, thorough requirements documentation that can drive development work.

## PRD Structure Template

Every PRD should follow this 12-section format:

### Section 1: Executive Summary
- One-paragraph project overview
- Primary business objective
- Target audience
- Expected outcome and success metrics

### Section 2: Problem Statement
- What problem exists today
- Who is affected and how
- Cost of inaction (quantified if possible)
- Current workarounds

### Section 3: Goals and Non-Goals

**Goals** — what this project will accomplish:
- Specific, measurable outcomes
- Tied to business metrics

**Non-Goals** — what this project will NOT do (equally important):
- Explicitly excluded scope
- Adjacent problems intentionally deferred

### Section 4: User Personas
- Name, role, and context
- Pain points relevant to this project
- Success criteria from their perspective

### Section 5: User Stories and Requirements
- Organized by persona or feature area
- Prioritized using MoSCoW (see below)
- Each story has acceptance criteria

### Section 6: Functional Requirements
- Detailed feature descriptions
- Input/output specifications
- Business rules and validations

### Section 7: Non-Functional Requirements
- Performance, security, scalability targets
- Use the NFR checklist below

### Section 8: System Architecture (High Level)
- Component diagram or description
- Integration points
- Data flow overview

### Section 9: Data Requirements
- Data models and schemas
- Data sources and sinks
- Retention and privacy considerations

### Section 10: UX/UI Requirements
- Wireframes or screen descriptions
- Interaction patterns
- Accessibility requirements

### Section 11: Release Plan
- MVP scope (Phase 1)
- Subsequent phases
- Dependencies and blockers

### Section 12: Open Questions and Risks
- Unresolved decisions
- Technical risks
- Dependencies on external teams

## User Story Format

### Template

```
As a [persona],
I want to [action/capability],
So that [benefit/value].

Priority: P0 | P1 | P2 | P3
```

### Priority Levels

| Priority | Meaning         | Criteria                                        |
|----------|-----------------|-------------------------------------------------|
| P0       | Must-have       | Launch blocker. System is unusable without it.   |
| P1       | Should-have     | Important for launch but has a workaround.       |
| P2       | Nice-to-have    | Enhances experience. Can ship without it.        |
| P3       | Future           | Desired but explicitly deferred to later phase.  |

### Example User Stories

```
As an API developer,
I want to search for fields by name or description,
So that I can quickly find the data elements I need.

Priority: P0
```

```
As a data steward,
I want to export field definitions to Excel,
So that I can share them with stakeholders who don't use the tool.

Priority: P1
```

```
As a platform engineer,
I want to see which API endpoints use a specific field,
So that I can assess the impact of schema changes.

Priority: P0
```

## Acceptance Criteria

### Given/When/Then Format

```
Given [precondition or context],
When [action is performed],
Then [expected outcome].
```

### Example Acceptance Criteria

```
Story: Search for fields by name

AC1:
  Given the user is on the field search page,
  When they type "user" into the search box,
  Then fields containing "user" in their name are displayed within 500ms.

AC2:
  Given the search returns results,
  When a result is clicked,
  Then the field detail view is shown with all metadata.

AC3:
  Given the search query matches no fields,
  When results are rendered,
  Then an empty state message is displayed with suggestions.

AC4:
  Given the user misspells a field name (e.g., "usr_naem"),
  When the search executes,
  Then fuzzy matching returns relevant results with similarity scores.
```

### Acceptance Criteria Best Practices

- Each story should have 3–7 acceptance criteria
- Cover the happy path, edge cases, and error states
- Be testable — each AC maps to at least one test case
- Include performance expectations where relevant (e.g., "within 500ms")
- Avoid implementation details; describe observable behavior

## Gap Analysis Techniques

### Questions to Ask

When reviewing a project description, systematically ask:

**User & Access**
- Who are the different user types?
- What permissions/roles exist?
- Is authentication required? What provider?
- Is there multi-tenancy?

**Data**
- Where does the data come from?
- How is data validated on input?
- What happens to stale or deleted data?
- Are there data privacy requirements (PII, GDPR)?

**Integration**
- What external systems does this integrate with?
- What happens when an integration is unavailable?
- Are there rate limits or quotas to handle?
- What authentication do integrations use?

**Operations**
- How will we know if the system is healthy?
- What metrics matter?
- How are errors surfaced to users vs. operators?
- What is the backup/recovery strategy?

**Edge Cases**
- What happens with empty states?
- How does the system handle concurrent modifications?
- What are the upper bounds (max records, max file size)?
- What happens at scale (10x, 100x current expectations)?

### Common Blind Spots

1. **Error handling** — what does the user see when things fail?
2. **Empty states** — first-time user experience with no data
3. **Bulk operations** — what if someone uploads 10,000 records?
4. **Audit trail** — who changed what and when?
5. **Migration** — how does existing data get into the new system?
6. **Offline/degraded mode** — what works when dependencies are down?
7. **Internationalization** — character encoding, locale, timezone
8. **Accessibility** — keyboard navigation, screen readers, contrast

## Non-Functional Requirements Checklist

### Performance
- [ ] Page load time targets (e.g., < 2 seconds)
- [ ] API response time targets (e.g., p95 < 500ms)
- [ ] Search response time targets
- [ ] Maximum concurrent users
- [ ] Data volume expectations (rows, storage)

### Security
- [ ] Authentication method (OAuth2, SAML, API keys)
- [ ] Authorization model (RBAC, ABAC)
- [ ] Data encryption (at rest, in transit)
- [ ] Input validation and sanitization
- [ ] Rate limiting
- [ ] Audit logging
- [ ] Vulnerability scanning requirements

### Accessibility
- [ ] WCAG compliance level (A, AA, AAA)
- [ ] Keyboard navigation support
- [ ] Screen reader compatibility
- [ ] Color contrast ratios
- [ ] Focus management

### Scalability
- [ ] Expected growth rate
- [ ] Horizontal scaling requirements
- [ ] Database scaling strategy
- [ ] Caching requirements
- [ ] CDN requirements

### Reliability
- [ ] Uptime target (99.9%, 99.99%)
- [ ] Recovery Time Objective (RTO)
- [ ] Recovery Point Objective (RPO)
- [ ] Backup frequency
- [ ] Disaster recovery plan

### Observability
- [ ] Logging standards and retention
- [ ] Metrics and dashboards
- [ ] Alerting thresholds
- [ ] Distributed tracing
- [ ] Health check endpoints

## Requirement Categorization (MoSCoW)

| Category    | Code | Definition                                                    |
|-------------|------|---------------------------------------------------------------|
| Must-have   | M    | Non-negotiable. System fails its purpose without it.          |
| Should-have | S    | Important. Expected by users but has a workaround.            |
| Could-have  | C    | Desirable. Enhances user experience if time permits.          |
| Won't-have  | W    | Explicitly excluded from this release. Documented for later.  |

### Categorization Heuristic

Ask these questions to categorize a requirement:

1. **Can the system launch without it?** No → Must-have
2. **Will users be significantly frustrated without it?** Yes → Should-have
3. **Does it differentiate us from alternatives?** Yes → Could-have
4. **Is it tangential to the core problem?** Yes → Won't-have (this release)

## Stakeholder Mapping

### Who Cares About What

| Stakeholder         | Primary Concerns                                    |
|---------------------|-----------------------------------------------------|
| Product Owner       | Business value, prioritization, timeline             |
| End Users           | Usability, performance, reliability                  |
| Developers          | Technical feasibility, maintainability, clear specs  |
| QA Engineers        | Testability, acceptance criteria, edge cases         |
| Security Team       | Auth, data protection, compliance                    |
| Operations/SRE      | Monitoring, deployment, scaling, incident response   |
| Data Team           | Data models, integrations, quality, governance       |
| Leadership          | ROI, timeline, risk, competitive advantage           |

### RACI Matrix Template

For each major feature, define:

- **R** (Responsible) — who does the work
- **A** (Accountable) — who approves/owns the decision
- **C** (Consulted) — whose input is needed
- **I** (Informed) — who needs to know the outcome

```
| Feature           | Product | Engineering | Design | QA  | Security |
|-------------------|---------|-------------|--------|-----|----------|
| Field Search      |   A     |     R       |   C    |  C  |    I     |
| Data Export        |   A     |     R       |   I    |  C  |    C     |
| API Auth           |   A     |     R       |   I    |  C  |    C     |
```
