# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: OpenAPI Mock Server · Created: 2026-05-11

## Philosophy

This model stores the operational scaffolding (specs, mock instances, sessions, users) in conventional relational tables, but keeps the spec-derived content — parsed operations, schemas, response definitions, and generated data — in JSONB columns. The insight is that OpenAPI specifications are inherently document-shaped: they arrive as a single YAML/JSON file and their internal structure (nested schemas, polymorphic compositions, recursive references) maps poorly to flat relational tables.

Instead of decomposing a 500-line OpenAPI spec into 200+ rows across 10 tables, this model stores the parsed spec as a structured JSONB document in a single column, then indexes specific paths for queryability. PostgreSQL's JSONB indexing (GIN indexes, `@>` containment, `->>`/`#>>` path extraction) provides fast access to any field without requiring full normalization.

This approach is how tools like Mockoon, Postman, and VS Code store API definitions internally: the document is the source of truth, and relational structure wraps around it for identity, access control, and operational state. It trades some query flexibility (you cannot JOIN across JSONB subfields as easily as across tables) for dramatically simpler write paths, faster spec import, and natural alignment with the document-shaped input format.

**Best for:** MVP and single-tenant CLI deployments where spec import speed, schema flexibility, and development velocity matter more than cross-spec analytical queries.

**Trade-offs:**
- **Pro:** Dramatically simpler spec import — parse once, store as JSONB, done
- **Pro:** Schema evolution is trivial — new OpenAPI keywords are automatically stored without migration
- **Pro:** Natural alignment with OpenAPI's document structure; no impedance mismatch
- **Pro:** Fewer tables (~12-14 vs. ~29); faster to understand and maintain
- **Pro:** GIN indexes on JSONB columns support fast containment and path queries
- **Con:** Cross-spec queries are harder (e.g., "all operations across all specs with a 429 response")
- **Con:** JSONB columns can grow large for complex specs; memory usage is less predictable
- **Con:** No referential integrity within JSONB; application must validate internal consistency
- **Con:** JSONB updates require rewriting the entire column value (no partial update in PostgreSQL < 17)
- **Con:** Reporting and analytics queries on JSONB are verbose and slower than relational equivalents

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.1.0 / 3.2.0 | Parsed spec stored as a complete JSONB document preserving the full OAS structure |
| JSON Schema 2020-12 | Schema definitions within the JSONB spec are native JSON Schema; no translation needed |
| RFC 7807 (Problem Details) | Error templates stored as JSONB objects matching the RFC 7807 structure |
| RFC 9110 (HTTP Semantics) | HTTP method and status code fields in the operation index table |
| ISO 8601 | All timestamps use `TIMESTAMPTZ` |
| GeoJSON (RFC 7946) | Example of a complex schema type that is naturally stored in JSONB without decomposition |

---

## Core Configuration

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    api_key_hash    VARCHAR(128),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users (tenant_id);
```

---

## Specification Storage (JSONB-First)

```sql
CREATE TABLE specs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    openapi_version VARCHAR(10) NOT NULL,
    -- The entire parsed OpenAPI spec as JSONB
    parsed_spec     JSONB NOT NULL,
    -- Raw source for re-parsing and diffing
    raw_content     TEXT NOT NULL,
    format          VARCHAR(10) NOT NULL DEFAULT 'yaml',
    checksum        VARCHAR(64) NOT NULL,
    -- Extracted top-level metadata for fast filtering
    info_title      VARCHAR(500),
    info_version    VARCHAR(50),
    base_url        VARCHAR(2048),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    parse_errors    JSONB,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_specs_tenant ON specs (tenant_id);
CREATE INDEX idx_specs_status ON specs (tenant_id, status);
CREATE INDEX idx_specs_checksum ON specs (checksum);
-- GIN index on parsed_spec for containment queries
CREATE INDEX idx_specs_parsed_gin ON specs USING GIN (parsed_spec jsonb_path_ops);

