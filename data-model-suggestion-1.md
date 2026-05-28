# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: OpenAPI Mock Server · Created: 2026-05-11

## Philosophy

This model decomposes every concept in the mock server domain into its own dedicated table with strict foreign key relationships. The parsed OpenAPI specification is fully decomposed into relational entities: specs, paths, operations, parameters, schemas, responses, and examples each live in their own table. Mock configuration, session state, request logs, generated data cache, and contract drift events are all first-class relational citizens.

This approach mirrors how enterprise API management platforms (Kong, Apigee, AWS API Gateway) model their internal configuration: every resource is independently addressable, queryable, and auditable. It is the natural fit when the mock server is deployed as a **multi-tenant hosted service** where multiple teams share infrastructure, administrators need to query across all specs, and the system must support complex cross-entity queries like "show me all operations across all specs that return a 429 status code."

The trade-off is a higher table count and more complex write paths. Importing a large OpenAPI spec requires inserting into 10+ tables in a transaction. But the read-side benefits — arbitrary SQL queries, standard tooling, straightforward indexing — make this the most maintainable choice for teams with relational database expertise.

**Best for:** Multi-tenant hosted mock server deployments where queryability, multi-user administration, and relational integrity matter more than write throughput.

**Trade-offs:**
- **Pro:** Full referential integrity; every entity independently queryable via standard SQL
- **Pro:** Well-understood by most engineering teams; extensive tooling ecosystem
- **Pro:** Natural fit for multi-tenant SaaS with row-level security
- **Pro:** Schema evolution via standard migrations (Flyway, Alembic, Knex)
- **Con:** High table count (~25-30 tables); importing a single OpenAPI spec touches many tables
- **Con:** OpenAPI's polymorphic constructs (oneOf, anyOf, discriminator) are awkward to normalize
- **Con:** Schema objects are recursive (schemas reference schemas); requires recursive CTEs or application-level tree walking
- **Con:** More rigid; adding new OpenAPI keywords requires migration

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.1.0 / 3.2.0 | Spec, path, operation, parameter, schema, and response tables mirror the OAS object model |
| JSON Schema 2020-12 | Schema table stores JSON Schema keywords as typed columns; `$ref` resolution tracked via `schema_refs` |
| RFC 7807 (Problem Details) | Error template table stores RFC 7807-conformant error structures for generated error responses |
| RFC 9110 (HTTP Semantics) | HTTP method and status code columns use standard values; content negotiation modeled explicitly |
| ISO 8601 | All timestamps use `TIMESTAMPTZ`; duration fields (latency simulation) use `INTERVAL` |

---

## Tenant & Identity Management

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,  -- URL-safe identifier
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, pro, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',   -- tenant-level config overrides
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants (slug);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- owner, admin, member, viewer
    api_key_hash    VARCHAR(128),  -- bcrypt hash of personal API key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users (tenant_id);
CREATE INDEX idx_users_api_key ON users (api_key_hash) WHERE api_key_hash IS NOT NULL;
```

---

## Specification Management

```sql
CREATE TABLE specs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(50),           -- spec version from info.version
    openapi_version VARCHAR(10) NOT NULL,  -- '3.0.3', '3.1.0', '3.2.0', '2.0'
    format          VARCHAR(10) NOT NULL DEFAULT 'yaml',  -- yaml, json
    raw_content     TEXT NOT NULL,          -- original spec text
    info_title      VARCHAR(500),
    info_description TEXT,
    base_url        VARCHAR(2048),         -- derived from servers[0].url
    checksum        VARCHAR(64) NOT NULL,  -- SHA-256 of raw_content for change detection
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, archived, error
    parse_errors    JSONB,                 -- array of parse error objects if status='error'
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_specs_tenant ON specs (tenant_id);
CREATE INDEX idx_specs_checksum ON specs (checksum);
CREATE INDEX idx_specs_status ON specs (tenant_id, status);

CREATE TABLE spec_servers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id         UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    url             VARCHAR(2048) NOT NULL,
    description     TEXT,
    variables       JSONB,  -- server variable definitions
    ordinal         INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spec_servers_spec ON spec_servers (spec_id);

