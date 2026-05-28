# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: OpenAPI Mock Server · Created: 2026-05-11

## Philosophy

This model treats every interaction with the mock server as an immutable event appended to a time-ordered log. The event store is the single source of truth. Materialized read models (views) are derived from the event stream and can be rebuilt at any time by replaying events from the beginning. This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from projections optimized for specific query patterns.

The core insight is that a mock server's most valuable output is not the mock responses themselves, but the **history of what happened**: which requests arrived, how they matched, what data was generated, when the spec drifted from reality, and how sessions evolved over time. An event-sourced model captures this history natively — every state change is recorded, nothing is overwritten, and the system supports temporal queries ("what was the session state at 14:32?") and replay ("re-process all requests from Tuesday's test run against the updated spec").

This pattern is used in production by EventStoreDB, Apache Kafka (as a commit log), AWS CloudTrail (audit events), and financial trading systems where audit trails are non-negotiable. For a mock server focused on contract drift detection and AI-driven analytics, event sourcing provides the richest possible data foundation.

**Best for:** Deployments where contract drift detection, full audit trails, temporal queries, analytics on mock usage patterns, and AI-driven insights from request history are primary value propositions.

**Trade-offs:**
- **Pro:** Complete audit trail — every interaction is permanently recorded and immutable
- **Pro:** Temporal queries are natural: "show me all drift events between 2pm and 3pm last Tuesday"
- **Pro:** Event replay enables powerful debugging: replay a failed test session against a fixed spec
- **Pro:** CQRS allows independently optimized read models for different consumers (dashboard, API, analytics)
- **Pro:** Natural fit for streaming architectures; events can be piped to Kafka, webhooks, or analytics
- **Con:** Higher storage requirements — events are never deleted (only compacted)
- **Con:** Read model staleness — projections may lag behind the event store
- **Con:** Eventual consistency adds complexity for clients expecting immediate read-after-write
- **Con:** More complex to implement correctly; requires understanding of event versioning and schema evolution
- **Con:** Simple queries require maintaining materialized views; no ad-hoc SQL against a single table

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.1.0 / 3.2.0 | Spec imported/updated events carry the full OAS document; spec state derived from event replay |
| JSON Schema 2020-12 | Schema definitions embedded in spec events; schema changes tracked as discrete events |
| RFC 7807 (Problem Details) | Error response events include RFC 7807 structures in their payload |
| OCSF (Open Cybersecurity Schema Framework) | Event structure inspired by OCSF's activity/category/class taxonomy for structured event logging |
| CloudEvents 1.0 (CNCF) | Event envelope format follows CloudEvents specification for interoperability |
| ISO 8601 | All timestamps use `TIMESTAMPTZ`; event ordering guaranteed by `sequence_number` |

---

## Event Store (Source of Truth)