-- Example: find all specs that define a User schema
-- SELECT id, name FROM specs
-- WHERE parsed_spec @> '{"components": {"schemas": {"User": {}}}}'::jsonb;
```

The `parsed_spec` JSONB column contains the complete OpenAPI document after parsing and `$ref` resolution. Example structure:

```json
{
  "openapi": "3.1.0",
  "info": { "title": "Pet Store", "version": "1.0.0" },
  "servers": [{ "url": "https://api.petstore.example.com/v1" }],
  "paths": {
    "/pets": {
      "get": {
        "operationId": "listPets",
        "summary": "List all pets",
        "parameters": [...],
        "responses": {
          "200": {
            "description": "A list of pets",
            "content": {
              "application/json": {
                "schema": { "$ref": "#/components/schemas/PetList" }
              }
            }
          }
        }
      }
    }
  },
  "components": {
    "schemas": {
      "Pet": {
        "type": "object",
        "required": ["id", "name"],
        "properties": {
          "id": { "type": "integer", "format": "int64" },
          "name": { "type": "string" },
          "tag": { "type": "string" }
        }
      }
    }
  }
}
```

---

## Operation Index (Queryable Projection)

While the full spec lives in JSONB, an **operation index** table provides fast relational lookups for request matching at runtime:

```sql
CREATE TABLE operation_index (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id         UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    path_pattern    VARCHAR(2048) NOT NULL,
    http_method     VARCHAR(10) NOT NULL,
    operation_id    VARCHAR(255),
    -- Denormalized from the spec for fast matching
    summary         TEXT,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    tags            TEXT[],                  -- PostgreSQL array of tag names
    -- JSONB slice: just this operation's definition (extracted from parsed_spec)
    operation_def   JSONB NOT NULL,
    -- Precompiled path regex for request matching
    path_regex      VARCHAR(2048),          -- e.g., '^/pets/([^/]+)$' for '/pets/{petId}'
    -- Parameter metadata for matching
    path_params     TEXT[],                 -- ['petId'] extracted from path_pattern
    has_request_body BOOLEAN NOT NULL DEFAULT false,
    response_codes  TEXT[],                 -- ['200', '404', '500']
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (spec_id, path_pattern, http_method)
);