CREATE TABLE spec_tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id         UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    ordinal         INT NOT NULL DEFAULT 0,
    UNIQUE (spec_id, name)
);

CREATE INDEX idx_spec_tags_spec ON spec_tags (spec_id);
```

---

## Path & Operation Modeling

```sql
CREATE TABLE paths (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id         UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    path_pattern    VARCHAR(2048) NOT NULL,  -- e.g., '/users/{userId}/orders'
    summary         TEXT,
    description     TEXT,
    UNIQUE (spec_id, path_pattern)
);

CREATE INDEX idx_paths_spec ON paths (spec_id);

CREATE TABLE operations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    path_id         UUID NOT NULL REFERENCES paths(id) ON DELETE CASCADE,
    spec_id         UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    http_method     VARCHAR(10) NOT NULL,  -- GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS, TRACE
    operation_id    VARCHAR(255),          -- operationId from spec
    summary         TEXT,
    description     TEXT,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    ordinal         INT NOT NULL DEFAULT 0,
    UNIQUE (path_id, http_method)
);

CREATE INDEX idx_operations_spec ON operations (spec_id);
CREATE INDEX idx_operations_path ON operations (path_id);
CREATE INDEX idx_operations_operation_id ON operations (operation_id);

CREATE TABLE operation_tags (
    operation_id    UUID NOT NULL REFERENCES operations(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES spec_tags(id) ON DELETE CASCADE,
    PRIMARY KEY (operation_id, tag_id)
);

CREATE TABLE parameters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id    UUID REFERENCES operations(id) ON DELETE CASCADE,
    path_id         UUID REFERENCES paths(id) ON DELETE CASCADE,  -- path-level params
    name            VARCHAR(255) NOT NULL,
    location        VARCHAR(10) NOT NULL,   -- path, query, header, cookie
    description     TEXT,
    required        BOOLEAN NOT NULL DEFAULT false,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    schema_id       UUID REFERENCES schemas(id),  -- FK to schema table
    style           VARCHAR(20),            -- simple, form, label, matrix, etc.
    explode         BOOLEAN,
    example         JSONB
);

CREATE INDEX idx_parameters_operation ON parameters (operation_id);
CREATE INDEX idx_parameters_path ON parameters (path_id);
```

---

## Schema Modeling

```sql
CREATE TABLE schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id         UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    ref_path        VARCHAR(1024),          -- e.g., '#/components/schemas/User'
    schema_name     VARCHAR(255),           -- component name if in components/schemas
    schema_type     VARCHAR(20),            -- object, array, string, number, integer, boolean, null
    format          VARCHAR(50),            -- date-time, email, uri, uuid, int32, int64, etc.
    title           VARCHAR(500),
    description     TEXT,
    nullable        BOOLEAN NOT NULL DEFAULT false,
    read_only       BOOLEAN NOT NULL DEFAULT false,
    write_only      BOOLEAN NOT NULL DEFAULT false,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    -- String constraints
    min_length      INT,
    max_length      INT,
    pattern         VARCHAR(1024),
    -- Numeric constraints
    minimum         NUMERIC,
    maximum         NUMERIC,
    exclusive_minimum BOOLEAN,
    exclusive_maximum BOOLEAN,
    multiple_of     NUMERIC,
    -- Array constraints
    min_items       INT,
    max_items       INT,
    unique_items    BOOLEAN,
    items_schema_id UUID REFERENCES schemas(id),  -- for array items
    -- Object constraints
    min_properties  INT,
    max_properties  INT,
    additional_properties BOOLEAN DEFAULT true,
    -- Enum values
    enum_values     JSONB,  -- JSON array of allowed values
    -- Default and examples
    default_value   JSONB,
    example         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schemas_spec ON schemas (spec_id);
CREATE INDEX idx_schemas_ref ON schemas (ref_path);
CREATE INDEX idx_schemas_name ON schemas (spec_id, schema_name);

