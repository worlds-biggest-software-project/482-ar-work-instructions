# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourced architecture with Command Query Responsibility Segregation (CQRS). All state changes are captured as immutable domain events in an append-only event store. Commands are validated and processed by command handlers that emit events. Read-optimized projections are built from the event stream for queries, search, and analytics. This design uses PostgreSQL as the event store with materialized read projections.

## Why This Suits the Domain

AR Work Instructions has several characteristics that make event sourcing compelling:

1. **Audit and compliance are first-class requirements.** Every step execution, approval, and modification must be recorded with timestamps and operator IDs. Event sourcing gives you a complete, immutable audit log by design rather than bolting on audit tables.
2. **Procedure versioning with approval workflows** maps naturally to a sequence of domain events (ProcedureDrafted, StepAdded, ReviewRequested, ReviewApproved, ProcedurePublished).
3. **Offline-first architecture** benefits enormously: devices can record local event streams and replay them upon reconnection, with deterministic conflict resolution based on event ordering.
4. **Anomaly detection** across completion data is easier when you have a full event stream to replay and analyze, rather than querying mutable state.
5. **Adaptive procedures** can use event history to build operator skill profiles dynamically.

## Trade-offs

**Strengths:**
- Perfect audit trail with zero additional effort
- Natural fit for offline-first with event replay and merge
- Temporal queries trivial (reconstruct state at any point in time)
- Decoupled read/write models allow independent optimization
- Event streams feed analytics, anomaly detection, and ML pipelines directly

**Weaknesses:**
- Higher conceptual complexity; steeper learning curve for developers
- Eventually consistent read projections (slight delay between write and read)
- Projection rebuilds can be slow with millions of events
- Schema evolution of events requires careful versioning (upcasting)
- More infrastructure: event store + projection database + possibly message broker
- Spatial anchor data and 3D overlay configurations are state-heavy, not event-heavy

## Event Store Schema

```sql
-- =============================================================
-- EVENT STORE (append-only)
-- =============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,  -- 'Procedure', 'Execution', 'Equipment', 'User'
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(200) NOT NULL,  -- 'ProcedureCreated', 'StepAdded', etc.
    event_version   INT NOT NULL DEFAULT 1,  -- schema version for upcasting
    sequence_number BIGINT NOT NULL,  -- per-aggregate ordering
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation_id, causation_id, user_id, device_id
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, sequence_number)
);

CREATE INDEX idx_events_aggregate ON event_store(aggregate_type, aggregate_id);
CREATE INDEX idx_events_type ON event_store(event_type);
CREATE INDEX idx_events_occurred ON event_store(occurred_at);

-- Global ordering for projections
CREATE SEQUENCE event_global_position;
ALTER TABLE event_store ADD COLUMN global_position BIGINT NOT NULL DEFAULT nextval('event_global_position');
CREATE INDEX idx_events_global_position ON event_store(global_position);

-- =============================================================
-- SNAPSHOTS (optional, for aggregates with long event histories)
-- =============================================================

CREATE TABLE aggregate_snapshots (
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_version BIGINT NOT NULL,  -- sequence_number at snapshot time
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);

-- =============================================================
-- PROJECTION TRACKING
-- =============================================================

CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_position   BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Domain Events Catalog

```
-- ORGANIZATION AGGREGATE
OrganizationCreated         { name, slug, subscription_tier }
OrganizationUpdated         { changed_fields }
SiteAdded                   { site_id, name, location, timezone }
SiteDeactivated             { site_id, reason }

-- USER AGGREGATE
UserRegistered              { email, display_name, auth_provider, organization_id }
UserRoleAssigned            { role_name, site_id }
UserRoleRevoked             { role_name, site_id }
UserSkillAssessed           { skill_level, equipment_category, assessed_by }
UserDeactivated             { reason }

-- EQUIPMENT AGGREGATE
EquipmentRegistered         { site_id, name, serial_number, manufacturer, model, category }
SpatialAnchorAttached       { anchor_id, anchor_type, transform_matrix, marker_value, point_cloud_url }
SpatialAnchorUpdated        { anchor_id, changed_fields }
SpatialAnchorRemoved        { anchor_id }
EquipmentDecommissioned     { reason }

-- PROCEDURE AGGREGATE (most complex)
ProcedureCreated            { code, title, description, procedure_type, complexity, default_language }
ProcedureMetadataUpdated    { changed_fields }
ProcedureVersionDrafted     { version_number, authored_by, change_summary }
StepAdded                   { step_id, step_number, title, instruction_text, step_type, is_critical }
StepUpdated                 { step_id, changed_fields }
StepReordered               { step_id, old_position, new_position }
StepRemoved                 { step_id }
StepMediaAttached           { step_id, media_id, media_type, file_url, display_order }
StepMediaRemoved            { step_id, media_id }
Step3DOverlayConfigured     { step_id, overlay_id, anchor_id, model_url, position, rotation, scale }
Step3DOverlayUpdated        { step_id, overlay_id, changed_fields }
StepAIInspectionConfigured  { step_id, model_name, pass_threshold, capture_type }
StepTranslationAdded        { step_id, language_code, title, instruction_text }
ReviewRequested             { version_id, requested_by, reviewer_id }
ReviewCompleted             { version_id, reviewer_id, decision, comments }
ProcedureApproved           { version_id, approved_by }
ProcedurePublished          { version_id, published_by }
ProcedureVersionWithdrawn   { version_id, reason }
ProcedureArchived           { reason }

