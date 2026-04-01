---
name: java-spring-patterns
description: >-
  Domain knowledge for Java 21+ and Spring Boot 3.x development patterns. Covers
  constructor injection, records for DTOs, JPA entity patterns, @ControllerAdvice,
  Spring profiles, Flyway migrations, and testing with Testcontainers. Use as the
  primary coding reference for all Java/Spring Boot code generation.
---

# Java Spring Patterns Skill

## Purpose

This skill establishes the coding standards and patterns for Java 21+ with Spring Boot 3.x.
All generated code should follow these conventions for consistency across the codebase.
This serves as the primary reference for code generation decisions.

## Java 21+ Features

### Records for DTOs

Use records for all immutable data carriers — request DTOs, response DTOs, and value objects:

```java
// Request DTO with validation
public record CreateFieldRequest(
    @NotBlank String fieldName,
    @NotBlank String fieldType,
    String format,
    String description,
    boolean required,
    @NotBlank String schemaName
) {}

// Response DTO
public record FieldResponse(
    Long id,
    String fieldName,
    String fieldType,
    String format,
    String description,
    boolean required,
    String schemaName,
    Instant createdAt
) {
    public static FieldResponse from(ApiField entity) {
        return new FieldResponse(
            entity.getId(),
            entity.getFieldName(),
            entity.getFieldType(),
            entity.getFormat(),
            entity.getDescription(),
            entity.isRequired(),
            entity.getSchemaName(),
            entity.getCreatedAt()
        );
    }
}

// Paginated response wrapper
public record PageResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean hasNext
) {
    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(
            page.getContent(),
            page.getNumber(),
            page.getSize(),
            page.getTotalElements(),
            page.getTotalPages(),
            page.hasNext()
        );
    }
}
```

### Sealed Classes

Use sealed classes for domain type hierarchies:

```java
public sealed interface SearchResult permits FieldResult, SchemaResult, EndpointResult {
    String displayName();
    double relevanceScore();
}

public record FieldResult(String fieldName, String type, double relevanceScore) implements SearchResult {
    @Override public String displayName() { return fieldName; }
}

public record SchemaResult(String schemaName, int fieldCount, double relevanceScore) implements SearchResult {
    @Override public String displayName() { return schemaName; }
}
```

### Pattern Matching

```java
// switch expressions with pattern matching
public String describe(SearchResult result) {
    return switch (result) {
        case FieldResult f -> "Field: %s (%s)".formatted(f.fieldName(), f.type());
        case SchemaResult s -> "Schema: %s with %d fields".formatted(s.schemaName(), s.fieldCount());
        case EndpointResult e -> "Endpoint: %s %s".formatted(e.method(), e.path());
    };
}

// instanceof pattern matching
if (obj instanceof ApiField field && field.isRequired()) {
    processRequired(field);
}
```

### Text Blocks

Use text blocks for multi-line strings, SQL, and JSON:

```java
String sql = """
    SELECT f.id, f.field_name, f.description
    FROM api_fields f
    WHERE f.schema_name = :schemaName
    ORDER BY f.field_name
    """;
```

### Virtual Threads (Spring Boot 3.2+)

Enable virtual threads in `application.yml`:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

## Spring Boot 3.x Conventions

### Constructor Injection (Always)

Never use `@Autowired` on fields. Always use constructor injection:

```java
@Service
public class FieldService {

    private final FieldRepository fieldRepository;
    private final SchemaRepository schemaRepository;

    // Single constructor — @Autowired not needed
    public FieldService(FieldRepository fieldRepository, SchemaRepository schemaRepository) {
        this.fieldRepository = fieldRepository;
        this.schemaRepository = schemaRepository;
    }
}
```

### Configuration Properties

```java
@ConfigurationProperties(prefix = "app.openapi")
public record OpenApiProperties(
    String specPath,
    boolean validateOnStartup,
    Duration cacheTtl
) {}

// Enable in application class or config
@EnableConfigurationProperties(OpenApiProperties.class)
```

## JPA Entity Patterns

