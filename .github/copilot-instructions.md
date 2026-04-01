# Data Dictionary SDLC Framework — Copilot Instructions

## Project Overview

This project uses an **agentic SDLC pipeline** to build a **Data Dictionary API Portal**.
The system ingests OpenAPI specifications, parses them into structured metadata, stores
everything in PostgreSQL with full-text search, and exposes a search-driven API portal.

The pipeline is managed by specialized agents that handle each development phase,
with human checkpoints at critical gates.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Language** | Java 21+ (virtual threads, records, sealed interfaces, pattern matching) |
| **Framework** | Spring Boot 3.x (WebFlux or MVC, Spring Data JPA) |
| **Database** | PostgreSQL 15+ with full-text search (`tsvector`/`tsquery`, `pg_trgm`) |
| **Build** | Maven (multi-module where appropriate) |
| **Testing** | JUnit 5, Mockito, Testcontainers (PostgreSQL) |
| **Cloud** | Configurable per project (Azure, AWS, GCP, or on-premises) |

> **Note:** The cloud provider is determined during Phase 0 discovery. The relevant cloud skill is loaded for subsequent phases based on the team's chosen provider and existing infrastructure.

## Available Agents

| Agent | Phase | Purpose |
|-------|-------|---------|
| `@sdlc-orchestrator` | All | Master pipeline manager — coordinates phase execution |
| `@phase0-prd-discovery` | 0 | Requirements gathering → Product Requirements Document |
| `@phase1a-architecture-greenfield` | 1A | Greenfield architecture — new system design, DB schema, API contracts |
| `@phase1b-architecture-brownfield` | 1B | Brownfield architecture — integration design, schema migration, API extensions |
| `@phase2-codegen` | 2 | Java/Spring Boot code generation from architecture |
| `@phase3-testing` | 3 | Test suite generation (unit, integration, E2E) |
| `@phase4-review` | 4 | Automated code review and quality analysis |
| `@phase5-documentation` | 5 | API docs, README, runbook generation |
| `@get-status` | Utility | Pipeline status check and progress reporting |
| `@checkpoint-feedback` | Utility | Capture structured feedback at checkpoints |
| `@instinct-manager` | Utility | Manage instinct store, extract learnings |
| `@skill-evolver` | Utility | Propose SKILL.md updates from instincts |

## Workflow

1. Start with `@sdlc-orchestrator` for full pipeline or `@phase0-prd-discovery` for phase-by-phase
2. Phase 0 determines project type (greenfield/brownfield) which routes to the appropriate Phase 1 agent
3. Each agent produces artifacts in the `reports/` directory
4. Handoff documents are created in `handoffs/` at phase boundaries
5. **Mandatory human checkpoint** at Phase 1→2 (Architecture); **recommended Final Review** after Phase 5
6. Use `@get-status` at any time to check pipeline progress and next steps

## Coding Standards

### Java Conventions
- Use Java 21+ features: records for DTOs, sealed interfaces for type hierarchies,
  pattern matching in switch expressions, virtual threads for I/O-bound operations
- Follow standard Java naming: `PascalCase` for classes, `camelCase` for methods/fields
- Package structure: `com.datadict.{module}.{layer}` (e.g., `com.datadict.search.controller`)
- Prefer constructor injection over field injection for Spring beans
- Use `Optional` for nullable return values; never return `null` from public methods

### Spring Boot Patterns
- REST controllers return `ResponseEntity<T>` with proper HTTP status codes
- Service layer handles business logic; repositories handle data access only
- Use `@Transactional` at the service layer, not at the repository layer
- Configuration via `application.yml` with profile-specific overrides
- Health checks and actuator endpoints enabled for production readiness

### Testing Requirements
- **Unit tests**: ≥80% line coverage, ≥70% branch coverage
- **Integration tests**: Use Testcontainers for PostgreSQL; test full request/response cycles
- **Naming**: `{MethodName}_When{Condition}_Should{ExpectedResult}`
- Every public service method must have corresponding tests
- Use `@MockBean` sparingly; prefer real implementations with Testcontainers

### Database Conventions
- Table names: `snake_case`, plural (e.g., `api_endpoints`, `data_definitions`)
- Primary keys: `BIGSERIAL` named `id`
- Timestamps: `created_at` and `updated_at` on every table, with `TIMESTAMPTZ`
- Full-text search: maintain `tsvector` columns with triggers for auto-update
- Migrations: Flyway with `V{N}__{description}.sql` naming

## File Structure

```
datadict/
├── .github/
│   ├── agents/              # Agent definitions (*.agent.md)
│   ├── copilot-instructions.md  # This file
│   ├── instructions/        # Shared instruction fragments
│   ├── instincts/           # Learned patterns (per-project instinct store)
│   ├── prompts/             # Reusable prompt templates
│   └── skills/              # Skill definitions
├── handoffs/                # Phase-to-phase handoff documents
│   ├── HANDOFF-TEMPLATE.md  # Reusable handoff template
│   └── CHECKPOINT-DEFINITIONS.md
├── reports/                 # Generated artifacts and reports
│   ├── Report-Status.md     # Pipeline status tracker
│   ├── feedback/            # Structured checkpoint feedback
│   └── *.md                 # Phase-specific deliverables
├── src/                     # Application source (generated by Phase 2)
├── ORCHESTRATION-BLUEPRINT.md  # Full pipeline design document
└── pom.xml                  # Maven build (generated by Phase 2)
```

## Report Artifacts

Each phase produces specific reports in `reports/`:

| Phase | Report | Description |
|-------|--------|-------------|
| 0 | `Product-Requirements-Document.md` | PRD with user stories and acceptance criteria |
| 1 | `Architecture-Decision-Record.md` | ADR with trade-off analysis |
| 1 | `Database-Schema-Design.md` | PostgreSQL schema with FTS setup |
| 1 | `API-Contract.md` | REST API contract with endpoints |
| 3 | `Test-Coverage-Report.md` | Coverage metrics and gap analysis |
| 4 | `Code-Review.md` | Review findings by severity |
| 5 | `Documentation-Checklist.md` | Documentation completeness tracker |

## Human Checkpoints

See `handoffs/CHECKPOINT-DEFINITIONS.md` for full details. Summary:

- **CP1 (Mandatory)**: Architecture Review after Phase 1 — review ADR, schema, API contract
- **CP2 (Recommended)**: Final Review after Phase 5 — review code quality, coverage, docs completeness
- **R1 (Recommended)**: PRD Review after Phase 0
- **R2 (Conditional)**: Post-Code-Review after Phase 4 (blocks only on CRITICAL findings)

## Learning Loop

The pipeline improves across runs via a continuous learning loop. At every checkpoint,
`@checkpoint-feedback` captures structured reviewer feedback into `reports/feedback/`.
After approval, `@instinct-manager` extracts reusable patterns into `.github/instincts/`.
When 5+ related instincts accumulate, `@skill-evolver` proposes updates to the relevant
`SKILL.md` files. Instincts persist per-project and can be copied to seed new projects.

## Context Rules for Agents

- Always read handoff documents from the previous phase before starting work
- Always update `reports/Report-Status.md` when changing phase status
- Write handoff documents using `handoffs/HANDOFF-TEMPLATE.md` as the base
- Reference the `ORCHESTRATION-BLUEPRINT.md` for pipeline design decisions
- Keep report artifacts self-contained — each should be readable without other context

---

*Data Dictionary SDLC Framework — Managed by GitHub Copilot Agents*
