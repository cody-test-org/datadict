---
name: aws-patterns
description: >-
  Domain knowledge for deploying Java/Spring Boot applications on AWS. Covers RDS PostgreSQL,
  ECS/Fargate, Secrets Manager, CloudWatch, ALB, ECR, and CloudFormation/CDK patterns.
  Use when the target cloud provider is AWS.
---

# AWS Patterns Skill

## Purpose

This skill provides domain knowledge for deploying and operating Java 21+ / Spring Boot 3.x
applications on Amazon Web Services. It covers infrastructure provisioning, container
deployment, secrets management, observability, and CI/CD patterns specific to the AWS
ecosystem. Use this skill when the customer's target environment is AWS.

## AWS RDS PostgreSQL

### Provisioning Patterns

Use RDS PostgreSQL 16+ for managed database hosting. Key configuration decisions:

```yaml
# Typical RDS instance configuration
Engine: postgres
EngineVersion: "16.4"
DBInstanceClass: db.r6g.large        # Graviton for cost efficiency
AllocatedStorage: 100
StorageType: gp3
MultiAZ: true                        # Production always multi-AZ
DeletionProtection: true
BackupRetentionPeriod: 14
StorageEncrypted: true
```

### Parameter Groups

Create a custom parameter group to enable required extensions and tune performance:

```
shared_preload_libraries: pg_stat_statements
log_min_duration_statement: 1000     # Log slow queries > 1s
work_mem: 256MB
maintenance_work_mem: 512MB
effective_cache_size: 4GB
random_page_cost: 1.1                # SSD-backed storage
```

### PostgreSQL Extensions

Enable extensions via parameter groups and SQL:

```sql
-- Required for full-text search and UUID generation
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

> **Note:** RDS supports most common extensions but not all. Check the
> [RDS PostgreSQL extensions list](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html#PostgreSQL.Concepts.General.FeatureSupport.Extensions)
> before depending on an extension.

### SSL Configuration

Always enforce SSL connections in production:

```yaml
# application-aws.yml
spring:
  datasource:
    url: jdbc:postgresql://${RDS_HOSTNAME}:5432/${RDS_DB_NAME}?sslmode=require&sslrootcert=/app/certs/rds-combined-ca-bundle.pem
```

Download the RDS CA bundle into your container image or mount it as a volume.

### IAM Database Authentication vs Password Auth

**IAM Auth** — recommended for ECS tasks with IAM roles:

```java
// Generate auth token programmatically
RdsUtilities rdsUtilities = RdsUtilities.builder()
    .region(Region.US_EAST_1)
    .build();

String authToken = rdsUtilities.generateAuthenticationToken(
    GenerateAuthenticationTokenRequest.builder()
        .hostname(rdsHostname)
        .port(5432)
        .username("app_user")
        .build()
);
```

**Password Auth** — simpler, use with Secrets Manager rotation:

```yaml
spring:
  datasource:
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

### Connection Pooling with RDS Proxy

Use RDS Proxy to pool connections and handle IAM auth transparently:

```yaml
# Point the application at the RDS Proxy endpoint instead of the RDS instance
spring:
  datasource:
    url: jdbc:postgresql://${RDS_PROXY_ENDPOINT}:5432/${RDS_DB_NAME}
    hikari:
      maximum-pool-size: 10          # Keep low — Proxy manages the real pool
      minimum-idle: 2
      connection-timeout: 5000
```

RDS Proxy benefits:
- Connection multiplexing (fewer DB connections needed)
- Automatic failover handling for Multi-AZ
- IAM authentication at the proxy layer
- Graceful handling of database failovers without application restarts

## ECS/Fargate Deployment

### Task Definition

```json
{
  "family": "datadict-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/datadictTaskRole",
  "containerDefinitions": [
    {
      "name": "datadict",
      "image": "ACCOUNT.dkr.ecr.REGION.amazonaws.com/datadict:latest",
      "portMappings": [
        { "containerPort": 8080, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "SPRING_PROFILES_ACTIVE", "value": "aws" }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:datadict/db-password"
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 120
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/datadict",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### ECS Service Configuration

```yaml
Service:
  LaunchType: FARGATE
  DesiredCount: 2
  DeploymentConfiguration:
    MaximumPercent: 200
    MinimumHealthyPercent: 100
    DeploymentCircuitBreaker:
      Enable: true
      Rollback: true
  NetworkConfiguration:
    AwsvpcConfiguration:
      Subnets: [subnet-private-1, subnet-private-2]
      SecurityGroups: [sg-ecs-tasks]
      AssignPublicIp: DISABLED
```

### ALB Target Group and Health Checks

```yaml
TargetGroup:
  TargetType: ip
  Protocol: HTTP
  Port: 8080
  HealthCheckPath: /actuator/health
  HealthCheckIntervalSeconds: 30
  HealthyThresholdCount: 2
  UnhealthyThresholdCount: 3
  HealthCheckTimeoutSeconds: 10
  Matcher:
    HttpCode: "200"