```java
@Entity
@Table(name = "api_fields", indexes = {
    @Index(name = "idx_field_schema", columnList = "schema_name"),
    @Index(name = "idx_field_name", columnList = "field_name")
})
public class ApiField {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "field_name", nullable = false, length = 255)
    private String fieldName;

    @Column(name = "field_type", nullable = false, length = 100)
    private String fieldType;

    @Column(name = "format", length = 100)
    private String format;

    @Column(name = "description", columnDefinition = "TEXT")
    private String description;

    @Column(name = "required", nullable = false)
    private boolean required;

    @Column(name = "deprecated", nullable = false)
    private boolean deprecated;

    @Column(name = "schema_name", nullable = false, length = 255)
    private String schemaName;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "spec_id", nullable = false)
    private ApiSpec spec;

    @OneToMany(mappedBy = "field", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<FieldConstraint> constraints = new ArrayList<>();

    protected ApiField() {} // JPA requires no-arg constructor

    public ApiField(String fieldName, String fieldType, String schemaName) {
        this.fieldName = fieldName;
        this.fieldType = fieldType;
        this.schemaName = schemaName;
    }

    @PrePersist
    protected void onCreate() {
        this.createdAt = Instant.now();
        this.updatedAt = Instant.now();
    }

    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = Instant.now();
    }

    // getters and setters omitted for brevity
}
```

## Spring Data Repositories

```java
public interface FieldRepository extends JpaRepository<ApiField, Long> {

    List<ApiField> findBySchemaName(String schemaName);

    Optional<ApiField> findBySchemaNameAndFieldName(String schemaName, String fieldName);

    @Query("SELECT f FROM ApiField f WHERE f.spec.id = :specId ORDER BY f.schemaName, f.fieldName")
    List<ApiField> findBySpecId(@Param("specId") Long specId);

    // Projection
    @Query("SELECT f.schemaName AS schemaName, COUNT(f) AS fieldCount " +
           "FROM ApiField f GROUP BY f.schemaName ORDER BY f.schemaName")
    List<SchemaFieldCount> countFieldsBySchema();

    interface SchemaFieldCount {
        String getSchemaName();
        Long getFieldCount();
    }

    // Paginated with filter
    @Query("SELECT f FROM ApiField f WHERE " +
           "(:schemaName IS NULL OR f.schemaName = :schemaName) AND " +
           "(:fieldType IS NULL OR f.fieldType = :fieldType)")
    Page<ApiField> findByFilters(
        @Param("schemaName") String schemaName,
        @Param("fieldType") String fieldType,
        Pageable pageable
    );
}
```

## Service Layer

```java
@Service
public class FieldService {

    private final FieldRepository fieldRepository;

    public FieldService(FieldRepository fieldRepository) {
        this.fieldRepository = fieldRepository;
    }

    @Transactional(readOnly = true)
    public PageResponse<FieldResponse> listFields(String schemaName, String fieldType,
                                                    int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("fieldName").ascending());
        Page<ApiField> result = fieldRepository.findByFilters(schemaName, fieldType, pageable);
        return PageResponse.from(result.map(FieldResponse::from));
    }

    @Transactional(readOnly = true)
    public FieldResponse getField(Long id) {
        return fieldRepository.findById(id)
            .map(FieldResponse::from)
            .orElseThrow(() -> new ResourceNotFoundException("Field", id));
    }

    @Transactional
    public FieldResponse createField(CreateFieldRequest request) {
        ApiField field = new ApiField(
            request.fieldName(),
            request.fieldType(),
            request.schemaName()
        );
        field.setFormat(request.format());
        field.setDescription(request.description());
        field.setRequired(request.required());

        ApiField saved = fieldRepository.save(field);
        return FieldResponse.from(saved);
    }
}
```

## Controller Patterns

```java
@RestController
@RequestMapping("/api/v1/fields")
public class FieldController {

    private final FieldService fieldService;

    public FieldController(FieldService fieldService) {
        this.fieldService = fieldService;
    }

    @GetMapping
    public PageResponse<FieldResponse> listFields(
            @RequestParam(required = false) String schemaName,
            @RequestParam(required = false) String fieldType,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return fieldService.listFields(schemaName, fieldType, page, size);
    }

    @GetMapping("/{id}")
    public FieldResponse getField(@PathVariable Long id) {
        return fieldService.getField(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public FieldResponse createField(@Valid @RequestBody CreateFieldRequest request) {
        return fieldService.createField(request);
    }
}
```

## Exception Handling

### @ControllerAdvice with ProblemDetail (RFC 7807)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setProperty("resourceType", ex.getResourceType());
        problem.setProperty("resourceId", ex.getResourceId());
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, "Validation failed");
        problem.setTitle("Validation Error");

        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage()));
        problem.setProperty("fieldErrors", errors);
        return problem;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGeneral(Exception ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected error occurred");
        problem.setTitle("Internal Server Error");
        return problem;
    }
}
```

### Custom Exception

```java
public class ResourceNotFoundException extends RuntimeException {

