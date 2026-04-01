# Skills, Tools, MCP Servers & Libraries Mapping

## Comprehensive SDLC Framework Reference for Data Dictionary on API Portal

> **Use Case**: Consolidated, searchable data dictionary auto-generated from OpenAPI specs.
> OpenAPI parsing → PostgreSQL storage with full-text search → Search UI on API Portal with autocomplete, filters, CSV/Excel export.

---

## Table of Contents

1. [SDLC Phase Overview](#sdlc-phase-overview)
2. [GitHub Copilot CLI Skills Mapping](#1-github-copilot-cli-skills-mapping)
3. [Java/Spring Boot Libraries & Tools](#2-javaspring-boot-libraries--tools)
4. [MCP Servers & Extensions](#3-mcp-servers--extensions)
5. [External Tools](#4-external-tools)
6. [Consolidated SDLC × Tool Matrix](#5-consolidated-sdlc--tool-matrix)
7. [Recommended Stack (Opinionated)](#6-recommended-stack-opinionated)

---

## SDLC Phase Overview

| Phase | Description | Data Dictionary Activities |
|-------|-------------|---------------------------|
| **1. Requirements** | Gather, analyze, validate requirements | Define what fields to extract from OpenAPI specs, search UX requirements, export formats |
| **2. Architecture** | Design system architecture, data models | Design PostgreSQL schema, search indexes, API endpoints, frontend components |
| **3. Infrastructure** | Provision cloud resources | Azure PostgreSQL, Container Apps/App Service, Storage, networking, RBAC |
| **4. Implementation** | Write application code | OpenAPI parser, DB layer, search API, frontend UI, export functionality |
| **5. Testing** | Unit, integration, E2E testing | Parser tests, search accuracy tests, UI E2E tests, export validation |
| **6. Deployment** | CI/CD, release management | GitHub Actions, azd up/deploy, database migrations |
| **7. Operations** | Monitoring, observability, maintenance | App Insights, Log Analytics, health checks, cost optimization |

---

## 1. GitHub Copilot CLI Skills Mapping

### Phase 3: Infrastructure — Azure Skills

#### `azure-prepare` — **🔴 Critical**
**What it does**: Analyzes your project and generates infrastructure-as-code (Bicep/Terraform), `azure.yaml`, and Dockerfiles.
**How it applies to Data Dictionary**:
- Generates Bicep/Terraform for Azure PostgreSQL Flexible Server
- Creates Container App or App Service definitions for the API + frontend
- Generates `azure.yaml` for `azd` orchestration
- Scaffolds Dockerfiles for the Java/Spring Boot services
- Can add Azure Blob Storage for storing uploaded OpenAPI spec files
- Can add Azure AI Search if enhancing beyond PostgreSQL FTS

**When to invoke**: At the start of the project after code structure exists. Re-invoke when adding new Azure services (e.g., adding Redis cache, Storage).

**Example prompt**: *"Prepare this Java Spring Boot project for Azure deployment with PostgreSQL Flexible Server, Container Apps, and Blob Storage"*

---

#### `azure-deploy` — **🔴 Critical**
**What it does**: Runs `azd up`, `azd deploy`, or infrastructure provisioning commands.
**How it applies**:
- Executes `azd provision` to create Azure resources (PostgreSQL, Container Apps, Storage)
- Runs `azd deploy` to push application code
- Handles Bicep/Terraform apply workflows
- Manages environment configuration

**When to invoke**: After `azure-prepare` has generated IaC, whenever deploying to any environment (dev/staging/prod).

**Example prompt**: *"Deploy the data dictionary to Azure using azd up"*

---

#### `azure-postgres` — **🔴 Critical**
**What it does**: Creates Azure Database for PostgreSQL Flexible Server instances and configures Entra ID passwordless authentication.
**How it applies**:
- Provisions the PostgreSQL Flexible Server that stores the parsed data dictionary
- Configures Entra ID (Azure AD) authentication — no passwords in connection strings
- Sets up managed identity access for Container Apps → PostgreSQL
- Configures developer access for local development
- Handles group-based permissions

**When to invoke**: During initial infrastructure setup and when onboarding developers who need DB access.

**Critical for this use case** because PostgreSQL is the core data store and search engine (tsvector/tsquery/pg_trgm).

---

#### `azure-ai` — **🟡 Helpful**
**What it does**: Manages Azure AI Search, Speech, OpenAI, Document Intelligence.
**How it applies**:
- **Azure AI Search** could supplement or replace PostgreSQL full-text search for very large datasets (>10M rows)
- Provides vector search / hybrid search capabilities for semantic search across data dictionary entries
- Could power "smart search" that understands field relationships beyond keyword matching

**When to invoke**: If PostgreSQL FTS proves insufficient for scale/features, or if semantic search is desired.

**Trade-off**: PostgreSQL FTS + pg_trgm handles most use cases well up to ~10M rows. Azure AI Search adds cost but provides enterprise-grade search with faceting, scoring profiles, and semantic ranking.

---

#### `azure-storage` — **🟡 Helpful**
**What it does**: Manages Azure Blob/File/Queue/Table Storage.
**How it applies**:
- **Blob Storage**: Store uploaded OpenAPI specification files (YAML/JSON)
- **Blob versioning**: Track spec version history
- **Lifecycle management**: Archive old spec versions to cool/archive tier
- Could store generated CSV/Excel export files for download

**When to invoke**: When implementing the OpenAPI spec upload/storage pipeline.

---

#### `azure-observability` — **🟡 Helpful**
**What it does**: Azure Monitor, Application Insights, Log Analytics, Alerts, Workbooks.
**How it applies**:
- Set up Azure Monitor dashboards for the data dictionary service
- Configure alerts for API latency, error rates, database connection issues
- Create Log Analytics workbooks for search analytics (popular searches, zero-result queries)
- KQL queries for operational intelligence

**When to invoke**: Post-deployment, during the operations phase.

---

#### `azure-diagnostics` — **🟡 Helpful**
**What it does**: Debug and troubleshoot production issues on Azure.
**How it applies**:
- Troubleshoot Container App issues (image pull failures, cold starts, health probes)
- Analyze PostgreSQL connection issues or slow queries via KQL logs
- Debug search performance issues in production
- Investigate parsing failures for malformed OpenAPI specs

**When to invoke**: When production issues arise — health probe failures, search latency spikes, parsing errors.

---

#### `azure-rbac` — **🟡 Helpful**
**What it does**: Finds the right Azure RBAC role with least privilege access.
**How it applies**:
- Determine what roles the Container App's managed identity needs (PostgreSQL access, Blob Storage access)
- Set up developer roles for the team
- Configure service principal permissions for CI/CD pipeline
- Ensure least-privilege access across all resources

**When to invoke**: During infrastructure setup and when adding team members or services.

---

#### `azure-compliance` — **🟢 Optional**
**What it does**: Security auditing, best practices assessment, Key Vault monitoring.
**How it applies**:
- Audit the data dictionary infrastructure for compliance
- Check for expired certificates or secrets
- Validate resource configurations against best practices
- Run Azure Quick Review (azqr) scans

**When to invoke**: Before production launch and periodically thereafter.

---

#### `azure-resource-visualizer` — **🟢 Optional**
**What it does**: Generates Mermaid architecture diagrams from Azure resource groups.
**How it applies**:
- Visualize the deployed data dictionary architecture
- Generate documentation diagrams showing PostgreSQL ↔ Container App ↔ Storage relationships
- Useful for architecture reviews and onboarding docs

**When to invoke**: After deployment, for documentation and architecture review.

---

#### `azure-cost-optimization` — **🟢 Optional**
**What it does**: Analyzes Azure costs and generates optimization recommendations.
**How it applies**:
- Identify if the PostgreSQL tier is oversized
- Find cost savings on Container App scaling configurations
- Optimize storage costs with lifecycle policies
- Right-size compute resources based on actual usage

**When to invoke**: Monthly or quarterly cost reviews, or when Azure spend exceeds expectations.

---

#### `azure-aigateway` — **🟢 Optional**
**What it does**: Configures Azure API Management as an AI Gateway.
**How it applies**:
- If the data dictionary exposes an API consumed by multiple teams, APIM can provide rate limiting, caching, and governance
- Could front the search API with semantic caching for repeated queries
- API analytics for understanding usage patterns

**When to invoke**: If the data dictionary API needs enterprise-grade API management (rate limiting, API keys, analytics).

---

#### `appinsights-instrumentation` — **🟡 Helpful**
**What it does**: Guidance for instrumenting apps with Azure Application Insights SDK.
**How it applies**:
- Instrument the Spring Boot API with App Insights SDK for distributed tracing
- Track custom metrics: search latency, parse duration, export generation time
- Capture dependency tracking (PostgreSQL queries, blob storage operations)
- Set up custom telemetry for search analytics

**When to invoke**: During implementation of the API layer, before production deployment.

---

### Phase 5: Testing — Agent Skills

#### `code-review` agent — **🔴 Critical**
**What it does**: Reviews code changes with high signal-to-noise ratio. Only surfaces bugs, security vulnerabilities, logic errors.
**How it applies**:
- Review OpenAPI parser code for edge cases (allOf/oneOf/anyOf handling, circular $ref)
- Validate database query safety (SQL injection prevention)
- Check search implementation for performance issues
- Review export code for memory management (large CSV/Excel files)

**When to invoke**: Before every PR merge. Critical for parser code and database interactions.

---

#### `testing-agent` — **🟡 Helpful**
**What it does**: Specialized for test creation and coverage analysis.
**How it applies**:
- Generate unit tests for the OpenAPI parser (edge cases, malformed specs)
- Create integration tests for the search API (query accuracy, pagination)
- Build E2E test scenarios for the search UI
- Analyze test coverage gaps

**When to invoke**: During the testing phase, and when implementing new features.

---

#### `architecture-reviewer` — **🟡 Helpful**
**What it does**: Validates DDD patterns, SOLID principles, and architectural decisions.
**How it applies**:
- Review the overall data dictionary architecture (separation of concerns)
- Validate the parser → storage → search → UI data flow
- Check for proper abstraction layers (repository pattern for DB, service layer for business logic)
- Ensure the design supports extensibility (adding new OpenAPI spec versions, new export formats)

**When to invoke**: During architecture phase and major refactors.

---

## 2. Java/Spring Boot Libraries & Tools

### 2A. OpenAPI Parsing

| Library | Relevance | Maven Coordinates | Description |
|---------|-----------|-------------------|-------------|
| **swagger-parser** | 🔴 Critical | `io.swagger.parser.v3:swagger-parser` | Parse, validate, dereference, bundle OpenAPI 2.0/3.0/3.1 specs. Handles `$ref` resolution, circular references, and multi-file specs. De facto standard in the Java ecosystem. |
| **openapi-generator** | 🟡 Helpful | `org.openapitools:openapi-generator-maven-plugin` | Generates Java POJOs, API clients, and server stubs from OpenAPI specs. Use to generate strongly-typed model classes matching extracted schema shapes. Also available as CLI. |
| **springdoc-openapi** | 🟡 Helpful | `org.springdoc:springdoc-openapi-starter-webmvc-ui` | Auto-generates OpenAPI docs from Spring MVC controllers. Useful for exposing the data dictionary's own API as an OpenAPI spec. Includes Swagger UI. |
| **Redocly CLI** | 🟡 Helpful | Language-agnostic CLI (install via `npm -g @redocly/cli` or Docker) | Lint, bundle, and validate OpenAPI specs before parsing. Supports 2.0/3.0/3.1/3.2 + AsyncAPI. Language-agnostic, run in CI/CD. |
| **swagger-core** | 🟢 Optional | `io.swagger.core.v3:swagger-core` | Core Swagger 2.x / OpenAPI 3.x model classes. Foundation types used by swagger-parser. Useful for lower-level programmatic manipulation. |

**Recommendation for Data Dictionary**:
- **Primary**: `io.swagger.parser.v3:swagger-parser` for parsing + dereferencing
- **Validation**: `Redocly CLI` for pre-parse linting/validation in CI/CD
- **Code generation**: `openapi-generator-maven-plugin` for generating typed POJOs from spec schemas
- **Key parsing challenges**: `$ref` resolution, `allOf`/`oneOf`/`anyOf` composition, nested objects, circular references, discriminator mappings

---

### 2B. Database (ORM / Data Access)

| Library | Relevance | Maven Coordinates | Description |
|---------|-----------|-------------------|-------------|
| **Spring Data JPA + Hibernate** | 🔴 Critical (Recommended) | `org.springframework.boot:spring-boot-starter-data-jpa` | Standard JPA persistence with Hibernate as the ORM. Repository pattern with derived queries, custom JPQL, and native SQL. Excellent Spring Boot integration. Use `@Query(nativeQuery=true)` for FTS queries. |
| **jOOQ** | 🟡 Helpful (Alternative) | `org.jooq:jooq` | Typesafe SQL query builder that generates Java code from your database schema. SQL-first approach — excellent for complex FTS queries (`tsvector`, `tsquery`). Closer to SQL than JPA. |
| **JdbcTemplate** | 🟡 Helpful | `org.springframework.boot:spring-boot-starter-jdbc` | Lightweight JDBC abstraction for raw SQL. Maximum control for FTS queries. Use alongside Spring Data JPA for queries the ORM can't express elegantly. |
| **Flyway** | 🔴 Critical | `org.flywaydb:flyway-core` | Database migration tool. SQL-based migrations are ideal for creating FTS indexes, tsvector columns, and pg_trgm extensions. First-class Spring Boot integration. |
| **Liquibase** | 🟢 Optional | `org.liquibase:liquibase-core` | Alternative migration tool with XML/YAML/JSON changesets. More complex than Flyway but supports rollback and diff. |
| **HikariCP** | 🟡 Helpful | Included in Spring Boot | High-performance JDBC connection pool. Spring Boot's default — no explicit config needed. Crucial for handling concurrent search queries. |

**Recommendation**:
- **Primary**: **Spring Data JPA** — standard repository pattern, derived queries, and `@Query(nativeQuery=true)` for FTS
- **Supplementary**: **JdbcTemplate** for complex FTS queries that need raw `tsvector`/`tsquery` syntax not easily expressed in JPQL
- **Alternative**: **jOOQ** if the team prefers a SQL-first, typesafe query builder (excellent for FTS but adds build complexity with code generation)
- **Migrations**: **Flyway** — SQL-based migrations are ideal for PostgreSQL FTS setup (extensions, indexes, triggers)

---

### 2C. PostgreSQL Full-Text Search

| Technique | Relevance | Description |
|-----------|-----------|-------------|
| **tsvector/tsquery** | 🔴 Critical | Core PostgreSQL FTS. Tokenizes, stems, normalizes text. Supports boolean operators, phrase search, prefix matching. Use for ranked keyword search across field names, descriptions, types. |
| **GIN indexes on tsvector** | 🔴 Critical | Essential for FTS performance. Create stored `search_vector` column with GIN index. Use `setweight()` for field importance ranking (field name = 'A', description = 'B', type = 'C'). |
| **pg_trgm extension** | 🔴 Critical | Trigram-based fuzzy matching. Essential for autocomplete ("as you type") and typo tolerance. `CREATE EXTENSION pg_trgm;` + GIN index with `gin_trgm_ops`. |
| **ts_rank / ts_rank_cd** | 🟡 Helpful | Rank search results by relevance. Use `ts_rank_cd` (cover density) for better ranking of adjacent terms. |
| **ts_headline** | 🟡 Helpful | Highlight matched terms in search results for better UX. |
| **similarity()** | 🟡 Helpful | From pg_trgm. Score fuzzy matches for autocomplete ranking. Adjust `pg_trgm.similarity_threshold`. |
| **Materialized Views** | 🟢 Optional | Pre-compute aggregations or denormalized search views. Refresh on spec re-parse. |

**Implementation Pattern**:
```sql
-- Weighted search vector column
ALTER TABLE data_dictionary ADD COLUMN search_vector tsvector;
UPDATE data_dictionary SET search_vector =
  setweight(to_tsvector('english', coalesce(field_name, '')), 'A') ||
  setweight(to_tsvector('english', coalesce(description, '')), 'B') ||
  setweight(to_tsvector('english', coalesce(data_type, '')), 'C');

-- GIN indexes
CREATE INDEX idx_search_vector ON data_dictionary USING GIN(search_vector);
CREATE INDEX idx_field_name_trgm ON data_dictionary USING GIN(field_name gin_trgm_ops);

-- Hybrid search query (FTS + trigram autocomplete)
SELECT *, ts_rank_cd(search_vector, query) AS rank
FROM data_dictionary, plainto_tsquery('english', $1) query
WHERE search_vector @@ query OR field_name % $1
ORDER BY rank DESC, similarity(field_name, $1) DESC
LIMIT 20;
```

**Performance**: GIN indexes + pg_trgm provide sub-10ms search for datasets up to ~10M rows.

**Java Access Patterns**:
```java
// Spring Data JPA — native query for hybrid FTS + trigram search
@Query(value = """
    SELECT *, ts_rank_cd(search_vector, query) AS rank
    FROM data_dictionary, plainto_tsquery('english', :term) query
    WHERE search_vector @@ query OR field_name % :term
    ORDER BY rank DESC, similarity(field_name, :term) DESC
    LIMIT :limit
    """, nativeQuery = true)
List<DataDictionaryEntry> hybridSearch(@Param("term") String term, @Param("limit") int limit);

// JdbcTemplate — for maximum control
jdbcTemplate.query(
    "SELECT *, ts_rank_cd(search_vector, plainto_tsquery('english', ?)) AS rank " +
    "FROM data_dictionary WHERE search_vector @@ plainto_tsquery('english', ?) " +
    "ORDER BY rank DESC LIMIT ?",
    new DataDictionaryRowMapper(), term, term, limit
);
```

---

### 2D. API Framework

| Framework | Relevance | Maven Coordinates | Description |
|-----------|-----------|-------------------|-------------|
| **Spring Boot 3.x + Spring Web MVC** | 🔴 Critical (Recommended) | `org.springframework.boot:spring-boot-starter-web` | Industry-standard Java web framework. Embedded Tomcat, auto-configuration, actuator health checks, OpenAPI integration via springdoc. Excellent for REST APIs with strong typing. |
| **Spring WebFlux** | 🟡 Helpful (Alternative) | `org.springframework.boot:spring-boot-starter-webflux` | Reactive, non-blocking web framework on Netty. Higher throughput under heavy concurrent load. Consider for high-concurrency search endpoints. Adds reactive programming complexity. |
| **Spring Boot Actuator** | 🔴 Critical | `org.springframework.boot:spring-boot-starter-actuator` | Production-ready features: health checks, metrics, info endpoints. Essential for Azure Container Apps health probes and monitoring. |
| **Spring Validation** | 🟡 Helpful | `org.springframework.boot:spring-boot-starter-validation` | Bean validation (JSR 380) with Hibernate Validator. Validate search query parameters, OpenAPI upload payloads, and export request DTOs. |
| **Quarkus** | 🟢 Optional | `io.quarkus:quarkus-bom` | Cloud-native Java framework with fast startup and low memory. Consider if startup time is critical (serverless/scale-to-zero). Smaller ecosystem than Spring. |
| **Micronaut** | 🟢 Optional | `io.micronaut:micronaut-bom` | Compile-time DI framework with fast startup. Alternative to Quarkus for cloud-native workloads. |

**Recommendation**:
- **Primary**: **Spring Boot 3.x + Spring Web MVC** — most mature Java ecosystem, massive community, excellent Azure integration, auto-config for PostgreSQL, Actuator for health probes
- **Alternative**: **Spring WebFlux** if search API needs to handle very high concurrency with reactive backpressure
- Quarkus/Micronaut only if cold start time on Azure Container Apps is a critical concern

---

### 2E. UI / Frontend (Separate Concern)

> **Note**: The frontend is a **separate concern** from the Java backend. The Spring Boot API serves JSON; the frontend consumes it. Choose based on team skills and UX requirements.

| Approach | Relevance | Description |
|----------|-----------|-------------|
| **React SPA (Vite + React)** | 🔴 Critical (Recommended) | Standalone React SPA consuming the Spring Boot REST API. Use Vite for build tooling, shadcn/ui for components, TanStack Table for data grids, TanStack Query for API state. Deploy as static assets on Azure Static Web Apps or CDN. |
| **Next.js** | 🟡 Helpful (Alternative) | React framework with SSR/SSG. Better SEO for public-facing data dictionary. API routes for BFF pattern. Heavier infrastructure (Node.js runtime). |
| **Thymeleaf** | 🟡 Helpful (Alternative) | Server-side templating integrated directly into Spring Boot. Simplest deployment (single artifact). Good for internal tools where SPA complexity isn't justified. Limited interactive UX. |
| **Vaadin** | 🟢 Optional | Full-stack Java framework — write UI in Java, compiles to web components. No JavaScript required. Good for Java-only teams. Heavier runtime, less flexible. |
| **HTMX + Thymeleaf** | 🟡 Helpful | Lightweight interactivity (autocomplete, search-as-you-type) without a JavaScript framework. Pairs with Spring Boot naturally. Increasingly popular for internal tooling. |

**Frontend Library Recommendations (for React SPA approach)**:
- **shadcn/ui** + **Radix UI** — Composable, accessible React components on Tailwind CSS
- **TanStack Table** (`@tanstack/react-table`) — Headless table with sorting, filtering, pagination
- **TanStack Query** (`@tanstack/react-query`) — Server state management with caching, background refetching
- **Tailwind CSS** — Utility-first CSS for rapid UI development

**Recommendation**:
- **For rich search UX**: **React SPA** (Vite) — decoupled from backend, best interactive search experience, modern tooling
- **For simplicity**: **Thymeleaf + HTMX** — single deployable artifact, no separate frontend build, good enough for internal tools
- **For SEO**: **Next.js** — SSR/SSG for indexable data dictionary pages

---

### 2F. Export (CSV/Excel)

| Library | Relevance | Maven Coordinates | Description |
|---------|-----------|-------------------|-------------|
| **Apache POI (XSSF)** | 🔴 Critical (Recommended) | `org.apache.poi:poi-ooxml` | Full-featured Excel (.xlsx) read/write. Rich formatting (fonts, colors, borders, formulas, charts). Streaming API (`SXSSFWorkbook`) for large files to avoid OOM. Industry standard in Java. |
| **OpenCSV** | 🔴 Critical | `com.opencsv:opencsv` | Fast CSV parsing/writing with bean binding. Annotation-driven mapping (`@CsvBindByName`). Handles quoting, escaping, and custom separators. |
| **Apache Commons CSV** | 🟡 Helpful (Alternative) | `org.apache.commons:commons-csv` | Lightweight CSV library from Apache Commons. Simpler API than OpenCSV, supports multiple CSV formats (RFC 4180, Excel, TDF). |
| **FastExcel** | 🟡 Helpful (Alternative) | `org.dhatim:fastexcel` | Lightweight, streaming-only Excel writer. 10x faster than POI for write-only scenarios. No read support. Consider for very large exports. |

**Recommendation**:
- **Primary**: **Apache POI (XSSF/SXSSF)** — handles both read and write, rich formatting, streaming for large exports. Use `SXSSFWorkbook` for exports >10K rows to keep memory bounded.
- **CSV**: **OpenCSV** — annotation-driven bean binding makes it trivial to export JPA entities to CSV
- **Alternative**: **FastExcel** if write performance is critical and rich formatting isn't needed

---

### 2G. Testing

| Library | Relevance | Maven Coordinates | Description |
|---------|-----------|-------------------|-------------|
| **JUnit 5 (Jupiter)** | 🔴 Critical | `org.junit.jupiter:junit-jupiter` | Standard Java testing framework. Parameterized tests, nested test classes, lifecycle hooks. Included via `spring-boot-starter-test`. |
| **Mockito** | 🔴 Critical | `org.mockito:mockito-core` | Mocking framework for unit tests. Mock database repositories, external API calls. Included via `spring-boot-starter-test`. |
| **Spring Boot Test** | 🔴 Critical | `org.springframework.boot:spring-boot-starter-test` | Integration testing with `@SpringBootTest`, `@DataJpaTest`, `@WebMvcTest`. Test slices for focused testing. Includes JUnit 5, Mockito, AssertJ, JSONPath. |
| **Testcontainers** | 🔴 Critical | `org.testcontainers:postgresql` | Run PostgreSQL in Docker for integration tests. Test real FTS queries, Flyway migrations, and search accuracy against actual PostgreSQL with pg_trgm. |
| **AssertJ** | 🟡 Helpful | `org.assertj:assertj-core` | Fluent assertion library. More readable than JUnit assertions. Included via `spring-boot-starter-test`. |
| **REST Assured** | 🟡 Helpful | `io.rest-assured:rest-assured` | Fluent HTTP API testing. Test Spring Boot REST endpoints with expressive DSL. Alternative to MockMvc for integration tests. |
| **Selenium WebDriver** | 🟡 Helpful | `org.seleniumhq.selenium:selenium-java` | Cross-browser E2E testing. Automate the search UI (autocomplete, filters, results display, export downloads). Mature ecosystem. |
| **Playwright for Java** | 🟡 Helpful (Alternative) | `com.microsoft.playwright:playwright` | Modern E2E testing from Microsoft. Auto-wait, Trace Viewer, better DX than Selenium. Newer Java API but rapidly maturing. |
| **JavaFaker** | 🟡 Helpful | `com.github.javafaker:javafaker` | Generate realistic test data. Useful for creating mock OpenAPI specs and data dictionary entries. |
| **ArchUnit** | 🟢 Optional | `com.tngtech.archunit:archunit-junit5` | Architecture testing — enforce layer dependencies, naming conventions, package structure in tests. |

**Testing Strategy**:
- **Unit tests** (JUnit 5 + Mockito): Parser logic, search query building, export formatting, service layer
- **Integration tests** (Spring Boot Test + Testcontainers): PostgreSQL FTS queries, API endpoints with `@WebMvcTest` / MockMvc, repository tests with `@DataJpaTest`
- **E2E tests** (Playwright for Java or Selenium): Search UI autocomplete, filter interactions, CSV/Excel download verification

---

### 2H. Build, Linting & CI/CD

| Tool | Relevance | Description |
|------|-----------|-------------|
| **Maven** | 🔴 Critical (Recommended) | Standard Java build tool. Declarative `pom.xml`, well-understood lifecycle, excellent Spring Boot support via `spring-boot-maven-plugin`. Prefer for conventional projects. |
| **Gradle** | 🟡 Helpful (Alternative) | Groovy/Kotlin DSL build tool. Faster incremental builds, more flexible. Prefer if team has Gradle experience or for multi-module projects. |
| **Checkstyle** | 🟡 Helpful | Code style enforcement. Integrate as Maven/Gradle plugin. Use Google Java Style or Sun conventions. |
| **SpotBugs** | 🟡 Helpful | Static analysis for bug patterns (successor to FindBugs). Catches null pointer dereferences, resource leaks, concurrency issues. |
| **PMD** | 🟡 Helpful | Source code analyzer for common programming flaws. Unused variables, empty catch blocks, suboptimal code. |
| **SonarQube / SonarCloud** | 🟢 Optional | Comprehensive code quality platform. Code smells, security vulnerabilities, coverage tracking. Use SonarCloud for GitHub-integrated analysis. |
| **GitHub Actions** | 🔴 Critical | CI/CD pipeline. Build → Test → Deploy workflow with `azd`. OIDC auth to Azure. |
| **azd (Azure Developer CLI)** | 🔴 Critical | Orchestrates provision + deploy. `azd provision` → `azd deploy`. Integrates with GitHub Actions via `azd pipeline config`. |
| **Docker** | 🟡 Helpful | Containerize the Spring Boot application. Required for Container Apps deployment. Use multi-stage builds with Eclipse Temurin base image. |
| **Jib** | 🟡 Helpful (Alternative) | Build Docker images without a Dockerfile. Maven/Gradle plugin from Google. Faster, reproducible builds, no Docker daemon required. |

**CI/CD Pipeline Pattern**:
```
PR → checkstyle → spotbugs → unit-test → build
merge → integration-test (testcontainers) → azd provision → azd deploy → smoke-test → E2E
```

**GitHub Actions Workflow Example** (Maven):
```yaml
name: CI/CD
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      - run: mvn verify -B --no-transfer-progress
      - run: mvn checkstyle:check spotbugs:check -B
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - run: azd deploy --no-prompt
```

---

## 3. MCP Servers & Extensions

### 3A. MCP Servers

| MCP Server | Relevance | Package/Config | Description |
|------------|-----------|----------------|-------------|
| **GitHub MCP Server** | 🔴 Critical | Built-in to Copilot CLI | PR management, issue tracking, code search, Actions monitoring. Already available in this environment. |
| **PostgreSQL MCP Server** | 🔴 Critical | `@modelcontextprotocol/server-postgres` | Expose PostgreSQL schema + read-only query access to Copilot agents. Enables: schema inspection, query generation, migration validation, search testing. |
| **Filesystem MCP Server** | 🟡 Helpful | `@modelcontextprotocol/server-filesystem` | Allow agents to read/write project files. Useful for code generation agents that modify multiple files. |
| **Custom OpenAPI Parser MCP Server** | 🟡 Helpful | Custom build (Java or Node.js) | Build a custom MCP server that exposes OpenAPI parsing as a tool. Agents could call `parse_openapi(url)` to extract schema info directly. Can be built in Java using the MCP Java SDK (`io.modelcontextprotocol:mcp`). |

**PostgreSQL MCP Server Configuration**:
```json
// .copilot/mcp-config.json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://localhost:5432/datadict"
      ]
    }
  }
}
```

**Custom OpenAPI MCP Server Concept**:
A custom MCP server could expose tools like:
- `parse_spec(url_or_path)` — Parse an OpenAPI spec and return structured field data
- `list_schemas(spec_id)` — List all schemas/components in a parsed spec
- `search_fields(query)` — Search the data dictionary directly
- `export_results(format, query)` — Generate CSV/Excel from search results

This would enable Copilot agents to interact with the data dictionary programmatically.

---

### 3B. Copilot Extensions

| Extension | Relevance | Description |
|-----------|-----------|-------------|
| **Custom Agent: `openapi-parser`** | 🟡 Helpful | A custom Copilot CLI agent (`.github/agents/openapi-parser.md`) specialized in parsing OpenAPI specs, understanding schema composition, and generating data dictionary entries. |
| **Custom Agent: `db-migration`** | 🟡 Helpful | Agent specialized in generating and reviewing Flyway SQL migrations. Understands FTS indexes, tsvector columns, pg_trgm configuration, and Spring Data JPA entity mappings. |
| **Custom Agent: `search-optimizer`** | 🟢 Optional | Agent that analyzes search queries, suggests index improvements, and tunes FTS weights based on usage patterns. |

**Custom Agent Definition Example** (`.github/agents/openapi-parser.md`):
```markdown
---
name: openapi-parser
description: Specialized agent for parsing OpenAPI specifications and generating data dictionary entries
tools:
  - filesystem
  - shell
---

You are an expert at parsing OpenAPI 2.0/3.0/3.1 specifications.
You understand $ref resolution, allOf/oneOf/anyOf composition, 
discriminator mappings, and nested object structures.

When asked to parse a spec:
1. Validate the spec using Redocly CLI
2. Parse and dereference using io.swagger.parser.v3 (swagger-parser)
3. Extract all schemas from components/schemas
4. For each schema, extract fields with: name, type, description, 
   constraints (required, minLength, maxLength, pattern, enum values)
5. Generate SQL INSERT statements or Flyway migration for the data_dictionary table
```

---

## 4. External Tools

### 4A. OpenAPI Management

| Tool | Relevance | Description | How It Fits |
|------|-----------|-------------|-------------|
| **Redocly** | 🟡 Helpful | OpenAPI linting, bundling, documentation generation | Validate specs before parsing. Generate beautiful API docs alongside the data dictionary. CLI integrates into CI/CD. |
| **Stoplight** | 🟢 Optional | API design-first platform with visual editor | Useful if teams are designing APIs before implementing. Data dictionary could consume Stoplight-managed specs. |
| **SwaggerHub** | 🟢 Optional | Swagger/OpenAPI hosting and collaboration | Source for OpenAPI specs if organization uses SwaggerHub as spec registry. Pull specs via API. |
| **Swagger Editor** | 🟢 Optional | Browser-based OpenAPI editor | Quick spec editing/validation during development. |

---

### 4B. Database Management

| Tool | Relevance | Description | How It Fits |
|------|-----------|-------------|-------------|
| **pgAdmin** | 🟡 Helpful | Official PostgreSQL admin GUI | Inspect tables, run FTS queries manually, verify search_vector contents, monitor query performance. |
| **DBeaver** | 🟢 Optional | Universal database client | Alternative to pgAdmin. Supports PostgreSQL with FTS query highlighting. |
| **Spring Boot DevTools** | 🟡 Helpful | Live reload and auto-restart during development | Automatic restart on code changes, remote debugging, H2 console for local dev. `spring-boot-devtools` dependency. |

---

### 4C. API Testing

| Tool | Relevance | Description | How It Fits |
|------|-----------|-------------|-------------|
| **Bruno** | 🟡 Helpful | Open-source API client (Git-friendly, no cloud) | Test search API endpoints locally. Collections stored in Git alongside code. Preferred over Postman for open-source/git-native workflows. |
| **Postman** | 🟢 Optional | Popular API platform | Team collaboration for API testing. Use if team already has Postman workflows. |
| **HTTPie** | 🟢 Optional | CLI HTTP client | Quick ad-hoc API testing from terminal. |
| **Thunder Client** | 🟢 Optional | VS Code extension for API testing | Lightweight alternative, stays in the editor. |

---

## 5. Consolidated SDLC × Tool Matrix

### Legend
- 🔴 **Critical** = Must-have for this phase/feature
- 🟡 **Helpful** = Significantly improves quality/speed
- 🟢 **Optional** = Nice-to-have, consider based on needs

| Tool/Skill | Requirements | Architecture | Infrastructure | Implementation | Testing | Deployment | Operations |
|------------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Copilot CLI Skills** | | | | | | | |
| `azure-prepare` | | 🟡 | 🔴 | | | | |
| `azure-deploy` | | | | | | 🔴 | |
| `azure-postgres` | | 🟡 | 🔴 | | | | |
| `azure-ai` | | 🟡 | 🟢 | 🟢 | | | |
| `azure-storage` | | | 🟡 | 🟡 | | | |
| `azure-observability` | | | | | | | 🟡 |
| `azure-diagnostics` | | | | | | | 🟡 |
| `azure-rbac` | | | 🟡 | | | | |
| `azure-compliance` | | | 🟢 | | | | 🟢 |
| `azure-resource-visualizer` | | 🟢 | | | | | 🟢 |
| `azure-cost-optimization` | | | | | | | 🟢 |
| `azure-aigateway` | | | 🟢 | | | | |
| `appinsights-instrumentation` | | | | 🟡 | | | 🟡 |
| `code-review` agent | | | | 🔴 | 🟡 | | |
| `testing-agent` | | | | | 🟡 | | |
| `architecture-reviewer` | | 🟡 | | | | | |
| **Java/Spring Boot Libraries** | | | | | | | |
| `swagger-parser` | | | | 🔴 | | | |
| `openapi-generator` | | | | 🟡 | | | |
| `springdoc-openapi` | | | | 🟡 | | 🟡 | |
| Spring Data JPA + Hibernate | | 🔴 | | 🔴 | | | |
| Flyway | | 🔴 | | 🔴 | | 🟡 | |
| JdbcTemplate / jOOQ | | | | 🟡 | | | |
| PostgreSQL FTS (tsvector/pg_trgm) | | 🔴 | | 🔴 | | | |
| Spring Boot 3.x + Web MVC | | 🟡 | | 🔴 | | | |
| Spring Boot Actuator | | | | 🟡 | | | 🟡 |
| React SPA / Thymeleaf | | 🟡 | | 🔴 | | | |
| Apache POI (XSSF) | | | | 🔴 | | | |
| OpenCSV | | | | 🟡 | | | |
| JUnit 5 + Mockito | | | | | 🔴 | | |
| Testcontainers | | | | | 🔴 | | |
| Spring Boot Test | | | | | 🔴 | | |
| Playwright / Selenium | | | | | 🟡 | | |
| REST Assured | | | | | 🟡 | | |
| Checkstyle / SpotBugs / PMD | | | | 🟡 | 🟡 | 🟡 | |
| Maven / Gradle | | | | 🔴 | 🔴 | 🔴 | |
| GitHub Actions | | | | | | 🔴 | |
| `azd` CLI | | | 🔴 | | | 🔴 | |
| Docker / Jib | | | | 🟡 | | 🟡 | |
| **MCP Servers** | | | | | | | |
| GitHub MCP (built-in) | 🟡 | | | 🟡 | | 🟡 | |
| PostgreSQL MCP | | 🟡 | | 🔴 | 🟡 | | |
| Custom OpenAPI MCP | | | | 🟡 | | | |
| **External Tools** | | | | | | | |
| Redocly | | | | 🟡 | | 🟡 | |
| pgAdmin / DBeaver | | | | 🟡 | 🟡 | | 🟡 |
| Bruno / Postman | | | | 🟡 | 🟡 | | |

---

## 6. Recommended Stack (Opinionated)

### Core Technology Choices

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA DICTIONARY STACK (Java)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PARSE LAYER          STORE LAYER           SERVE LAYER         │
│  ─────────────        ───────────           ───────────         │
│  swagger-parser  →    Spring Data JPA →     Spring Boot 3.x     │
│  openapi-generator    Hibernate / JDBC      (Web MVC + REST)    │
│  Redocly CLI          PostgreSQL 16+        springdoc-openapi   │
│                       tsvector/pg_trgm      React SPA (Vite)    │
│                       GIN indexes            or Thymeleaf+HTMX  │
│                       Flyway migrations                         │
│                                                                 │
│  EXPORT LAYER         TEST LAYER            DEPLOY LAYER        │
│  ────────────         ──────────            ────────────        │
│  Apache POI (xlsx)    JUnit 5 (unit)        GitHub Actions      │
│  OpenCSV (csv)        Mockito (mocks)       Maven (build)       │
│                       Testcontainers (IT)   Docker / Jib        │
│                       Playwright (E2E)      azd CLI             │
│                       Spring Boot Test      Azure Container Apps│
│                       REST Assured (API)                        │
│                                                                 │
│  QUALITY              INFRA (Azure)         MCP SERVERS         │
│  ───────              ──────────────        ───────────         │
│  Checkstyle           PostgreSQL Flex       GitHub MCP          │
│  SpotBugs             Container Apps        PostgreSQL MCP      │
│  PMD                  Blob Storage          Custom OpenAPI MCP  │
│  SonarCloud           Entra ID auth                             │
│                                                                 │
│  MONITORING           COPILOT SKILLS        AGENTS              │
│  ──────────           ──────────────        ──────              │
│  App Insights         azure-prepare         code-review         │
│  Azure Monitor        azure-deploy          testing-agent       │
│  Log Analytics        azure-postgres        architecture-reviewer│
│                       azure-observability   openapi-parser (cust)│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Why These Choices?

| Decision | Rationale |
|----------|-----------|
| **Spring Data JPA + JdbcTemplate over jOOQ** | JPA provides the standard repository pattern for CRUD; JdbcTemplate gives raw SQL escape hatch for FTS queries. jOOQ is excellent but adds code-gen build complexity. JPA + JdbcTemplate covers 95% of cases with zero extra build steps. |
| **Spring Boot 3.x over Quarkus/Micronaut** | Largest ecosystem, most documentation, best Azure integration (Azure Spring Apps, App Insights Java agent). GraalVM native-image support if startup time becomes critical. Enterprise teams know Spring. |
| **Flyway over Liquibase** | SQL-based migrations are clearer for FTS setup (`CREATE EXTENSION pg_trgm`, GIN indexes, tsvector triggers). Simpler mental model. First-class Spring Boot auto-configuration. |
| **React SPA over Thymeleaf** | Better interactive search UX (autocomplete, instant filtering, client-side sorting). Decoupled from backend — can deploy to CDN. Thymeleaf is viable for simpler internal tools. |
| **JUnit 5 + Testcontainers over mocks-only** | Real PostgreSQL FTS queries cannot be adequately tested with mocks. Testcontainers gives real pg_trgm + tsvector behavior in CI. Fast startup with reusable containers. |
| **Apache POI over FastExcel** | Handles both read and write with rich formatting. Streaming API (SXSSF) keeps memory bounded. Industry standard — every Java team knows POI. |
| **PostgreSQL FTS over Elasticsearch** | Eliminates a separate service. FTS + pg_trgm handles data dictionary scale perfectly. Simpler architecture. Lower cost. One database for storage AND search. |
| **Maven over Gradle** | More predictable for CI/CD, declarative XML is easier for Copilot agents to reason about, Spring Boot's default. Gradle is fine if team prefers it. |
| **Azure Container Apps** | Serverless containers with scale-to-zero. Perfect for a data dictionary service that may have bursty usage. Simpler than AKS. Java 21 runtime support. |

### Key Architectural Decisions

1. **PostgreSQL as both storage AND search engine** — eliminates Elasticsearch/Azure AI Search complexity
2. **Spring Data JPA + JdbcTemplate for hybrid data access** — JPA for CRUD, raw SQL for FTS queries
3. **Flyway for migrations** — SQL-native migrations ideal for PostgreSQL extensions and FTS indexes
4. **Decoupled frontend** — React SPA (or Thymeleaf) consuming Spring Boot REST API; independently deployable
5. **MCP-first agent integration** — PostgreSQL MCP server enables agents to query/validate data dictionary directly
6. **Weighted search vectors** — Field names weighted 'A', descriptions 'B', types 'C' for intuitive ranking
7. **Hybrid search** — tsvector for keyword relevance + pg_trgm for autocomplete/fuzzy matching
8. **Java 21 LTS** — virtual threads (Project Loom) for high-concurrency search, pattern matching, records for DTOs

---

## Appendix: Project Setup Quick Reference

### Maven (`pom.xml` dependencies)

```xml
<properties>
    <java.version>21</java.version>
    <spring-boot.version>3.3.0</spring-boot.version>
    <testcontainers.version>1.19.8</testcontainers.version>
</properties>

<dependencies>
    <!-- Core: Web + JPA + PostgreSQL -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- OpenAPI Parsing -->
    <dependency>
        <groupId>io.swagger.parser.v3</groupId>
        <artifactId>swagger-parser</artifactId>
        <version>2.1.22</version>
    </dependency>

    <!-- API Documentation -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.5.0</version>
    </dependency>

    <!-- Export -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml</artifactId>
        <version>5.2.5</version>
    </dependency>
    <dependency>
        <groupId>com.opencsv</groupId>
        <artifactId>opencsv</artifactId>
        <version>5.9</version>
    </dependency>

    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <version>${testcontainers.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>${testcontainers.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>com.microsoft.playwright</groupId>
        <artifactId>playwright</artifactId>
        <version>1.44.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        <!-- Checkstyle -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-checkstyle-plugin</artifactId>
            <version>3.3.1</version>
        </plugin>
        <!-- SpotBugs -->
        <plugin>
            <groupId>com.github.spotbugs</groupId>
            <artifactId>spotbugs-maven-plugin</artifactId>
            <version>4.8.5.0</version>
        </plugin>
    </plugins>
</build>
```

### Scaffold Project

```bash
# Generate Spring Boot project (via Spring Initializr CLI or curl)
curl https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.3.0 \
  -d baseDir=datadict \
  -d groupId=com.example \
  -d artifactId=datadict \
  -d javaVersion=21 \
  -d dependencies=web,data-jpa,postgresql,flyway,actuator,validation \
  -o datadict.zip && unzip datadict.zip

# MCP Server (PostgreSQL)
npx @modelcontextprotocol/server-postgres postgresql://localhost/datadict

# Azure
winget install Microsoft.Azd   # or: brew install azd (macOS)

# Frontend (if using React SPA, in separate directory — requires Node.js for frontend tooling)
# npm create vite@latest frontend -- --template react-ts
# cd frontend && npx shadcn@latest init
# npm install @tanstack/react-table @tanstack/react-query
# NOTE: Frontend is a separate concern. If using Thymeleaf/Vaadin, no separate setup needed.
```