```

### Spring Boot Actuator Integration

Configure Actuator endpoints for ALB and ECS health checks:

```yaml
# application-aws.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
  health:
    db:
      enabled: true
    diskspace:
      enabled: false           # Not meaningful on Fargate
```

## ECR (Elastic Container Registry)

### Container Registry Setup

```bash
# Create repository
aws ecr create-repository \
  --repository-name datadict \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin ACCOUNT.dkr.ecr.us-east-1.amazonaws.com
```

### Lifecycle Policies

Keep only recent images to control costs:

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Expire untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }
  ]
}
```

### Image Scanning

ECR supports both basic and enhanced scanning (via Amazon Inspector):

- **Basic scanning:** Free, checks against CVE databases on push
- **Enhanced scanning:** Continuous scanning, OS and language package vulnerabilities
- Always enable `scanOnPush: true` for production repositories

## AWS Secrets Manager

### Spring Cloud AWS Integration

Add the dependency:

```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-secrets-manager</artifactId>
</dependency>
```

### Accessing Secrets in Spring Boot

Use the `spring.config.import` mechanism (Spring Boot 3.x preferred approach):

```yaml
# application-aws.yml
spring:
  config:
    import: aws-secretsmanager:datadict/db-credentials
  datasource:
    username: ${db.username}
    password: ${db.password}
```

The secret `datadict/db-credentials` should be a JSON object:

```json
{
  "db.username": "app_user",
  "db.password": "secret-password-here",
  "db.host": "datadict-db.cluster-xxx.us-east-1.rds.amazonaws.com"
}
```

### Secret Rotation

Configure automatic rotation for database credentials:

```yaml
# CloudFormation snippet
SecretRotation:
  Type: AWS::SecretsManager::RotationSchedule
  Properties:
    SecretId: !Ref DatabaseSecret
    RotationLambdaARN: !GetAtt RotationLambda.Arn
    RotationRules:
      AutomaticallyAfterDays: 30
```

Ensure the application handles credential refresh gracefully — HikariCP will automatically
pick up new credentials on connection creation if using Secrets Manager integration.

## CloudWatch

### Logging

ECS tasks log to CloudWatch Logs automatically via the `awslogs` driver. Structure logs
as JSON for easier querying:

```java
// logback-spring.xml for JSON structured logging
<configuration>
    <springProfile name="aws">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
        </appender>
        <root level="INFO">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>
</configuration>
```

### CloudWatch Metrics via Micrometer

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-cloudwatch2</artifactId>
</dependency>
```

```yaml
# application-aws.yml
management:
  cloudwatch:
    metrics:
      export:
        namespace: DataDict
        enabled: true
        step: 60s
```

### CloudWatch Alarms

Set up alarms for critical metrics:

```yaml
# CloudFormation
HighCPUAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: datadict-high-cpu
    MetricName: CPUUtilization
    Namespace: AWS/ECS
    Statistic: Average
    Period: 300
    EvaluationPeriods: 2
    Threshold: 80
    ComparisonOperator: GreaterThanThreshold
    Dimensions:
      - Name: ServiceName
        Value: datadict
      - Name: ClusterName
        Value: !Ref ECSCluster
    AlarmActions:
      - !Ref AlertSNSTopic

ErrorRateAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: datadict-error-rate
    MetricName: 5xxErrors
    Namespace: DataDict
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 1
    Threshold: 10
    ComparisonOperator: GreaterThanThreshold
    AlarmActions:
      - !Ref AlertSNSTopic
```

### Spring Boot Actuator → CloudWatch Metrics

Micrometer automatically publishes Actuator metrics to CloudWatch. Key metrics to monitor:

- `jvm.memory.used` — heap and non-heap memory
- `http.server.requests` — request count, latency percentiles, error rates
- `hikaricp.connections.active` — database connection pool utilization
- `spring.data.repository.*` — JPA query performance

## CloudFormation / CDK Patterns

### Java CDK Stack Structure

```java
public class DataDictStack extends Stack {