CREATE INDEX idx_op_index_spec ON operation_index (spec_id);
CREATE INDEX idx_op_index_method ON operation_index (http_method);
CREATE INDEX idx_op_index_operation_id ON operation_index (operation_id);
CREATE INDEX idx_op_index_tags ON operation_index USING GIN (tags);
CREATE INDEX idx_op_index_response_codes ON operation_index USING GIN (response_codes);
```

This table is **rebuilt on spec import** — it is a materialized projection, not a source of truth. The source of truth is always `specs.parsed_spec`.

---

## Mock Instances

```sql
CREATE TABLE mock_instances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    spec_id         UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    -- Configuration as JSONB for flexibility
    config          JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- config example:
    -- {
    --   "basePath": "/",
    --   "port": 4010,
    --   "latencyMs": 0,
    --   "errorRate": 0.0,
    --   "corsEnabled": true,
    --   "validateRequests": true,
    --   "aiDataEnabled": true,
    --   "aiModel": "claude-sonnet-4-20250514",
    --   "proxy": {
    --     "upstreamUrl": "https://api.example.com",
    --     "driftDetection": true
    --   }
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'stopped',
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mock_instances_tenant ON mock_instances (tenant_id);
CREATE INDEX idx_mock_instances_spec ON mock_instances (spec_id);
CREATE INDEX idx_mock_instances_status ON mock_instances (status);
```

---

## Mock Rules & Overrides

```sql
CREATE TABLE mock_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mock_instance_id UUID NOT NULL REFERENCES mock_instances(id) ON DELETE CASCADE,
    name            VARCHAR(255),
    priority        INT NOT NULL DEFAULT 0,
    -- Matching conditions as JSONB (flexible, extensible)
    match_criteria  JSONB NOT NULL,
    -- match_criteria example:
    -- {
    --   "path": "/users/{userId}",
    --   "method": "GET",
    --   "headers": { "Authorization": { "matches": "Bearer .*" } },
    --   "query": { "include": { "equalTo": "profile" } },
    --   "body": { "jsonPath": "$.email", "matches": ".*@example.com" }
    -- }

    -- Response definition as JSONB
    response_def    JSONB NOT NULL,
    -- response_def example:
    -- {
    --   "status": 200,
    --   "headers": { "Content-Type": "application/json" },
    --   "body": { "id": 1, "name": "Test User" },
    --   "delayMs": 500,
    --   "template": true  -- if true, body is a Handlebars template
    -- }

    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mock_rules_instance ON mock_rules (mock_instance_id);
CREATE INDEX idx_mock_rules_priority ON mock_rules (mock_instance_id, priority DESC);
CREATE INDEX idx_mock_rules_match_gin ON mock_rules USING GIN (match_criteria jsonb_path_ops);

-- Error templates (RFC 7807)
CREATE TABLE error_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenants(id) ON DELETE CASCADE,
    status_code     INT NOT NULL,
    owasp_category  VARCHAR(50),
    template        JSONB NOT NULL,
    -- template example (RFC 7807):
    -- {
    --   "type": "https://api.example.com/errors/insufficient-funds",
    --   "title": "Insufficient Funds",
    --   "status": 402,
    --   "detail": "Account {{accountId}} has insufficient balance for this transaction",
    --   "instance": "/transactions/{{transactionId}}"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_error_templates_status ON error_templates (status_code);
```

---

## Stateful Sessions & Scenarios

```sql
CREATE TABLE scenarios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mock_instance_id UUID NOT NULL REFERENCES mock_instances(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    -- Full state machine definition as JSONB
    state_machine   JSONB NOT NULL,
    -- state_machine example:
    -- {
    --   "initialState": "empty_cart",
    --   "states": {
    --     "empty_cart": {
    --       "transitions": [{
    --         "on": { "method": "POST", "path": "/cart/items" },
    --         "to": "has_items",
    --         "mutations": { "addToArray": { "path": "$.cart.items", "fromBody": "$.item" } },
    --         "response": { "status": 201 }
    --       }]
    --     },
    --     "has_items": {
    --       "transitions": [
    --         { "on": { "method": "POST", "path": "/checkout" }, "to": "checked_out" },
    --         { "on": { "method": "DELETE", "path": "/cart/items/{itemId}" }, "to": "empty_cart",
    --           "conditions": { "stateQuery": "$.cart.items[?(@.length == 1)]" } }
    --       ]
    --     },
    --     "checked_out": { "transitions": [] }
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scenarios_mock ON scenarios (mock_instance_id);

CREATE TABLE sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mock_instance_id UUID NOT NULL REFERENCES mock_instances(id) ON DELETE CASCADE,
    scenario_id     UUID REFERENCES scenarios(id) ON DELETE SET NULL,
    session_token   VARCHAR(128) NOT NULL UNIQUE,
    current_state   VARCHAR(255) NOT NULL DEFAULT 'initial',
    -- All accumulated session state in one JSONB document
    state_data      JSONB NOT NULL DEFAULT '{}',
    -- state_data example:
    -- {
    --   "users": [
    --     { "id": "uuid-1", "name": "Alice", "email": "alice@example.com" },
    --     { "id": "uuid-2", "name": "Bob", "email": "bob@example.com" }
    --   ],
    --   "cart": { "items": [], "total": 0 },
    --   "lastAction": "POST /users"
    -- }
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_mock ON sessions (mock_instance_id);
CREATE INDEX idx_sessions_token ON sessions (session_token);
CREATE INDEX idx_sessions_expires ON sessions (expires_at) WHERE expires_at IS NOT NULL;
```

---

## AI Data Generation Cache

```sql
CREATE TABLE ai_cache (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Cache key is a hash of: schema JSON + generation parameters
    cache_key       VARCHAR(128) NOT NULL UNIQUE,
    -- The schema definition that was sent to the AI
    schema_input    JSONB NOT NULL,
    -- Generated data (can be a single object or array of examples)
    generated_data  JSONB NOT NULL,
    -- Metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "model": "claude-sonnet-4-20250514",
    --   "promptTokens": 245,
    --   "completionTokens": 189,
    --   "generationTimeMs": 1200,
    --   "fieldNames": ["id", "name", "email", "createdAt"],
    --   "schemaPath": "#/components/schemas/User"
    -- }
    hit_count       INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_used_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_cache_key ON ai_cache (cache_key);
CREATE INDEX idx_ai_cache_lru ON ai_cache (last_used_at);
```

---

## Request Logging

```sql
CREATE TABLE request_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mock_instance_id UUID NOT NULL REFERENCES mock_instances(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES sessions(id) ON DELETE SET NULL,
    -- Request and response stored as JSONB for maximum flexibility
    request         JSONB NOT NULL,
    -- request example:
    -- {
    --   "method": "POST",
    --   "path": "/users",
    --   "headers": { "content-type": "application/json", "authorization": "Bearer xxx" },
    --   "query": {},
    --   "body": { "name": "Alice", "email": "alice@example.com" }
    -- }
    response        JSONB NOT NULL,
    -- response example:
    -- {
    --   "status": 201,
    --   "headers": { "content-type": "application/json" },
    --   "body": { "id": "uuid-1", "name": "Alice", "email": "alice@example.com" },
    --   "source": "ai_generated"
    -- }
    -- Matching metadata
    matched_operation VARCHAR(255),          -- operationId or 'path:method' string
    match_result    VARCHAR(20) NOT NULL,
    validation_errors JSONB,
    duration_ms     INT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_request_logs_mock ON request_logs (mock_instance_id);
CREATE INDEX idx_request_logs_created ON request_logs (mock_instance_id, created_at DESC);
CREATE INDEX idx_request_logs_session ON request_logs (session_id) WHERE session_id IS NOT NULL;
CREATE INDEX idx_request_logs_match ON request_logs (match_result);
-- GIN index for searching within request/response bodies
CREATE INDEX idx_request_logs_request_gin ON request_logs USING GIN (request jsonb_path_ops);
```

---

## Contract Drift Detection

```sql
CREATE TABLE drift_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mock_instance_id UUID NOT NULL REFERENCES mock_instances(id) ON DELETE CASCADE,
    request_log_id  UUID REFERENCES request_logs(id) ON DELETE SET NULL,
    -- Drift details as a structured JSONB document
    drift           JSONB NOT NULL,
    -- drift example:
    -- {
    --   "type": "field_type_mismatch",
    --   "severity": "breaking",
    --   "operationId": "getUser",
    --   "path": "GET /users/{userId}",
    --   "jsonPath": "$.response.body.age",
    --   "expected": { "type": "integer" },
    --   "actual": { "type": "string", "value": "25" },
    --   "specRef": "#/components/schemas/User/properties/age",
    --   "message": "Field 'age' expected integer but received string '25'"
    -- }
    severity        VARCHAR(20) NOT NULL,   -- extracted for indexing
    drift_type      VARCHAR(50) NOT NULL,   -- extracted for indexing
    resolved        BOOLEAN NOT NULL DEFAULT false,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drift_mock ON drift_events (mock_instance_id);
CREATE INDEX idx_drift_severity ON drift_events (severity);
CREATE INDEX idx_drift_type ON drift_events (drift_type);
CREATE INDEX idx_drift_unresolved ON drift_events (mock_instance_id) WHERE resolved = false;
CREATE INDEX idx_drift_created ON drift_events (created_at DESC);
-- GIN index for searching drift details
CREATE INDEX idx_drift_details_gin ON drift_events USING GIN (drift jsonb_path_ops);
```

---

## Spec Gap Analysis

```sql
CREATE TABLE spec_gaps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id         UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    gap             JSONB NOT NULL,
    -- gap example:
    -- {
    --   "type": "missing_error_response",
    --   "severity": "warning",
    --   "operationId": "createUser",
    --   "path": "POST /users",
    --   "description": "Operation defines 201 and 400 responses but missing 409 (Conflict) for duplicate email",
    --   "suggestedFix": {
    --     "statusCode": "409",
    --     "description": "Conflict - resource already exists",
    --     "schema": { "$ref": "#/components/schemas/Error" }
    --   },
    --   "autoFixable": true
    -- }
    gap_type        VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spec_gaps_spec ON spec_gaps (spec_id);
CREATE INDEX idx_spec_gaps_type ON spec_gaps (gap_type);
CREATE INDEX idx_spec_gaps_unresolved ON spec_gaps (spec_id) WHERE resolved = false;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | tenants, users |
| Specification | 1 | specs (with JSONB parsed_spec) |
| Operation Index | 1 | operation_index (materialized projection) |
| Mock Configuration | 3 | mock_instances, mock_rules, error_templates |
| Sessions & Scenarios | 2 | scenarios, sessions |
| AI Cache | 1 | ai_cache |
| Request Logging | 1 | request_logs |
| Contract Drift | 1 | drift_events |
| Spec Gaps | 1 | spec_gaps |
| **Total** | **13** | Less than half the normalized model |

---

## Key Design Decisions

1. **Parsed spec stored as a single JSONB column.** The entire resolved OpenAPI document lives in `specs.parsed_spec`. This means importing a spec is a single INSERT — no multi-table transaction required. The GIN index with `jsonb_path_ops` supports fast containment queries like `WHERE parsed_spec @> '{"components":{"schemas":{"User":{}}}}'`.

2. **Operation index as a materialized projection.** The `operation_index` table is rebuilt on spec import. It extracts the minimal fields needed for fast runtime request matching (path pattern, method, compiled regex) while keeping the full operation definition in JSONB. This gives the runtime the speed of relational lookups without requiring full normalization:
   ```sql
   -- Fast request matching at runtime
   SELECT operation_def, path_params
   FROM operation_index
   WHERE spec_id = $1
     AND http_method = $2
     AND $3 ~ path_regex
   ORDER BY length(path_pattern) DESC
   LIMIT 1;
   ```

3. **Mock configuration in JSONB.** The `mock_instances.config` column stores all mock settings as a single JSONB object. This avoids having to add a new column every time a configuration option is introduced. The application validates the config against a TypeScript/JSON Schema interface at write time.

4. **State machine definitions are JSONB documents.** Scenario state machines are inherently tree/graph-shaped structures that map naturally to JSON. Storing them as JSONB preserves the structure as authored (potentially by an AI from a natural language description) without requiring decomposition into `states` and `transitions` tables.

5. **Request and response bodies stored as JSONB in the log.** This enables powerful diagnostic queries using PostgreSQL's JSON operators:
   ```sql
   -- Find all requests where the user tried to create a duplicate email
   SELECT id, request->'body'->>'email', response->>'status'
   FROM request_logs
   WHERE mock_instance_id = $1
     AND request->>'method' = 'POST'
     AND request->>'path' = '/users'
     AND (response->>'status')::int = 409;
   ```

6. **Drift event details in JSONB with extracted scalar columns for indexing.** The `severity` and `drift_type` columns are extracted from the JSONB `drift` document and stored as regular columns. This provides the best of both worlds: flexible structured data in JSONB with fast B-tree index scans on the most-filtered columns.

7. **No relational decomposition of schemas.** Unlike Suggestion 1, this model does not attempt to normalize OpenAPI Schema Objects into `schemas`, `schema_properties`, and `schema_compositions` tables. The recursive, polymorphic nature of JSON Schema (allOf, oneOf, anyOf, discriminators, recursive $refs) is stored naturally in JSONB. The trade-off is that queries like "find all properties of type string with format email across all specs" require JSONB path queries rather than simple WHERE clauses.