-- Properties within object schemas
CREATE TABLE schema_properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_schema_id UUID NOT NULL REFERENCES schemas(id) ON DELETE CASCADE,
    property_name   VARCHAR(255) NOT NULL,
    schema_id       UUID NOT NULL REFERENCES schemas(id) ON DELETE CASCADE,
    required        BOOLEAN NOT NULL DEFAULT false,
    ordinal         INT NOT NULL DEFAULT 0,
    UNIQUE (parent_schema_id, property_name)
);

CREATE INDEX idx_schema_props_parent ON schema_properties (parent_schema_id);

-- Composition: allOf, oneOf, anyOf
CREATE TABLE schema_compositions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_schema_id UUID NOT NULL REFERENCES schemas(id) ON DELETE CASCADE,
    composition_type VARCHAR(10) NOT NULL,  -- allOf, oneOf, anyOf
    child_schema_id  UUID NOT NULL REFERENCES schemas(id) ON DELETE CASCADE,
    ordinal         INT NOT NULL DEFAULT 0
);

CREATE INDEX idx_schema_comp_parent ON schema_compositions (parent_schema_id);

-- Discriminator definitions
CREATE TABLE schema_discriminators (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schema_id       UUID NOT NULL REFERENCES schemas(id) ON DELETE CASCADE,
    property_name   VARCHAR(255) NOT NULL,
    mapping         JSONB  -- { "dog": "#/components/schemas/Dog", ... }
);
```

---

## Response & Request Body Modeling

```sql
CREATE TABLE responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id    UUID NOT NULL REFERENCES operations(id) ON DELETE CASCADE,
    status_code     VARCHAR(5) NOT NULL,    -- '200', '404', '5XX', 'default'
    description     TEXT,
    UNIQUE (operation_id, status_code)
);

CREATE INDEX idx_responses_operation ON responses (operation_id);

CREATE TABLE response_contents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    response_id     UUID NOT NULL REFERENCES responses(id) ON DELETE CASCADE,
    media_type      VARCHAR(255) NOT NULL,  -- application/json, text/xml, etc.
    schema_id       UUID REFERENCES schemas(id),
    example         JSONB,
    examples        JSONB,  -- named examples map
    UNIQUE (response_id, media_type)
);

CREATE INDEX idx_response_contents_response ON response_contents (response_id);

CREATE TABLE response_headers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    response_id     UUID NOT NULL REFERENCES responses(id) ON DELETE CASCADE,
    header_name     VARCHAR(255) NOT NULL,
    description     TEXT,
    schema_id       UUID REFERENCES schemas(id),
    required        BOOLEAN NOT NULL DEFAULT false,
    UNIQUE (response_id, header_name)
);

CREATE TABLE request_bodies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operation_id    UUID NOT NULL REFERENCES operations(id) ON DELETE CASCADE,
    description     TEXT,
    required        BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_request_bodies_operation ON request_bodies (operation_id);

CREATE TABLE request_body_contents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_body_id UUID NOT NULL REFERENCES request_bodies(id) ON DELETE CASCADE,
    media_type      VARCHAR(255) NOT NULL,
    schema_id       UUID REFERENCES schemas(id),
    example         JSONB,
    examples        JSONB,
    UNIQUE (request_body_id, media_type)
);
```

---

## Mock Configuration & Rules

```sql
CREATE TABLE mock_instances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    spec_id         UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    base_path       VARCHAR(512) NOT NULL DEFAULT '/',
    port            INT,
    status          VARCHAR(20) NOT NULL DEFAULT 'stopped',  -- running, stopped, error
    latency_ms      INT DEFAULT 0,           -- global artificial latency
    error_rate      DECIMAL(5,4) DEFAULT 0,  -- 0.0000 to 1.0000; probability of injecting errors
    cors_enabled    BOOLEAN NOT NULL DEFAULT true,
    validate_requests BOOLEAN NOT NULL DEFAULT true,
    ai_data_enabled BOOLEAN NOT NULL DEFAULT true,  -- use AI for semantic data generation
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mock_instances_tenant ON mock_instances (tenant_id);
CREATE INDEX idx_mock_instances_spec ON mock_instances (spec_id);
CREATE INDEX idx_mock_instances_status ON mock_instances (status);