    public DataDictStack(final Construct scope, final String id, final StackProps props) {
        super(scope, id, props);

        // VPC
        Vpc vpc = Vpc.Builder.create(this, "DataDictVpc")
            .maxAzs(2)
            .natGateways(1)
            .build();

        // RDS
        DatabaseInstance database = DatabaseInstance.Builder.create(this, "DataDictDb")
            .engine(DatabaseInstanceEngine.postgres(
                PostgresInstanceEngineProps.builder()
                    .version(PostgresEngineVersion.VER_16_4)
                    .build()))
            .instanceType(InstanceType.of(InstanceClass.R6G, InstanceSize.LARGE))
            .vpc(vpc)
            .vpcSubnets(SubnetSelection.builder().subnetType(SubnetType.PRIVATE_WITH_EGRESS).build())
            .multiAz(true)
            .allocatedStorage(100)
            .storageEncrypted(true)
            .databaseName("datadict")
            .credentials(Credentials.fromGeneratedSecret("app_user"))
            .build();

        // ECS Cluster
        Cluster cluster = Cluster.Builder.create(this, "DataDictCluster")
            .vpc(vpc)
            .containerInsights(true)
            .build();

        // Fargate Service with ALB
        ApplicationLoadBalancedFargateService service =
            ApplicationLoadBalancedFargateService.Builder.create(this, "DataDictService")
                .cluster(cluster)
                .cpu(1024)
                .memoryLimitMiB(2048)
                .desiredCount(2)
                .taskImageOptions(ApplicationLoadBalancedTaskImageOptions.builder()
                    .image(ContainerImage.fromEcrRepository(repository, "latest"))
                    .containerPort(8080)
                    .environment(Map.of("SPRING_PROFILES_ACTIVE", "aws"))
                    .secrets(Map.of(
                        "DB_PASSWORD", Secret.fromSecretsManager(database.getSecret(), "password")
                    ))
                    .build())
                .build();

        // Health check
        service.getTargetGroup().configureHealthCheck(HealthCheck.builder()
            .path("/actuator/health")
            .healthyHttpCodes("200")
            .interval(Duration.seconds(30))
            .build());
    }
}
```

### Stack Structure Recommendations

```
infra/
├── src/main/java/com/example/infra/
│   ├── InfraApp.java               # CDK app entry point
│   ├── NetworkStack.java           # VPC, subnets, NAT
│   ├── DatabaseStack.java          # RDS, security groups
│   ├── ContainerStack.java         # ECR, ECS, Fargate, ALB
│   └── ObservabilityStack.java     # CloudWatch dashboards, alarms
├── cdk.json
└── pom.xml
```

## GitHub Actions for AWS

### OIDC Authentication (Recommended)

Use OIDC federation — no long-lived AWS credentials in GitHub:

```yaml
# .github/workflows/deploy-aws.yml
name: Deploy to AWS
on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/github-actions-deploy
          aws-region: us-east-1

      - name: Login to ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        env:
          ECR_REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/datadict:$IMAGE_TAG .
          docker push $ECR_REGISTRY/datadict:$IMAGE_TAG

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: task-definition.json
          service: datadict
          cluster: datadict-cluster
          wait-for-service-stability: true
```

### IAM Role Trust Policy for GitHub OIDC

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:ORG/REPO:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

## Spring Cloud AWS Integration Patterns

### Dependency BOM

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.awspring.cloud</groupId>
            <artifactId>spring-cloud-aws-dependencies</artifactId>
            <version>3.2.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Common Starters

```xml
<!-- Secrets Manager config import -->
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-secrets-manager</artifactId>
</dependency>

<!-- S3 for file storage -->
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-s3</artifactId>
</dependency>

<!-- SQS for async messaging -->
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sqs</artifactId>
</dependency>
```

### Spring Profile for AWS

```yaml
# application-aws.yml
spring:
  config:
    import: aws-secretsmanager:datadict/credentials
  datasource:
    url: jdbc:postgresql://${db.host}:5432/datadict?sslmode=require
    username: ${db.username}
    password: ${db.password}
    hikari:
      maximum-pool-size: 15
      minimum-idle: 5
  threads:
    virtual:
      enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      probes:
        enabled: true
  cloudwatch:
    metrics:
      export:
        namespace: DataDict
        enabled: true
        step: 60s

logging:
  level:
    io.awspring: INFO
    software.amazon.awssdk: WARN
```

## Key Differences from Azure

| Concern | Azure | AWS |
|---|---|---|
| Container orchestration | Container Apps | ECS/Fargate |
| Container registry | Azure Container Registry (ACR) | Elastic Container Registry (ECR) |
| Managed PostgreSQL | Azure Database for PostgreSQL | RDS PostgreSQL |
| Secrets | Azure Key Vault | AWS Secrets Manager |
| Observability | Application Insights + Azure Monitor | CloudWatch + X-Ray |
| Load balancer | Built into Container Apps | Application Load Balancer (ALB) |
| IaC | Bicep / ARM Templates | CloudFormation / CDK |
| CI/CD auth | Azure OIDC (`azure/login`) | AWS OIDC (`aws-actions/configure-aws-credentials`) |
| Spring integration | Spring Cloud Azure | Spring Cloud AWS (io.awspring) |
| Config import | `azure-keyvault:` | `aws-secretsmanager:` |
| Connection pooling | Built-in to Flexible Server | RDS Proxy (separate resource) |
| Metrics export | `micrometer-registry-azure-monitor` | `micrometer-registry-cloudwatch2` |

### Migration Considerations

When adapting from Azure patterns to AWS:

1. **Replace Bicep with CDK or CloudFormation** — CDK in Java keeps the IaC in the same language
2. **Replace Azure Key Vault references** with Secrets Manager `spring.config.import`
3. **Replace Container Apps config** with ECS task definitions and service configs
4. **Replace Application Insights** with CloudWatch metrics + structured logging
5. **Replace `azure/login` GitHub Action** with `aws-actions/configure-aws-credentials`
6. **Add ALB configuration** — Container Apps includes a load balancer; ECS requires explicit ALB setup
7. **Add RDS Proxy** if connection pooling at the infrastructure level is needed