```sql
-- The single source of truth. All state is derived from this table.
CREATE TABLE events (
    -- Ordering and identity
    sequence_number BIGSERIAL PRIMARY KEY,  -- global total order
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    -- Aggregate identification
    aggregate_type  VARCHAR(50) NOT NULL,   -- 'spec', 'mock_instance', 'session', 'drift_monitor'
    aggregate_id    UUID NOT NULL,          -- ID of the aggregate this event belongs to
    -- Event classification (inspired by CloudEvents + OCSF)
    event_type      VARCHAR(100) NOT NULL,  -- e.g., 'spec.imported', 'request.received', 'drift.detected'
    event_version   INT NOT NULL DEFAULT 1, -- schema version of the event payload
    -- Tenant context
    tenant_id       UUID NOT NULL,
    -- The event payload — all domain data lives here
    payload         JSONB NOT NULL,
    -- Metadata
    correlation_id  UUID,                   -- links related events (e.g., request → response → drift)
    causation_id    UUID,                   -- the event that caused this event
    actor_id        UUID,                   -- user or system that triggered the event
    source          VARCHAR(255),           -- 'cli', 'api', 'proxy', 'scheduler', 'ai_engine'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Immutability enforced: no UPDATE or DELETE allowed (enforce via REVOKE or RLS policy)
-- REVOKE UPDATE, DELETE ON events FROM app_user;

-- Primary query patterns
CREATE INDEX idx_events_aggregate ON events (aggregate_type, aggregate_id, sequence_number);
CREATE INDEX idx_events_type ON events (event_type, created_at);
CREATE INDEX idx_events_tenant ON events (tenant_id, created_at);
CREATE INDEX idx_events_correlation ON events (correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_events_created ON events (created_at);
-- GIN index for payload queries
CREATE INDEX idx_events_payload_gin ON events USING GIN (payload jsonb_path_ops);

-- Partition by month for scalability
-- In production, convert to:
-- CREATE TABLE events (...) PARTITION BY RANGE (created_at);
-- CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

---

## Event Type Catalog

The following event types are emitted by the system. Each event type has a versioned payload schema.

### Specification Events

```
spec.imported          — A new OpenAPI spec was loaded
spec.updated           — An existing spec was re-imported with changes
spec.archived          — A spec was deactivated
spec.parse_failed      — Spec parsing failed with errors
spec.gaps_detected     — AI identified gaps in the spec
```

**Example payload for `spec.imported`:**
```json
{
  "specId": "uuid-spec-1",
  "name": "Pet Store API",
  "openapiVersion": "3.1.0",
  "format": "yaml",
  "checksum": "sha256:abc123...",
  "infoTitle": "Pet Store API",
  "infoVersion": "1.0.0",
  "baseUrl": "https://api.petstore.example.com/v1",
  "operationCount": 12,
  "schemaCount": 8,
  "parsedSpec": { "...full parsed spec..." },
  "rawContentSize": 4523
}
```

### Mock Instance Events

```
mock.created           — A new mock instance was configured
mock.started           — Mock server started listening
mock.stopped           — Mock server stopped
mock.config_changed    — Configuration was updated
mock.rule_added        — A mock rule/override was added
mock.rule_removed      — A mock rule was removed
mock.scenario_added    — A stateful scenario was defined
```

### Request/Response Events

```
request.received       — An HTTP request arrived at the mock
request.matched        — Request was matched to an operation
request.unmatched      — No matching operation found
request.validation_failed — Request failed spec validation
response.generated     — Mock response was generated
response.ai_generated  — AI was used to generate response data
response.rule_applied  — A mock rule override was applied
response.error_generated — An error response (RFC 7807) was generated
```

**Example payload for `request.received` + `response.generated` (correlated):**
```json
// request.received
{
  "mockInstanceId": "uuid-mock-1",
  "sessionId": "uuid-session-1",
  "request": {
    "method": "POST",
    "path": "/users",
    "headers": { "content-type": "application/json" },
    "query": {},
    "body": { "name": "Alice", "email": "alice@example.com" },
    "clientIp": "10.0.0.5"
  },
  "matchedOperationId": "createUser",
  "matchedPath": "/users",
  "matchResult": "matched"
}