-- Override rules for specific operations
CREATE TABLE mock_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mock_instance_id UUID NOT NULL REFERENCES mock_instances(id) ON DELETE CASCADE,
    operation_id    UUID REFERENCES operations(id) ON DELETE SET NULL,
    name            VARCHAR(255),
    priority        INT NOT NULL DEFAULT 0,  -- higher = checked first
    -- Request matching conditions
    match_headers   JSONB,   -- { "Authorization": { "matches": "Bearer .*" } }
    match_query     JSONB,   -- { "status": { "equalTo": "active" } }
    match_body      JSONB,   -- JSONPath or body matching conditions
    -- Response override
    response_status INT,
    response_headers JSONB,
    response_body   JSONB,
    response_delay_ms INT,   -- per-rule latency override
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mock_rules_instance ON mock_rules (mock_instance_id);
CREATE INDEX idx_mock_rules_operation ON mock_rules (operation_id);
CREATE INDEX idx_mock_rules_priority ON mock_rules (mock_instance_id, priority DESC);

-- Error templates (RFC 7807 Problem Details)
CREATE TABLE error_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenants(id) ON DELETE CASCADE,  -- NULL = system-wide
    status_code     INT NOT NULL,
    type_uri        VARCHAR(2048),          -- RFC 7807 type URI
    title           VARCHAR(500) NOT NULL,
    detail_template VARCHAR(2048),          -- Handlebars template for detail field
    owasp_category  VARCHAR(50),            -- e.g., 'API1:2023', 'API2:2023'
    headers         JSONB,
    body_template   JSONB NOT NULL,         -- RFC 7807 JSON structure template
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_error_templates_status ON error_templates (status_code);
CREATE INDEX idx_error_templates_owasp ON error_templates (owasp_category) WHERE owasp_category IS NOT NULL;
```

---

## Stateful Session Management

```sql
CREATE TABLE sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mock_instance_id UUID NOT NULL REFERENCES mock_instances(id) ON DELETE CASCADE,
    session_token   VARCHAR(128) NOT NULL UNIQUE,
    state_name      VARCHAR(255) NOT NULL DEFAULT 'initial',  -- current state in state machine
    state_data      JSONB NOT NULL DEFAULT '{}',  -- accumulated session state (created resources, etc.)
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_mock ON sessions (mock_instance_id);
CREATE INDEX idx_sessions_token ON sessions (session_token);
CREATE INDEX idx_sessions_expires ON sessions (expires_at) WHERE expires_at IS NOT NULL;

-- Scenario definitions (natural language → state machine)
CREATE TABLE scenarios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mock_instance_id UUID NOT NULL REFERENCES mock_instances(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,                   -- natural language scenario description
    initial_state   VARCHAR(255) NOT NULL DEFAULT 'initial',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scenarios_mock ON scenarios (mock_instance_id);

-- State transitions within a scenario
CREATE TABLE scenario_transitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scenario_id     UUID NOT NULL REFERENCES scenarios(id) ON DELETE CASCADE,
    from_state      VARCHAR(255) NOT NULL,
    to_state        VARCHAR(255) NOT NULL,
    trigger_method  VARCHAR(10) NOT NULL,   -- HTTP method that triggers this transition
    trigger_path    VARCHAR(2048) NOT NULL,  -- path pattern
    trigger_conditions JSONB,               -- additional conditions (body content, headers)
    state_mutations JSONB,                  -- how to modify state_data on transition
    response_override JSONB,                -- custom response for this transition
    ordinal         INT NOT NULL DEFAULT 0
);

CREATE INDEX idx_transitions_scenario ON scenario_transitions (scenario_id);
CREATE INDEX idx_transitions_from ON scenario_transitions (scenario_id, from_state);
```

---

## AI Data Generation Cache

```sql
CREATE TABLE ai_generation_cache (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schema_id       UUID NOT NULL REFERENCES schemas(id) ON DELETE CASCADE,
    cache_key       VARCHAR(128) NOT NULL,  -- SHA-256 of schema + generation params
    generated_data  JSONB NOT NULL,         -- the AI-generated example data
    model_used      VARCHAR(100),           -- e.g., 'claude-sonnet-4-20250514'
    prompt_tokens   INT,
    completion_tokens INT,
    generation_time_ms INT,
    hit_count       INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_used_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ
);