-- EXECUTION AGGREGATE
ExecutionStarted            { procedure_version_id, equipment_id, operator_id, device_type, is_offline }
StepExecutionStarted        { step_id, step_number }
StepExecutionCompleted      { step_id, status, duration_seconds, operator_notes }
StepAICheckPerformed        { step_id, result, confidence, capture_url }
StepSkipped                 { step_id, reason }
StepRetried                 { step_id, attempt_number }
ExecutionPaused             { reason }
ExecutionResumed            {}
ExecutionCompleted          { total_duration_seconds }
ExecutionAborted            { reason }
ExecutionSynced             { synced_at, conflict_resolution }

-- REMOTE EXPERT AGGREGATE
ExpertSessionRequested      { execution_id, operator_id }
ExpertSessionStarted        { expert_id }
ExpertAnnotationAdded       { annotation_type, position, content, timestamp_ms }
ExpertSessionEnded          { duration_seconds, recording_url }

-- INTEGRATION EVENTS
WorkOrderLinked             { execution_id, external_system, external_id }
WebhookDelivered            { webhook_id, event_type, response_code }
WebhookDeliveryFailed       { webhook_id, event_type, error }
CADImportInitiated          { source_system, source_filename }
CADImportCompleted          { output_gltf_url, polygon_count }
CADImportFailed             { error_message }

-- ANALYTICS EVENTS
AnomalyDetected             { alert_type, severity, procedure_id, step_id, description, detection_data }
AnomalyAcknowledged         { alert_id, acknowledged_by }
```

## Command Handlers (Pseudocode)

```
-- Example: Publishing a procedure
COMMAND PublishProcedure {
    procedure_id: UUID,
    version_id: UUID,
    published_by: UUID
}

HANDLER:
    1. Load Procedure aggregate from event stream
    2. VALIDATE: version status == 'approved'
    3. VALIDATE: published_by has 'procedure.publish' permission
    4. VALIDATE: no other version currently published for this procedure
    5. EMIT ProcedurePublished { version_id, published_by }
    6. (Optional) EMIT ProcedureVersionWithdrawn for previously published version

-- Example: Completing a step execution
COMMAND CompleteStepExecution {
    execution_id: UUID,
    step_id: UUID,
    status: 'passed' | 'failed',
    duration_seconds: INT,
    operator_notes: TEXT
}

HANDLER:
    1. Load Execution aggregate from event stream
    2. VALIDATE: execution status == 'in_progress'
    3. VALIDATE: step is the current active step (or allow out-of-order based on procedure config)
    4. If step has AI inspection config:
        - VALIDATE: AI check has been performed
    5. EMIT StepExecutionCompleted { step_id, status, duration_seconds, operator_notes }
    6. If all steps completed: EMIT ExecutionCompleted
```

## Read Projections

```sql
-- =============================================================
-- PROJECTION: Procedures (denormalized for fast reads)
-- =============================================================