// response.generated (causation_id = request.received event id)
{
  "mockInstanceId": "uuid-mock-1",
  "responseStatus": 201,
  "responseHeaders": { "content-type": "application/json" },
  "responseBody": { "id": "uuid-new", "name": "Alice", "email": "alice@example.com" },
  "responseSource": "ai_generated",
  "durationMs": 145,
  "aiCacheHit": false,
  "aiModel": "claude-sonnet-4-20250514",
  "aiTokensUsed": 234
}
```

### Session Events

```
session.created        — A new stateful session was started
session.state_changed  — Session transitioned to a new state
session.data_mutated   — Session state data was modified
session.expired        — Session TTL elapsed
session.destroyed      — Session was manually deleted
```

**Example payload for `session.state_changed`:**
```json
{
  "sessionId": "uuid-session-1",
  "scenarioId": "uuid-scenario-1",
  "previousState": "empty_cart",
  "newState": "has_items",
  "trigger": { "method": "POST", "path": "/cart/items" },
  "stateDelta": {
    "added": { "cart.items[0]": { "id": "item-1", "name": "Widget", "qty": 2 } },
    "removed": {},
    "modified": {}
  }
}
```

### Contract Drift Events

```
drift.detected         — A divergence between spec and real API was found
drift.resolved         — A previously detected drift was marked as resolved
drift.cluster_created  — AI grouped related drift events into a cluster
drift.alert_sent       — A notification (email, webhook, GitHub issue) was sent
```

### AI Events

```
ai.generation_requested — AI data generation was requested
ai.generation_completed — AI returned generated data
ai.generation_cached   — Generated data was stored in cache
ai.cache_hit           — A cached AI generation was reused
ai.spec_gap_analysis   — AI analyzed spec for gaps
```

---

## Read Models (Materialized Projections)

Read models are rebuilt from the event stream. They are disposable — if corrupted or if the schema changes, they can be dropped and rebuilt by replaying events.

### Current Spec State

```sql
CREATE TABLE rm_specs (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    openapi_version VARCHAR(10) NOT NULL,
    parsed_spec     JSONB NOT NULL,
    raw_content     TEXT NOT NULL,
    checksum        VARCHAR(64) NOT NULL,
    info_title      VARCHAR(500),
    info_version    VARCHAR(50),
    base_url        VARCHAR(2048),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    operation_count INT NOT NULL DEFAULT 0,
    schema_count    INT NOT NULL DEFAULT 0,
    last_event_seq  BIGINT NOT NULL,        -- last processed event sequence_number
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_specs_tenant ON rm_specs (tenant_id);
CREATE INDEX idx_rm_specs_status ON rm_specs (tenant_id, status);
```

### Current Mock Instance State

```sql
CREATE TABLE rm_mock_instances (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    spec_id         UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'stopped',
    rule_count      INT NOT NULL DEFAULT 0,
    scenario_count  INT NOT NULL DEFAULT 0,
    total_requests  BIGINT NOT NULL DEFAULT 0,
    last_request_at TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_mocks_tenant ON rm_mock_instances (tenant_id);
CREATE INDEX idx_rm_mocks_spec ON rm_mock_instances (spec_id);
CREATE INDEX idx_rm_mocks_status ON rm_mock_instances (status);
```

### Operation Index (for request matching)

```sql
CREATE TABLE rm_operation_index (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    spec_id         UUID NOT NULL,
    path_pattern    VARCHAR(2048) NOT NULL,
    http_method     VARCHAR(10) NOT NULL,
    operation_id    VARCHAR(255),
    path_regex      VARCHAR(2048),
    path_params     TEXT[],
    operation_def   JSONB NOT NULL,
    has_request_body BOOLEAN NOT NULL DEFAULT false,
    response_codes  TEXT[],
    last_event_seq  BIGINT NOT NULL,
    UNIQUE (spec_id, path_pattern, http_method)
);

CREATE INDEX idx_rm_ops_spec ON rm_operation_index (spec_id);
CREATE INDEX idx_rm_ops_method ON rm_operation_index (http_method);
```

### Active Sessions

```sql
CREATE TABLE rm_sessions (
    id              UUID PRIMARY KEY,
    mock_instance_id UUID NOT NULL,
    scenario_id     UUID,
    session_token   VARCHAR(128) NOT NULL UNIQUE,
    current_state   VARCHAR(255) NOT NULL,
    state_data      JSONB NOT NULL DEFAULT '{}',
    request_count   INT NOT NULL DEFAULT 0,
    transition_count INT NOT NULL DEFAULT 0,
    expires_at      TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_sessions_mock ON rm_sessions (mock_instance_id);
CREATE INDEX idx_rm_sessions_token ON rm_sessions (session_token);
```

### AI Generation Cache

```sql
CREATE TABLE rm_ai_cache (
    cache_key       VARCHAR(128) PRIMARY KEY,
    schema_input    JSONB NOT NULL,
    generated_data  JSONB NOT NULL,
    model_used      VARCHAR(100),
    hit_count       INT NOT NULL DEFAULT 0,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    last_used_at    TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_ai_cache_lru ON rm_ai_cache (last_used_at);
```

### Drift Summary

```sql
CREATE TABLE rm_drift_summary (
    id              UUID PRIMARY KEY,
    mock_instance_id UUID NOT NULL,
    drift_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    operation_id    VARCHAR(255),
    json_path       VARCHAR(1024),
    message         TEXT,
    occurrence_count INT NOT NULL DEFAULT 1,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_at    TIMESTAMPTZ NOT NULL,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    resolved_at     TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL
);

CREATE INDEX idx_rm_drift_mock ON rm_drift_summary (mock_instance_id);
CREATE INDEX idx_rm_drift_unresolved ON rm_drift_summary (mock_instance_id) WHERE resolved = false;
CREATE INDEX idx_rm_drift_severity ON rm_drift_summary (severity);
```

### Request Analytics (Time-Bucketed)

```sql
-- Aggregated request metrics, updated incrementally from request events
CREATE TABLE rm_request_metrics (
    mock_instance_id UUID NOT NULL,
    bucket          TIMESTAMPTZ NOT NULL,   -- truncated to 1-minute buckets
    http_method     VARCHAR(10) NOT NULL,
    path_pattern    VARCHAR(2048),
    -- Counters
    total_requests  INT NOT NULL DEFAULT 0,
    matched_count   INT NOT NULL DEFAULT 0,
    unmatched_count INT NOT NULL DEFAULT 0,
    validation_error_count INT NOT NULL DEFAULT 0,
    -- Latency percentiles
    avg_duration_ms NUMERIC(10,2),
    p50_duration_ms INT,
    p95_duration_ms INT,
    p99_duration_ms INT,
    -- AI generation stats
    ai_generated_count INT NOT NULL DEFAULT 0,
    ai_cache_hit_count INT NOT NULL DEFAULT 0,
    ai_tokens_used  INT NOT NULL DEFAULT 0,
    -- Drift stats
    drift_detected_count INT NOT NULL DEFAULT 0,
    last_event_seq  BIGINT NOT NULL,
    PRIMARY KEY (mock_instance_id, bucket, http_method, path_pattern)
);

CREATE INDEX idx_rm_metrics_mock_time ON rm_request_metrics (mock_instance_id, bucket DESC);
```

---

## Projection Engine

The projection engine reads events from the event store and updates read models. It tracks its position using a checkpoint table:

```sql
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_sequence   BIGINT NOT NULL DEFAULT 0,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'running',  -- running, paused, rebuilding, error
    error_message   TEXT
);

-- Seed with initial checkpoints
INSERT INTO projection_checkpoints (projection_name, last_sequence) VALUES
    ('specs', 0),
    ('mock_instances', 0),
    ('operation_index', 0),
    ('sessions', 0),
    ('ai_cache', 0),
    ('drift_summary', 0),
    ('request_metrics', 0);
```

**Projection rebuild example:**
```sql
-- To rebuild the specs projection from scratch:
UPDATE projection_checkpoints SET last_sequence = 0, status = 'rebuilding'
WHERE projection_name = 'specs';

TRUNCATE rm_specs;

-- Then the projection engine replays all spec.* events:
-- SELECT * FROM events
-- WHERE aggregate_type = 'spec'
--   AND sequence_number > 0
-- ORDER BY sequence_number;
```

---

## Snapshot Support

For aggregates with long event histories, snapshots avoid replaying thousands of events:

```sql
CREATE TABLE snapshots (
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_version INT NOT NULL,
    state           JSONB NOT NULL,         -- serialized aggregate state at this point
    last_event_seq  BIGINT NOT NULL,        -- events up to this sequence are included
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, snapshot_version)
);

CREATE INDEX idx_snapshots_aggregate ON snapshots (aggregate_type, aggregate_id, snapshot_version DESC);

-- To rebuild aggregate state:
-- 1. Load latest snapshot (if any)
-- 2. Replay events after snapshot's last_event_seq
-- 3. Apply events to snapshot state
```

---

## Tenant & Identity (Read Model)

```sql
CREATE TABLE rm_tenants (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_users (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    api_key_hash    VARCHAR(128),
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_rm_users_tenant ON rm_users (tenant_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (the single source of truth, partitionable) |
| Projection Infrastructure | 2 | projection_checkpoints, snapshots |
| Read Model: Tenant & Identity | 2 | rm_tenants, rm_users |
| Read Model: Specifications | 1 | rm_specs |
| Read Model: Operations | 1 | rm_operation_index |
| Read Model: Mock Instances | 1 | rm_mock_instances |
| Read Model: Sessions | 1 | rm_sessions |
| Read Model: AI Cache | 1 | rm_ai_cache |
| Read Model: Drift | 1 | rm_drift_summary |
| Read Model: Analytics | 1 | rm_request_metrics |
| **Total** | **12** | 1 write table + 11 read/infrastructure tables |

---

## Key Design Decisions

1. **Single event store table with JSONB payloads.** All domain events flow into one table, differentiated by `event_type` and `aggregate_type`. This simplifies the write path to a single INSERT and makes cross-aggregate queries trivial. The alternative — one table per event type — adds tables without proportional benefit for a system of this scale.

2. **CloudEvents-inspired envelope structure.** Each event carries `id`, `source`, `event_type`, `created_at`, and `payload` — aligning with the CNCF CloudEvents specification. This means events can be forwarded to external systems (Kafka, webhooks, CloudEvents-compatible consumers) without transformation.

3. **Correlation and causation IDs for request tracing.** A single HTTP request to the mock server generates a chain of events: `request.received` → `request.matched` → `response.ai_generated` → `response.generated`. The `correlation_id` groups them; the `causation_id` establishes the causal chain. This enables queries like:
   ```sql
   -- Trace the full lifecycle of a single mock request
   SELECT event_type, payload, created_at
   FROM events
   WHERE correlation_id = $1
   ORDER BY sequence_number;
   ```

4. **Read models are disposable and rebuildable.** Every `rm_*` table can be dropped and rebuilt by replaying events from the event store. This means schema changes to read models are non-destructive: add the new column, replay events, and the column is populated from historical data. This is uniquely powerful for evolving the analytics and dashboard capabilities over time.

5. **Time-bucketed request metrics.** Rather than querying raw request events for dashboard charts (expensive), the `rm_request_metrics` table pre-aggregates into 1-minute buckets. The projection engine updates these incrementally as new `request.*` events arrive. This supports efficient time-series queries:
   ```sql
   -- Requests per minute for the last hour
   SELECT bucket, total_requests, avg_duration_ms
   FROM rm_request_metrics
   WHERE mock_instance_id = $1
     AND bucket >= now() - INTERVAL '1 hour'
   ORDER BY bucket;
   ```

6. **Snapshots for long-lived sessions.** A stateful mock session that processes thousands of requests would require replaying thousands of events to reconstruct its current state. Snapshots capture the session state periodically, and only events after the snapshot need to be replayed. The snapshot interval is configurable per aggregate type.

7. **Event immutability enforced at the database level.** The `events` table has no UPDATE or DELETE permissions for the application user. Events are append-only. Corrections are modeled as compensating events (e.g., `drift.resolved` compensates for `drift.detected`), preserving the complete history.

8. **No request logging table — request events ARE the log.** Unlike models 1 and 2, there is no separate `request_logs` table. The `request.received` and `response.generated` events in the event store serve as the request log. The `rm_request_metrics` read model provides aggregated analytics. If a developer needs the raw request/response, they query the event store directly with a correlation ID or time range. This eliminates data duplication.

9. **Natural support for event streaming.** Because all state changes flow through the event store, adding real-time features (live dashboard updates, webhook notifications on drift detection, Slack alerts) is a matter of subscribing to the event stream — either via PostgreSQL LISTEN/NOTIFY or by tailing the events table. No additional infrastructure is needed for the notification pipeline.