    private final String resourceType;
    private final Object resourceId;

    public ResourceNotFoundException(String resourceType, Object resourceId) {
        super("%s not found with id: %s".formatted(resourceType, resourceId));
        this.resourceType = resourceType;
        this.resourceId = resourceId;
    }

    public String getResourceType() { return resourceType; }
    public Object getResourceId() { return resourceId; }
}
```

## Flyway Migrations

### Naming Convention

```
V1__create_api_specs_table.sql
V2__create_api_fields_table.sql
V3__add_search_vector_column.sql
V4__create_field_constraints_table.sql
R__refresh_search_views.sql            # Repeatable migration
```

### Example Migration

```sql
-- V2__create_api_fields_table.sql
CREATE TABLE api_fields (
    id              BIGSERIAL PRIMARY KEY,
    field_name      VARCHAR(255)  NOT NULL,
    field_type      VARCHAR(100)  NOT NULL,
    format          VARCHAR(100),
    description     TEXT,
    required        BOOLEAN       NOT NULL DEFAULT FALSE,
    deprecated      BOOLEAN       NOT NULL DEFAULT FALSE,
    schema_name     VARCHAR(255)  NOT NULL,
    spec_id         BIGINT        NOT NULL REFERENCES api_specs(id),
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_api_fields_schema_name ON api_fields(schema_name);
CREATE INDEX idx_api_fields_spec_id ON api_fields(spec_id);
```

### Flyway Configuration

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
```

## Spring Profiles

### application.yml (shared)

```yaml
spring:
  application:
    name: datadict
  jpa:
    open-in-view: false
    properties:
      hibernate:
        default_batch_fetch_size: 20
```

### application-dev.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/datadict_dev
    username: dev_user
    password: dev_password
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: validate
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

### application-prod.yml

```yaml
spring:
  datasource:
    url: ${DATABASE_URL}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: none
```

## Testing Patterns

### Unit Test with Mockito

```java
@ExtendWith(MockitoExtension.class)
class FieldServiceTest {

    @Mock
    private FieldRepository fieldRepository;

    @InjectMocks
    private FieldService fieldService;

    @Test
    void getField_whenExists_returnsResponse() {
        ApiField field = new ApiField("userId", "integer", "User");
        field.setFormat("int64");
        when(fieldRepository.findById(1L)).thenReturn(Optional.of(field));

        FieldResponse result = fieldService.getField(1L);

        assertThat(result.fieldName()).isEqualTo("userId");
        assertThat(result.fieldType()).isEqualTo("integer");
    }

    @Test
    void getField_whenNotExists_throwsException() {
        when(fieldRepository.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> fieldService.getField(99L))
            .isInstanceOf(ResourceNotFoundException.class)
            .hasMessageContaining("Field not found");
    }
}
```

### Integration Test with Testcontainers

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class FieldControllerIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void listFields_returnsPagedResults() {
        ResponseEntity<PageResponse> response = restTemplate.getForEntity(
            "/api/v1/fields?page=0&size=10", PageResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotNull();
    }
}
```

### MockMvc Controller Test

```java
@WebMvcTest(FieldController.class)
class FieldControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private FieldService fieldService;

    @Test
    void createField_withValidRequest_returnsCreated() throws Exception {
        String requestBody = """
            {
                "fieldName": "userId",
                "fieldType": "integer",
                "schemaName": "User"
            }
            """;

        when(fieldService.createField(any())).thenReturn(
            new FieldResponse(1L, "userId", "integer", null, null, false, "User", Instant.now()));

        mockMvc.perform(post("/api/v1/fields")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.fieldName").value("userId"));
    }

    @Test
    void createField_withMissingRequired_returnsBadRequest() throws Exception {
        String requestBody = """
            {
                "fieldName": "",
                "fieldType": "integer",
                "schemaName": "User"
            }
            """;

        mockMvc.perform(post("/api/v1/fields")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.fieldErrors.fieldName").exists());
    }
}
```

## Maven Configuration

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.4.1</version>
</parent>

<properties>
    <java.version>21</java.version>
    <testcontainers.version>1.20.4</testcontainers.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
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
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-database-postgresql</artifactId>
    </dependency>

    <!-- Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers-bom</artifactId>
            <version>${testcontainers.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```