CREATE TABLE proj_procedures (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    code                VARCHAR(50) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    procedure_type      VARCHAR(50) NOT NULL,
    complexity          VARCHAR(20) NOT NULL,
    default_language    VARCHAR(10) NOT NULL,
    current_version_id  UUID,
    current_version_num INT,
    current_status      VARCHAR(30),
    step_count          INT NOT NULL DEFAULT 0,
    is_archived         BOOLEAN NOT NULL DEFAULT false,
    created_by          UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_procedures_org ON proj_procedures(organization_id);
CREATE INDEX idx_proj_procedures_type ON proj_procedures(procedure_type);
CREATE INDEX idx_proj_procedures_status ON proj_procedures(current_status);

-- =============================================================
-- PROJECTION: Full procedure with steps (for AR playback)
-- =============================================================

CREATE TABLE proj_procedure_steps (
    id                  UUID PRIMARY KEY,
    procedure_id        UUID NOT NULL,
    version_id          UUID NOT NULL,
    step_number         INT NOT NULL,
    title               VARCHAR(500) NOT NULL,
    instruction_text    TEXT NOT NULL,
    step_type           VARCHAR(50) NOT NULL,
    is_critical         BOOLEAN NOT NULL,
    estimated_duration  INT,
    can_skip            BOOLEAN NOT NULL DEFAULT false,
    media               JSONB NOT NULL DEFAULT '[]',  -- denormalized media list
    overlay_3d          JSONB,  -- denormalized 3D overlay config
    ai_inspection       JSONB,  -- denormalized AI check config
    translations        JSONB NOT NULL DEFAULT '{}',  -- { "es": { title, text }, "de": { title, text } }
    UNIQUE (version_id, step_number)
);

CREATE INDEX idx_proj_steps_version ON proj_procedure_steps(version_id);

-- =============================================================
-- PROJECTION: Execution dashboard
-- =============================================================

CREATE TABLE proj_execution_summary (
    id                  UUID PRIMARY KEY,
    procedure_code      VARCHAR(50) NOT NULL,
    procedure_title     VARCHAR(500) NOT NULL,
    version_number      INT NOT NULL,
    equipment_name      VARCHAR(255) NOT NULL,
    site_name           VARCHAR(255) NOT NULL,
    operator_name       VARCHAR(255) NOT NULL,
    operator_id         UUID NOT NULL,
    status              VARCHAR(30) NOT NULL,
    started_at          TIMESTAMPTZ NOT NULL,
    completed_at        TIMESTAMPTZ,
    total_duration_seconds INT,
    steps_total         INT NOT NULL DEFAULT 0,
    steps_completed     INT NOT NULL DEFAULT 0,
    steps_failed        INT NOT NULL DEFAULT 0,
    steps_skipped       INT NOT NULL DEFAULT 0,
    device_type         VARCHAR(50),
    is_offline          BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_proj_exec_operator ON proj_execution_summary(operator_id);
CREATE INDEX idx_proj_exec_status ON proj_execution_summary(status);
CREATE INDEX idx_proj_exec_started ON proj_execution_summary(started_at);

-- =============================================================
-- PROJECTION: Equipment with spatial anchors
-- =============================================================

CREATE TABLE proj_equipment (
    id              UUID PRIMARY KEY,
    site_id         UUID NOT NULL,
    site_name       VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    serial_number   VARCHAR(255),
    category_name   VARCHAR(255),
    manufacturer    VARCHAR(255),
    model           VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    anchor_count    INT NOT NULL DEFAULT 0,
    procedure_count INT NOT NULL DEFAULT 0,
    spatial_anchors JSONB NOT NULL DEFAULT '[]'  -- denormalized for AR device lookup
);

CREATE INDEX idx_proj_equip_site ON proj_equipment(site_id);

-- =============================================================
-- PROJECTION: Operator skill profile
-- =============================================================

CREATE TABLE proj_operator_profiles (
    user_id             UUID PRIMARY KEY,
    display_name        VARCHAR(255) NOT NULL,
    organization_id     UUID NOT NULL,
    skill_levels        JSONB NOT NULL DEFAULT '{}',  -- { "category_id": "expert", ... }
    total_executions    INT NOT NULL DEFAULT 0,
    completed_executions INT NOT NULL DEFAULT 0,
    avg_completion_time_seconds DOUBLE PRECISION,
    failure_rate        DOUBLE PRECISION,
    last_execution_at   TIMESTAMPTZ
);

-- =============================================================
-- PROJECTION: Anomaly detection feed
-- =============================================================

CREATE TABLE proj_anomaly_feed (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    procedure_code  VARCHAR(50),
    procedure_title VARCHAR(500),
    step_title      VARCHAR(500),
    description     TEXT NOT NULL,
    is_acknowledged BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by VARCHAR(255),
    detected_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_anomaly_org ON proj_anomaly_feed(organization_id);
CREATE INDEX idx_proj_anomaly_severity ON proj_anomaly_feed(severity);
```

## Offline Sync Strategy with Events

The event-sourced architecture provides a natural solution for offline-first operation:

1. **Device event log:** AR devices maintain a local event log (IndexedDB or SQLite) recording all step executions, AI checks, and annotations while offline.
2. **Sync on reconnect:** When connectivity returns, the device pushes its local event stream to the server.
3. **Conflict detection:** The server checks for conflicting events (e.g., same execution modified by an expert remotely while the operator was offline). Conflicts are resolved by timestamp ordering with operator events taking priority for step completions.
4. **Idempotency:** Each event carries a deterministic event_id generated on the device, ensuring replay safety.

## Scalability Considerations

- **Event store partitioning:** Partition the `event_store` table by `aggregate_type` and time range. Execution events will vastly outnumber procedure authoring events.
- **Snapshotting:** For Procedure aggregates with hundreds of steps and revisions, create snapshots every N events (e.g., every 50) to avoid replaying long event histories.
- **Projection workers:** Run projection updaters as independent workers consuming from the event stream. Scale horizontally by partitioning by aggregate_id.
- **Event archival:** Move events older than a retention period to cold storage (S3/GCS) while keeping projections current. Rebuild from archive if needed.
- **Message broker option:** For high-throughput deployments, front the event store with Apache Kafka or NATS JetStream. Events are published to topics, consumed by projection workers, and persisted to PostgreSQL in batches.

## Migration Path

- **From relational:** Existing relational data can be converted to events through a "snapshotting migration" that creates synthetic creation events from current state, then switches to event-sourced writes going forward.
- **To hybrid:** If CQRS complexity proves excessive for some aggregates (e.g., Equipment is mostly CRUD), those can be reverted to simple relational tables while keeping event sourcing for Procedure and Execution aggregates where the audit benefits are highest.
- **Event versioning:** Use an upcaster registry that transforms old event schemas to current versions on read. Never modify stored events; always version forward.
