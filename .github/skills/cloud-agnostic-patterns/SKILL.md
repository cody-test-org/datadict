---
name: cloud-agnostic-patterns
description: >-
  Domain knowledge for building cloud-agnostic Java/Spring Boot applications. Covers
  abstraction patterns, environment-based configuration, Docker-first deployment, and
  portable infrastructure strategies. Use as the default cloud skill when the provider
  is unknown or when designing for portability across Azure, AWS, and GCP.
---

# Cloud-Agnostic Patterns Skill

## Purpose

This skill provides domain knowledge for building Java 21+ / Spring Boot 3.x applications
that run on any cloud provider without code changes. It covers abstraction strategies,
portable configuration, Docker-first deployment, and universal observability patterns.
Use this skill as the default when the target cloud provider is unknown or when designing
for multi-cloud portability.

## Spring Profiles for Environment Abstraction

Use Spring profiles to isolate all cloud-specific configuration into separate files.
Application code should never reference a specific cloud provider directly.

### Profile File Structure

```
src/main/resources/
├── application.yml              # Shared defaults (all environments)
├── application-local.yml        # Local development (Docker Compose)
├── application-azure.yml        # Azure-specific config
├── application-aws.yml          # AWS-specific config
├── application-gcp.yml          # GCP-specific config
└── application-test.yml         # Test environment (Testcontainers)
```

### Shared Configuration (application.yml)

```yaml
spring:
  application:
    name: datadict
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        default_batch_fetch_size: 20
  flyway:
    enabled: true
    locations: classpath:db/migration

management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics
  endpoint:
    health:
      probes:
        enabled: true

server:
  port: 8080
```

### Cloud-Specific Overrides

Each cloud profile overrides only what differs — datasource URLs, secrets config, and
metrics exporters:

```yaml
# application-local.yml — no cloud dependencies
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/datadict
    username: dev_user
    password: dev_password
  jpa:
    show-sql: true
```

Activate via environment variable (works on every platform):

```bash
SPRING_PROFILES_ACTIVE=aws      # or azure, gcp, local
```

### Code Rule

**Never import cloud-specific classes in application code.** Cloud SDKs belong only in
configuration classes guarded by `@Profile`:

```java
@Configuration
@Profile("aws")
public class AwsConfig {
    // AWS-specific beans only instantiated when profile is active
}
```

## Docker-First Deployment

Docker is the universal deployment unit — it works with every orchestrator on every cloud.

### Multi-Stage Dockerfile

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /workspace
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -B
COPY src src
RUN ./mvnw package -DskipTests -B

