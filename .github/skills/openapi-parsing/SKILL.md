---
name: openapi-parsing
description: >-
  Domain knowledge for parsing OpenAPI 3.x specifications in Java. Covers swagger-parser
  library usage, $ref resolution, allOf/oneOf/anyOf composition, field metadata extraction,
  and schema traversal patterns. Use when generating OpenAPI parsing code.
---

# OpenAPI Parsing Skill

## Purpose

This skill provides patterns and code examples for parsing OpenAPI 3.x specification
files in Java using the swagger-parser library. It covers the full lifecycle from loading
a spec file, resolving references, traversing schemas, extracting field metadata, and
mapping endpoints to their request/response schemas.

## OpenAPI 3.x Structure Overview

### Core Structure

An OpenAPI 3.x document has these top-level sections:

```yaml
openapi: "3.0.3"
info:
  title: My API
  version: 1.0.0
paths:
  /users:
    get: ...
    post: ...
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
          format: int64
        name:
          type: string
  parameters: ...
  responses: ...
  requestBodies: ...
```

### Components/Schemas

All reusable data models live under `components/schemas`. Each schema defines:

- **type** — `object`, `string`, `integer`, `number`, `boolean`, `array`
- **properties** — map of field name → schema
- **required** — list of required field names
- **description** — human-readable documentation
- **format** — type refinement (`int32`, `int64`, `date-time`, `email`, `uuid`, etc.)
- **enum** — list of allowed values

### $ref References

References use JSON Pointer syntax to point to reusable components:

```yaml
$ref: '#/components/schemas/User'
```

When parsing, references must be resolved before traversing properties. The swagger-parser
library handles this automatically with `resolveCompletely()`.

### Composition Keywords

- **allOf** — combines multiple schemas (inheritance/mixins). All listed schemas must be satisfied.
- **oneOf** — exactly one of the listed schemas must match (polymorphism).
- **anyOf** — one or more of the listed schemas can match.
- **discriminator** — used with oneOf/anyOf to indicate which schema applies based on a property value.

```yaml
Pet:
  oneOf:
    - $ref: '#/components/schemas/Cat'
    - $ref: '#/components/schemas/Dog'
  discriminator:
    propertyName: petType
```

## Maven Dependency

```xml
<dependency>
    <groupId>io.swagger.parser.v3</groupId>
    <artifactId>swagger-parser</artifactId>
    <version>2.1.22</version>
</dependency>
```

## Java Parsing with swagger-parser

### Loading and Resolving a Spec

```java
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.media.Schema;
import io.swagger.v3.parser.OpenAPIV3Parser;
import io.swagger.v3.parser.core.models.ParseOptions;
import io.swagger.v3.parser.core.models.SwaggerParseResult;

public class OpenApiLoader {

    public OpenAPI loadSpec(String filePath) {
        ParseOptions options = new ParseOptions();
        options.setResolve(true);        // resolve $ref pointers
        options.setResolveFully(true);   // inline all references

        SwaggerParseResult result = new OpenAPIV3Parser().readLocation(filePath, null, options);

        if (result.getMessages() != null && !result.getMessages().isEmpty()) {
            result.getMessages().forEach(msg ->
                System.err.println("Parse warning: " + msg));
        }

        OpenAPI openAPI = result.getOpenAPI();
        if (openAPI == null) {
            throw new IllegalArgumentException("Failed to parse OpenAPI spec: " + filePath);
        }
        return openAPI;
    }
}
```

### Accessing Component Schemas

```java
import io.swagger.v3.oas.models.media.Schema;
import java.util.Map;

public Map<String, Schema> getSchemas(OpenAPI openAPI) {
    if (openAPI.getComponents() == null || openAPI.getComponents().getSchemas() == null) {
        return Map.of();
    }
    return openAPI.getComponents().getSchemas();
}
```

## Schema Traversal Patterns

### Recursive Field Extraction

