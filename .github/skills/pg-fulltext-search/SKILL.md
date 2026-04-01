---
name: pg-fulltext-search
description: >-
  Domain knowledge for implementing PostgreSQL full-text search with tsvector/tsquery
  and fuzzy matching with pg_trgm. Covers GIN index creation, search ranking, autocomplete,
  and integration with Spring Data JPA / JdbcTemplate. Use when building search features.
---

# PostgreSQL Full-Text Search Skill

## Purpose

This skill provides patterns for implementing search in PostgreSQL-backed Java applications.
It covers two complementary approaches — full-text search (tsvector/tsquery) for semantic
keyword matching and pg_trgm for fuzzy/similarity matching — and shows how to integrate
both with Spring Data JPA and JdbcTemplate.

## PostgreSQL FTS Fundamentals

### tsvector — Document Representation

A `tsvector` is a sorted list of normalized lexemes (word stems) with position information.
PostgreSQL uses language-aware stemming so "running", "runs", and "ran" all normalize to "run".

```sql
SELECT to_tsvector('english', 'The quick brown foxes jumped over lazy dogs');
-- Result: 'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2
```

### tsquery — Search Query Representation

A `tsquery` represents a search condition using lexemes and boolean operators.

```sql
-- Simple word
SELECT to_tsquery('english', 'running');    -- 'run'

-- Boolean operators: & (AND), | (OR), ! (NOT), <-> (FOLLOWED BY)
SELECT to_tsquery('english', 'quick & fox');
SELECT to_tsquery('english', 'cat | dog');
SELECT to_tsquery('english', '!spam');
SELECT to_tsquery('english', 'quick <-> fox');  -- phrase search
```

### Query Construction Functions

| Function            | Input                    | Best For                       |
|---------------------|--------------------------|--------------------------------|
| `to_tsquery`        | Structured query syntax  | Power users, programmatic use  |
| `plainto_tsquery`   | Plain text (AND terms)   | Simple search boxes            |
| `phraseto_tsquery`  | Plain text (phrase)      | Exact phrase matching          |
| `websearch_to_tsquery` | Web-style syntax      | User-facing search (PG 11+)   |

```sql
-- websearch_to_tsquery supports quotes, OR, and - (NOT)
SELECT websearch_to_tsquery('english', '"quick fox" OR lazy -cat');
-- Result: 'quick' <-> 'fox' | 'lazi' & !'cat'
```

### The Match Operator @@

```sql
SELECT *
FROM documents
WHERE to_tsvector('english', content) @@ websearch_to_tsquery('english', 'search terms');
```

## GIN Index Setup

### Basic tsvector GIN Index

```sql
-- Option 1: Index on expression (no stored column needed)
CREATE INDEX idx_documents_fts ON documents
    USING GIN (to_tsvector('english', title || ' ' || description));

-- Option 2: Stored generated column with index (faster queries)
ALTER TABLE documents
    ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(description, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(tags, '')), 'C')
    ) STORED;

CREATE INDEX idx_documents_search_vector ON documents USING GIN (search_vector);
```

### Weighted Search Vectors

Weights (A, B, C, D) let you rank matches in titles higher than matches in body text:

```sql
-- A = 1.0, B = 0.4, C = 0.2, D = 0.1 (defaults)
ALTER TABLE api_fields
    ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(field_name, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(description, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(schema_name, '')), 'C') ||
        setweight(to_tsvector('english', coalesce(field_type, '')), 'D')
    ) STORED;
```

## pg_trgm for Fuzzy Search

### Enabling the Extension

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

### Trigram Similarity

pg_trgm breaks strings into 3-character sequences and computes similarity (0.0–1.0):

```sql
SELECT similarity('postgresql', 'postgre');   -- 0.6153846
SELECT similarity('search', 'serach');        -- 0.4545455  (handles typos)

-- Similarity operators
SELECT * FROM fields WHERE field_name % 'user_naem';      -- similarity > threshold
SELECT * FROM fields WHERE field_name <% 'user_naem';     -- word similarity
```

### GIN Trigram Index

```sql
CREATE INDEX idx_fields_name_trgm ON api_fields
    USING GIN (field_name gin_trgm_ops);

CREATE INDEX idx_fields_desc_trgm ON api_fields
    USING GIN (description gin_trgm_ops);
```

### Setting Similarity Threshold

```sql
-- Default threshold is 0.3; lower for more results, raise for stricter matching
SET pg_trgm.similarity_threshold = 0.2;

-- Or use similarity() directly with a custom threshold
SELECT * FROM fields
WHERE similarity(field_name, 'usernaem') > 0.25
ORDER BY similarity(field_name, 'usernaem') DESC;
```