CREATE UNIQUE INDEX idx_ai_cache_key ON ai_generation_cache (cache_key);
CREATE INDEX idx_ai_cache_schema ON ai_generation_cache (schema_id);
CREATE INDEX idx_ai_cache_expires ON ai_generation_cache (expires_at) WHERE expires_at IS NOT NULL;
CREATE INDEX idx_ai_cache_lru ON ai_generation_cache (last_used_at);
```

---

## Request Logging

```sql
CREATE TABLE request_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mock_instance_id UUID NOT NULL REFERENCES mock_instances(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES sessions(id) ON DELETE SET NULL,
    -- Request details
    request_method  VARCHAR(10) NOT NULL,
    request_path    VARCHAR(2048) NOT NULL,
    request_headers JSONB,
    request_query   JSONB,
    request_body    TEXT,
    -- Matching details
    matched_operation_id UUID REFERENCES operations(id) ON DELETE SET NULL,
    matched_rule_id UUID REFERENCES mock_rules(id) ON DELETE SET NULL,
    match_result    VARCHAR(20) NOT NULL,    -- matched, unmatched, validation_error
    validation_errors JSONB,                 -- array of validation error objects
    -- Response details
    response_status INT NOT NULL,
    response_headers JSONB,
    response_body   TEXT,
    response_source VARCHAR(30) NOT NULL,    -- spec_example, ai_generated, rule_override, error_template
    -- Timing
    duration_ms     INT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partition by month for high-volume deployments
-- CREATE TABLE request_logs (...) PARTITION BY RANGE (created_at);

CREATE INDEX idx_request_logs_mock ON request_logs (mock_instance_id);
CREATE INDEX idx_request_logs_session ON request_logs (session_id) WHERE session_id IS NOT NULL;
CREATE INDEX idx_request_logs_operation ON request_logs (matched_operation_id) WHERE matched_operation_id IS NOT NULL;
CREATE INDEX idx_request_logs_created ON request_logs (mock_instance_id, created_at DESC);
CREATE INDEX idx_request_logs_match ON request_logs (match_result);
```

---

## Contract Drift Detection

```sql
CREATE TABLE proxy_targets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mock_instance_id UUID NOT NULL REFERENCES mock_instances(id) ON DELETE CASCADE,
    upstream_url    VARCHAR(2048) NOT NULL,
    auth_header     VARCHAR(2048),          -- encrypted auth token for upstream
    enabled         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proxy_targets_mock ON proxy_targets (mock_instance_id);

CREATE TABLE drift_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mock_instance_id UUID NOT NULL REFERENCES mock_instances(id) ON DELETE CASCADE,
    operation_id    UUID REFERENCES operations(id) ON DELETE SET NULL,
    drift_type      VARCHAR(50) NOT NULL,   -- field_type_mismatch, missing_field, extra_field,
                                            -- status_code_mismatch, header_mismatch, schema_violation
    severity        VARCHAR(20) NOT NULL,   -- breaking, warning, info
    expected_value  TEXT,
    actual_value    TEXT,
    json_path       VARCHAR(1024),          -- JSONPath to the divergent field
    spec_ref        VARCHAR(1024),          -- reference to the relevant spec section
    request_log_id  UUID REFERENCES request_logs(id) ON DELETE SET NULL,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    resolved_at     TIMESTAMPTZ,
    resolved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drift_events_mock ON drift_events (mock_instance_id);
CREATE INDEX idx_drift_events_operation ON drift_events (operation_id);
CREATE INDEX idx_drift_events_type ON drift_events (drift_type);
CREATE INDEX idx_drift_events_severity ON drift_events (severity);
CREATE INDEX idx_drift_events_unresolved ON drift_events (mock_instance_id, resolved) WHERE resolved = false;
CREATE INDEX idx_drift_events_created ON drift_events (created_at DESC);
```

---

## Spec Gap Detection

```sql
CREATE TABLE spec_gaps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id         UUID NOT NULL REFERENCES specs(id) ON DELETE CASCADE,
    operation_id    UUID REFERENCES operations(id) ON DELETE SET NULL,
    gap_type        VARCHAR(50) NOT NULL,   -- missing_error_response, missing_auth,
                                            -- undocumented_query_param, naming_convention,
                                            -- missing_pagination, incomplete_schema
    severity        VARCHAR(20) NOT NULL,   -- critical, warning, suggestion
    description     TEXT NOT NULL,
    suggested_fix   JSONB,                  -- AI-generated suggestion for fixing the gap
    auto_fixable    BOOLEAN NOT NULL DEFAULT false,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spec_gaps_spec ON spec_gaps (spec_id);
CREATE INDEX idx_spec_gaps_type ON spec_gaps (gap_type);
CREATE INDEX idx_spec_gaps_unresolved ON spec_gaps (spec_id, resolved) WHERE resolved = false;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | tenants, users |
| Specification | 3 | specs, spec_servers, spec_tags |
| Paths & Operations | 4 | paths, operations, operation_tags, parameters |
| Schemas | 4 | schemas, schema_properties, schema_compositions, schema_discriminators |
| Responses & Request Bodies | 5 | responses, response_contents, response_headers, request_bodies, request_body_contents |
| Mock Configuration | 3 | mock_instances, mock_rules, error_templates |
| Sessions & Scenarios | 3 | sessions, scenarios, scenario_transitions |
| AI Cache | 1 | ai_generation_cache |
| Request Logging | 1 | request_logs |
| Contract Drift | 2 | proxy_targets, drift_events |
| Spec Gaps | 1 | spec_gaps |
| **Total** | **29** | |

---

## Key Design Decisions

1. **Full decomposition of OpenAPI spec objects into relational tables.** The `schemas`, `schema_properties`, and `schema_compositions` tables mirror the OpenAPI Schema Object hierarchy. This enables SQL queries like "find all schemas that use `format: email`" or "list all operations returning an array of User objects" — queries that are impossible when the spec is stored as a single JSONB blob.

2. **Recursive schema references via self-referencing foreign keys.** The `schemas.items_schema_id` and `schema_properties.schema_id` columns point back to `schemas`, modeling the recursive nature of JSON Schema. Querying schema trees requires recursive CTEs:
   ```sql
   WITH RECURSIVE schema_tree AS (
       SELECT id, schema_name, schema_type, 0 AS depth
       FROM schemas WHERE id = $1
       UNION ALL
       SELECT s.id, s.schema_name, s.schema_type, st.depth + 1
       FROM schemas s
       JOIN schema_properties sp ON sp.schema_id = s.id
       JOIN schema_tree st ON sp.parent_schema_id = st.id
   )
   SELECT * FROM schema_tree;
   ```

3. **Session state stored as JSONB within a relational session table.** While the session row itself is relational (with proper foreign keys and indexes), the accumulated state data is JSONB because its shape depends entirely on the API being mocked. A session for a user-management API accumulates `{"users": [...]}` while a session for an e-commerce API accumulates `{"cart": {...}, "orders": [...]}`.

4. **AI generation cache keyed by schema hash.** The `cache_key` is a SHA-256 hash of the schema definition plus generation parameters, ensuring that identical schemas produce cache hits regardless of which spec they appear in. The `hit_count` and `last_used_at` columns support LRU eviction.

5. **Request logs designed for time-partitioning.** The `request_logs` table includes a comment showing how to convert it to a partitioned table. In production, high-volume mock servers can generate millions of log entries; monthly partitioning enables efficient pruning of old data.

6. **Drift events link back to request logs.** Each `drift_event` references the specific `request_log` that triggered it, providing a full audit trail from "we detected a drift" back to "here is the exact request and response that diverged."

7. **Multi-tenant via `tenant_id` foreign keys.** Every user-facing table includes a `tenant_id` column, enabling PostgreSQL Row Level Security policies for tenant isolation without schema-per-tenant complexity.