# Runtime stage
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --from=build /workspace/target/*.jar app.jar
USER app

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=120s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Docker Compose for Local Development

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: local
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/datadict
      SPRING_DATASOURCE_USERNAME: dev_user
      SPRING_DATASOURCE_PASSWORD: dev_password
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: datadict
      POSTGRES_USER: dev_user
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev_user -d datadict"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

### Deployment Portability

The same Docker image deploys to any orchestrator:

| Local | Azure | AWS | GCP |
|---|---|---|---|
| Docker Compose | Container Apps | ECS/Fargate | Cloud Run |
| `docker compose up` | `az containerapp up` | `aws ecs deploy` | `gcloud run deploy` |

## Database Portability

### Use Standard PostgreSQL Only

Stick to standard PostgreSQL features that work identically across all managed services:

```sql
-- ✅ Portable: standard PostgreSQL
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- ✅ Portable: standard full-text search
CREATE INDEX idx_fields_search ON api_fields USING gin(to_tsvector('english', description));

-- ❌ Avoid: provider-specific features
-- Aurora Serverless Data API, Cloud SQL–specific flags, etc.
```

### Connection String via Environment Variables

Never hardcode connection details. Use environment variables that every cloud can inject:

```yaml
spring:
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/datadict}
    username: ${DATABASE_USERNAME:dev_user}
    password: ${DATABASE_PASSWORD:dev_password}
    hikari:
      maximum-pool-size: ${DB_POOL_SIZE:10}
      minimum-idle: ${DB_POOL_MIN:2}
```

### Migration Safety

Flyway migrations should use only ANSI SQL and standard PostgreSQL syntax:

```sql
-- V1__create_tables.sql
CREATE TABLE api_fields (
    id              BIGSERIAL PRIMARY KEY,
    field_name      VARCHAR(255) NOT NULL,
    field_type      VARCHAR(100) NOT NULL,
    description     TEXT,
    schema_name     VARCHAR(255) NOT NULL,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
```

## Secrets Management Abstraction

### Strategy: Environment Variables as the Universal Interface

Every cloud injects secrets differently, but they all support environment variables.
Design the application to read secrets from environment variables, and let the platform
handle the injection:

```yaml
# application.yml — cloud-agnostic secret consumption
spring:
  datasource:
    password: ${DATABASE_PASSWORD}

app:
  api-key: ${API_KEY}
  jwt-secret: ${JWT_SECRET}
```

### Provider-Specific Injection (Outside Application Code)

| Provider | Mechanism |
|---|---|
| Local | `.env` file or Docker Compose `environment:` |
| Azure | Key Vault → Container Apps secrets → env vars |
| AWS | Secrets Manager → ECS task definition `secrets:` → env vars |
| GCP | Secret Manager → Cloud Run `--set-secrets` → env vars |
| Kubernetes | Kubernetes Secrets → pod `envFrom:` → env vars |

### Spring Cloud Config (Optional)

For advanced scenarios, use Spring Cloud Config Server as a provider-neutral config source:

```yaml
spring:
  config:
    import: configserver:http://config-server:8888
```

This decouples secret management entirely from the cloud provider.

## Observability Portability

### OpenTelemetry as the Universal Standard

Use OpenTelemetry (OTel) for traces, metrics, and logs. It exports to any backend:

```xml
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

```yaml
# application.yml
otel:
  exporter:
    otlp:
      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4318}
  service:
    name: datadict
```

### Exporter by Provider

| Provider | OTel Exporter Target |
|---|---|
| Local | Jaeger / Grafana LGTM stack |
| Azure | Azure Monitor OpenTelemetry Exporter |
| AWS | AWS Distro for OpenTelemetry (ADOT) → CloudWatch / X-Ray |
| GCP | Google Cloud Trace + Cloud Monitoring |
| Datadog | Datadog OTel Collector |

### Micrometer as the Metrics Abstraction

Micrometer works with any registry — swap the dependency per environment:

```xml
<!-- Local / Prometheus -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Swap to `micrometer-registry-cloudwatch2`, `micrometer-registry-azure-monitor`, or
`micrometer-registry-datadog` via Maven profiles or runtime classpath.

## CI/CD Portability

### GitHub Actions as the Universal CI Platform

GitHub Actions works with every cloud. Keep cloud-specific logic isolated to the deploy job:

```yaml
# .github/workflows/ci.yml — portable build + test
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
          cache: maven
      - run: ./mvnw verify -B

  # Cloud-specific deploy job — only part that changes per provider
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    uses: ./.github/workflows/deploy-${{ vars.CLOUD_PROVIDER }}.yml
    secrets: inherit
```

### Reusable Workflow Pattern

Create provider-specific deploy workflows that the main CI calls:

```
.github/workflows/
├── ci.yml                    # Universal: build, test, lint
├── deploy-azure.yml          # Azure-specific deploy
├── deploy-aws.yml            # AWS-specific deploy
└── deploy-gcp.yml            # GCP-specific deploy
```

## Health Checks

### Spring Actuator — Universal Health Endpoints

Spring Actuator health endpoints work identically on every platform:

```yaml
management:
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  health:
    db:
      enabled: true
```

Exposed endpoints:

| Endpoint | Purpose | Used By |
|---|---|---|
| `/actuator/health` | Overall health | Load balancers, dashboards |
| `/actuator/health/liveness` | Is the app alive? | Kubernetes liveness probe, ECS health check |
| `/actuator/health/readiness` | Can it accept traffic? | Kubernetes readiness probe, ALB health check |

Every orchestrator can use these endpoints without application changes:

```yaml
# Kubernetes
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8080 }
```

## Configuration Pattern — 12-Factor App

Follow the [Twelve-Factor App](https://12factor.net/) methodology for portability:

### All Config via Environment Variables

```java
@ConfigurationProperties(prefix = "app")
public record AppProperties(
    String apiKey,
    Duration cacheTtl,
    int maxPageSize,
    boolean featureSearchEnabled
) {}
```

```yaml
app:
  api-key: ${API_KEY:default-dev-key}
  cache-ttl: ${CACHE_TTL:PT5M}
  max-page-size: ${MAX_PAGE_SIZE:100}
  feature-search-enabled: ${FEATURE_SEARCH_ENABLED:true}
```

### No Hardcoded Cloud References in Code

```java
// ✅ Good — reads from config, works anywhere
@Value("${app.storage.base-url}")
private String storageBaseUrl;

// ❌ Bad — hardcoded to a specific cloud
private String storageBaseUrl = "https://myaccount.blob.core.windows.net";
```

## IaC Options by Provider

| Provider | Primary IaC | Alternative |
|---|---|---|
| Azure | Bicep | Terraform |
| AWS | CloudFormation / CDK | Terraform |
| GCP | Deployment Manager | Terraform |
| Any | **Terraform** | Pulumi |

### Terraform as the Portable Option

When multi-cloud is a hard requirement, use Terraform for all providers:

```hcl
# main.tf — provider-agnostic structure
module "database" {
  source = "./modules/${var.cloud_provider}/database"
  # ...
}

module "container_service" {
  source = "./modules/${var.cloud_provider}/container"
  # ...
}
```

```
infra/
├── modules/
│   ├── azure/
│   │   ├── database/       # Azure Database for PostgreSQL
│   │   └── container/      # Container Apps
│   ├── aws/
│   │   ├── database/       # RDS PostgreSQL
│   │   └── container/      # ECS Fargate
│   └── gcp/
│       ├── database/       # Cloud SQL
│       └── container/      # Cloud Run
├── main.tf
└── variables.tf
```

## Decision Matrix: Cloud-Specific vs Cloud-Agnostic

| Factor | Use Cloud-Specific | Use Cloud-Agnostic |
|---|---|---|
| Single cloud, long-term | ✅ Optimize for that cloud | |
| Multi-cloud requirement | | ✅ Portability is critical |
| Startup / MVP | ✅ Ship fast, optimize later | |
| Enterprise with cloud strategy | ✅ Leverage managed services | |
| Unknown target cloud | | ✅ Default to portable |
| Cost-sensitive | ✅ Native services often cheaper | |
| Team expertise | ✅ Use what team knows | |
| Regulatory / data sovereignty | | ✅ May need to switch providers |
| Performance-critical | ✅ Native integrations are faster | |
| Vendor lock-in concern | | ✅ Minimize switching costs |

### Recommended Default Approach

1. **Write cloud-agnostic application code** (12-factor, env vars, no cloud imports in business logic)
2. **Use cloud-specific infrastructure** (native IaC, managed services) — this is where cloud value lives
3. **Isolate cloud coupling to configuration and deployment** — Spring profiles, Dockerfiles, CI/CD workflows
4. **Use OpenTelemetry and Micrometer** — swap exporters without code changes
5. **Default to this skill** when the cloud provider is not yet decided; switch to `aws-patterns` or the Azure patterns when the provider is confirmed