## Search Ranking

### ts_rank and ts_rank_cd

```sql
-- ts_rank: term frequency based ranking
SELECT
    title,
    ts_rank(search_vector, query) AS rank
FROM documents, websearch_to_tsquery('english', 'api field') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- ts_rank_cd: cover density ranking (considers proximity of matched terms)
SELECT
    title,
    ts_rank_cd(search_vector, query, 32) AS rank  -- 32 = normalize by rank / (1 + rank)
FROM documents, websearch_to_tsquery('english', 'api field') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### Normalization Options (bitmask)

| Value | Effect                                  |
|-------|-----------------------------------------|
| 0     | Default (no normalization)              |
| 1     | Divides rank by 1 + log(document length)|
| 2     | Divides rank by document length         |
| 4     | Divides rank by harmonic distance       |
| 8     | Divides rank by unique word count       |
| 16    | Divides rank by 1 + log(unique words)   |
| 32    | Divides rank by itself + 1              |

```sql
-- Combine with bitwise OR: normalize by doc length and unique words
SELECT ts_rank_cd(search_vector, query, 2|8) AS rank ...
```

## Autocomplete Implementation

### Prefix Matching with tsquery

```sql
-- :* is the prefix match operator in tsquery
SELECT * FROM api_fields
WHERE search_vector @@ to_tsquery('english', 'user:*')
ORDER BY ts_rank(search_vector, to_tsquery('english', 'user:*')) DESC
LIMIT 10;
```

### Trigram-Based Autocomplete

```sql
-- Combine prefix LIKE with trigram similarity for best autocomplete
SELECT field_name, schema_name,
       similarity(field_name, :query) AS sim
FROM api_fields
WHERE field_name ILIKE :query || '%'
   OR field_name % :query
ORDER BY
    (field_name ILIKE :query || '%') DESC,  -- exact prefix first
    sim DESC
LIMIT 10;
```

## Spring Data JPA Integration

### Native @Query Examples

```java
public interface FieldRepository extends JpaRepository<ApiField, Long> {

    // Full-text search with ranking
    @Query(value = """
        SELECT f.* FROM api_fields f,
               websearch_to_tsquery('english', :query) q
        WHERE f.search_vector @@ q
        ORDER BY ts_rank_cd(f.search_vector, q) DESC
        """, nativeQuery = true)
    List<ApiField> fullTextSearch(@Param("query") String query);

    // Full-text search with pagination
    @Query(value = """
        SELECT f.* FROM api_fields f,
               websearch_to_tsquery('english', :query) q
        WHERE f.search_vector @@ q
        ORDER BY ts_rank_cd(f.search_vector, q) DESC
        """,
        countQuery = """
        SELECT count(*) FROM api_fields f,
               websearch_to_tsquery('english', :query) q
        WHERE f.search_vector @@ q
        """,
        nativeQuery = true)
    Page<ApiField> fullTextSearch(@Param("query") String query, Pageable pageable);

    // Fuzzy search with pg_trgm
    @Query(value = """
        SELECT f.* FROM api_fields f
        WHERE f.field_name % :query
        ORDER BY similarity(f.field_name, :query) DESC
        LIMIT :limit
        """, nativeQuery = true)
    List<ApiField> fuzzySearchByName(@Param("query") String query, @Param("limit") int limit);

    // Search with headline snippets
    @Query(value = """
        SELECT f.*,
               ts_headline('english', f.description,
                   websearch_to_tsquery('english', :query),
                   'StartSel=<mark>, StopSel=</mark>, MaxFragments=2'
               ) AS headline
        FROM api_fields f,
             websearch_to_tsquery('english', :query) q
        WHERE f.search_vector @@ q
        ORDER BY ts_rank_cd(f.search_vector, q) DESC
        """, nativeQuery = true)
    List<Object[]> searchWithHeadlines(@Param("query") String query);
}
```

## JdbcTemplate for Complex FTS

```java
@Repository
public class FieldSearchDao {

    private final JdbcTemplate jdbc;