```java
import io.swagger.v3.oas.models.media.ArraySchema;
import io.swagger.v3.oas.models.media.MapSchema;
import io.swagger.v3.oas.models.media.Schema;

import java.util.*;

public record FieldInfo(
    String path,
    String type,
    String format,
    String description,
    boolean required,
    boolean deprecated,
    List<String> enumValues,
    Map<String, Object> constraints
) {}

public class SchemaTraverser {

    private final Set<String> visited = new HashSet<>();

    public List<FieldInfo> extractFields(String schemaName, Schema<?> schema) {
        List<FieldInfo> fields = new ArrayList<>();
        visited.clear();
        traverseSchema(schemaName, schema, "", fields, Set.of());
        return fields;
    }

    @SuppressWarnings("unchecked")
    private void traverseSchema(String context, Schema<?> schema, String parentPath,
                                 List<FieldInfo> fields, Set<String> requiredFields) {
        if (schema == null) return;

        // Guard against circular references
        String schemaId = System.identityHashCode(schema) + parentPath;
        if (visited.contains(schemaId)) return;
        visited.add(schemaId);

        // Handle allOf composition — merge all sub-schemas
        if (schema.getAllOf() != null) {
            Set<String> mergedRequired = new HashSet<>(requiredFields);
            for (Schema<?> subSchema : schema.getAllOf()) {
                if (subSchema.getRequired() != null) {
                    mergedRequired.addAll(subSchema.getRequired());
                }
                traverseSchema(context, subSchema, parentPath, fields, mergedRequired);
            }
            return;
        }

        // Handle oneOf / anyOf — traverse each variant
        List<Schema> variants = schema.getOneOf() != null ? schema.getOneOf() : schema.getAnyOf();
        if (variants != null) {
            for (Schema<?> variant : variants) {
                traverseSchema(context, variant, parentPath, fields, requiredFields);
            }
            return;
        }

        // Handle object with properties
        Map<String, Schema> properties = schema.getProperties();
        if (properties != null) {
            Set<String> required = new HashSet<>(requiredFields);
            if (schema.getRequired() != null) {
                required.addAll(schema.getRequired());
            }

            for (Map.Entry<String, Schema> entry : properties.entrySet()) {
                String fieldName = entry.getKey();
                Schema<?> fieldSchema = entry.getValue();
                String fieldPath = parentPath.isEmpty() ? fieldName : parentPath + "." + fieldName;

                fields.add(buildFieldInfo(fieldPath, fieldSchema, required.contains(fieldName)));

                // Recurse into nested objects
                if ("object".equals(fieldSchema.getType()) && fieldSchema.getProperties() != null) {
                    Set<String> childRequired = fieldSchema.getRequired() != null
                        ? new HashSet<>(fieldSchema.getRequired()) : Set.of();
                    traverseSchema(context, fieldSchema, fieldPath, fields, childRequired);
                }

                // Recurse into array items
                if ("array".equals(fieldSchema.getType()) && fieldSchema.getItems() != null) {
                    String itemPath = fieldPath + "[]";
                    Schema<?> itemSchema = fieldSchema.getItems();
                    if ("object".equals(itemSchema.getType())) {
                        Set<String> childRequired = itemSchema.getRequired() != null
                            ? new HashSet<>(itemSchema.getRequired()) : Set.of();
                        traverseSchema(context, itemSchema, itemPath, fields, childRequired);
                    }
                }
            }
        }
    }

    @SuppressWarnings("unchecked")
    private FieldInfo buildFieldInfo(String path, Schema<?> schema, boolean isRequired) {
        Map<String, Object> constraints = new LinkedHashMap<>();
        if (schema.getMinLength() != null) constraints.put("minLength", schema.getMinLength());
        if (schema.getMaxLength() != null) constraints.put("maxLength", schema.getMaxLength());
        if (schema.getMinimum() != null) constraints.put("minimum", schema.getMinimum());
        if (schema.getMaximum() != null) constraints.put("maximum", schema.getMaximum());
        if (schema.getPattern() != null) constraints.put("pattern", schema.getPattern());
        if (schema.getMinItems() != null) constraints.put("minItems", schema.getMinItems());
        if (schema.getMaxItems() != null) constraints.put("maxItems", schema.getMaxItems());
        if (schema.getUniqueItems() != null) constraints.put("uniqueItems", schema.getUniqueItems());

        List<String> enumValues = schema.getEnum() != null
            ? schema.getEnum().stream().map(Object::toString).toList()
            : List.of();

        return new FieldInfo(
            path,
            resolveType(schema),
            schema.getFormat(),
            schema.getDescription(),
            isRequired,
            Boolean.TRUE.equals(schema.getDeprecated()),
            enumValues,
            constraints
        );
    }

    private String resolveType(Schema<?> schema) {
        if (schema.getType() != null) return schema.getType();
        if (schema.get$ref() != null) {
            String ref = schema.get$ref();
            return ref.substring(ref.lastIndexOf('/') + 1);
        }
        return "unknown";
    }
}
```

### Handling Map/Dictionary Schemas

```java
// additionalProperties indicates a map/dictionary type
if (schema.getAdditionalProperties() instanceof Schema<?> valueSchema) {
    String mapValueType = resolveType(valueSchema);
    // type is Map<String, mapValueType>
}
```

## Field Metadata Extraction

Key metadata fields available on every `Schema<?>` object:

| Method                  | Returns          | Description                              |
|-------------------------|------------------|------------------------------------------|
| `getType()`             | `String`         | Base type: object, string, integer, etc. |
| `getFormat()`           | `String`         | Format hint: int64, date-time, uuid      |
| `getDescription()`      | `String`         | Human-readable field description         |
| `getEnum()`             | `List<Object>`   | Allowed values for enum fields           |
| `getRequired()`         | `List<String>`   | Required child property names            |
| `getDeprecated()`       | `Boolean`        | Whether the field is deprecated          |
| `getDefault()`          | `Object`         | Default value                            |
| `getExample()`          | `Object`         | Example value                            |
| `getReadOnly()`         | `Boolean`        | Read-only (response only)                |
| `getWriteOnly()`        | `Boolean`        | Write-only (request only)                |
| `getNullable()`         | `Boolean`        | Whether null is allowed                  |
| `getMinLength()`        | `Integer`        | Minimum string length                    |
| `getMaxLength()`        | `Integer`        | Maximum string length                    |
| `getMinimum()`          | `BigDecimal`     | Minimum numeric value                    |
| `getMaximum()`          | `BigDecimal`     | Maximum numeric value                    |
| `getPattern()`          | `String`         | Regex validation pattern                 |
| `getMinItems()`         | `Integer`        | Minimum array items                      |
| `getMaxItems()`         | `Integer`        | Maximum array items                      |
| `getUniqueItems()`      | `Boolean`        | Whether array items must be unique       |

## Endpoint-to-Field Mapping

### Extracting Request/Response Schemas per Path

```java
import io.swagger.v3.oas.models.PathItem;
import io.swagger.v3.oas.models.Operation;
import io.swagger.v3.oas.models.media.Content;
import io.swagger.v3.oas.models.media.MediaType;
import io.swagger.v3.oas.models.parameters.Parameter;

public record EndpointSchema(
    String method,
    String path,
    String operationId,
    Schema<?> requestBodySchema,
    Map<String, Schema<?>> responseSchemas,
    List<Parameter> parameters
) {}

public List<EndpointSchema> extractEndpoints(OpenAPI openAPI) {
    List<EndpointSchema> endpoints = new ArrayList<>();

    openAPI.getPaths().forEach((path, pathItem) -> {
        Map<PathItem.HttpMethod, Operation> operations = pathItem.readOperationsMap();
        operations.forEach((method, operation) -> {
            Schema<?> requestSchema = extractRequestSchema(operation);
            Map<String, Schema<?>> responseSchemas = extractResponseSchemas(operation);

            endpoints.add(new EndpointSchema(
                method.name(),
                path,
                operation.getOperationId(),
                requestSchema,
                responseSchemas,
                operation.getParameters() != null ? operation.getParameters() : List.of()
            ));
        });
    });
    return endpoints;
}

private Schema<?> extractRequestSchema(Operation operation) {
    if (operation.getRequestBody() == null) return null;
    Content content = operation.getRequestBody().getContent();
    if (content == null) return null;
    MediaType json = content.get("application/json");
    return json != null ? json.getSchema() : null;
}

private Map<String, Schema<?>> extractResponseSchemas(Operation operation) {
    Map<String, Schema<?>> schemas = new LinkedHashMap<>();
    if (operation.getResponses() == null) return schemas;

    operation.getResponses().forEach((statusCode, response) -> {
        if (response.getContent() != null) {
            MediaType json = response.getContent().get("application/json");
            if (json != null && json.getSchema() != null) {
                schemas.put(statusCode, json.getSchema());
            }
        }
    });
    return schemas;
}
```

## Common Pitfalls

### Circular References

Schemas can reference each other, creating infinite recursion during traversal.
Always maintain a `visited` set and check before recursing. Use `System.identityHashCode()`
or schema name tracking to detect cycles.

### Discriminator Handling

When `oneOf` or `anyOf` includes a `discriminator`, the discriminator property indicates
which sub-schema applies. Always check for discriminator metadata:

```java
if (schema.getDiscriminator() != null) {
    String propName = schema.getDiscriminator().getPropertyName();
    Map<String, String> mapping = schema.getDiscriminator().getMapping();
    // mapping: discriminator value → $ref path
}
```

### Nullable vs Required

In OpenAPI 3.0, `nullable: true` means the field accepts null. In OpenAPI 3.1, nullable
is replaced with `type: ["string", "null"]`. These are distinct from `required` — a field
can be required but nullable (must be present, can be null).

### Fully Resolving References

Always use `setResolveFully(true)` in parse options. Without it, `$ref` objects remain
unresolved and `getProperties()` returns null on referenced schemas.

### allOf Merging

When `allOf` contains multiple schemas, properties from all sub-schemas must be merged.
The swagger-parser resolves references but does NOT automatically merge allOf properties.
You must iterate each allOf entry and collect properties manually.

### Missing Type Field

Some schemas omit `type` when using composition keywords. If `getType()` returns null,
check for `allOf`, `oneOf`, `anyOf`, or `$ref` before assuming the schema is invalid.

### Array Items

Always check `schema.getItems()` is not null before accessing array item schemas.
A bare `type: array` without `items` is technically valid but provides no item type info.