    public FieldSearchDao(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public List<FieldSearchResult> search(String query, int limit, int offset) {
        return jdbc.query("""
            WITH search_query AS (
                SELECT websearch_to_tsquery('english', ?) AS q
            )
            SELECT
                f.id,
                f.field_name,
                f.schema_name,
                f.field_type,
                f.description,
                ts_rank_cd(f.search_vector, sq.q) AS fts_rank,
                similarity(f.field_name, ?) AS trgm_score,
                ts_headline('english', f.description, sq.q,
                    'StartSel=<mark>, StopSel=</mark>, MaxFragments=2, MaxWords=30'
                ) AS headline
            FROM api_fields f
            CROSS JOIN search_query sq
            WHERE f.search_vector @@ sq.q
               OR f.field_name %% ?
            ORDER BY
                (CASE WHEN f.search_vector @@ sq.q THEN 1 ELSE 0 END) DESC,
                ts_rank_cd(f.search_vector, sq.q) DESC,
                similarity(f.field_name, ?) DESC
            LIMIT ? OFFSET ?
            """,
            new Object[]{query, query, query, query, limit, offset},
            (rs, rowNum) -> new FieldSearchResult(
                rs.getLong("id"),
                rs.getString("field_name"),
                rs.getString("schema_name"),
                rs.getString("field_type"),
                rs.getString("description"),
                rs.getDouble("fts_rank"),
                rs.getDouble("trgm_score"),
                rs.getString("headline")
            ));
    }
}
```

## Combined Search Strategy

### FTS + Trigram in One Query

The most effective approach combines both methods with weighted scoring:

```sql
WITH query AS (
    SELECT
        websearch_to_tsquery('english', :input) AS fts_query,
        :input AS raw_input
)
SELECT
    f.id,
    f.field_name,
    f.description,
    -- Combined score: FTS relevance + fuzzy similarity
    (
        COALESCE(ts_rank_cd(f.search_vector, q.fts_query), 0) * 2.0 +
        COALESCE(similarity(f.field_name, q.raw_input), 0) * 1.0
    ) AS combined_score,
    f.search_vector @@ q.fts_query AS fts_match,
    f.field_name % q.raw_input AS fuzzy_match
FROM api_fields f
CROSS JOIN query q
WHERE f.search_vector @@ q.fts_query
   OR f.field_name % q.raw_input
ORDER BY combined_score DESC
LIMIT 20;
```

### Spring Service Combining Both

```java
@Service
public class FieldSearchService {

    private final FieldSearchDao searchDao;

    public FieldSearchService(FieldSearchDao searchDao) {
        this.searchDao = searchDao;
    }

    public SearchResponse search(String query, int page, int size) {
        if (query == null || query.isBlank()) {
            return SearchResponse.empty();
        }

        String sanitized = sanitizeSearchInput(query);
        int offset = page * size;
        List<FieldSearchResult> results = searchDao.search(sanitized, size + 1, offset);

        boolean hasNext = results.size() > size;
        if (hasNext) {
            results = results.subList(0, size);
        }

        return new SearchResponse(results, page, size, hasNext);
    }

    private String sanitizeSearchInput(String input) {
        // Remove characters that could break tsquery parsing
        return input.replaceAll("[!&|():<>*\\\\]", " ").strip();
    }
}
```

## Performance Tuning

### Index-Only Scans

Ensure GIN indexes cover the columns used in WHERE clauses. Avoid function calls in
WHERE that prevent index usage:

```sql
-- BAD: cannot use GIN index (function on column)
WHERE to_tsvector('english', description) @@ query

-- GOOD: uses GIN index on stored column
WHERE search_vector @@ query
```

### Materialized Views for Complex Search

```sql
CREATE MATERIALIZED VIEW field_search_mv AS
SELECT
    f.id,
    f.field_name,
    f.schema_name,
    f.field_type,
    f.description,
    f.endpoint_path,
    setweight(to_tsvector('english', coalesce(f.field_name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(f.description, '')), 'B') ||
    setweight(to_tsvector('english', coalesce(f.schema_name, '')), 'C') AS search_vector
FROM api_fields f;

CREATE INDEX idx_field_search_mv_fts ON field_search_mv USING GIN (search_vector);
CREATE INDEX idx_field_search_mv_trgm ON field_search_mv USING GIN (field_name gin_trgm_ops);

-- Refresh periodically or after data changes
REFRESH MATERIALIZED VIEW CONCURRENTLY field_search_mv;
```

### search_path Configuration

Set `search_path` in `application.yml` to avoid schema-qualifying every query:

```yaml
spring:
  datasource:
    hikari:
      connection-init-sql: SET search_path TO public
```

### Query Execution Plan Analysis

Always check query plans when tuning search performance:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT f.* FROM api_fields f
WHERE f.search_vector @@ websearch_to_tsquery('english', 'user account')
ORDER BY ts_rank_cd(f.search_vector, websearch_to_tsquery('english', 'user account')) DESC
LIMIT 20;
```

Look for:
- **Bitmap Index Scan on GIN index** — good, index is being used
- **Seq Scan** — bad for large tables, index is missing or not applicable
- **Sort** — expected for ORDER BY rank; ensure it does not dominate execution time
