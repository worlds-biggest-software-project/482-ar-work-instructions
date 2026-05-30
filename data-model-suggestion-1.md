# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A fully normalized (3NF+) relational schema in PostgreSQL, using strict foreign keys, check constraints, and proper indexing. Every entity is its own table with atomic columns; no embedded JSON or denormalized blobs. This is the most traditional and well-understood approach for enterprise applications with strong audit, compliance, and reporting requirements.

## Why This Suits the Domain

AR Work Instructions is fundamentally a **structured workflow system** with clear entity hierarchies: organizations contain sites, sites contain equipment, equipment has procedures, procedures have steps, steps have media and spatial anchors. The compliance and audit requirements (per-step timestamps, operator ID, pass/fail records) demand referential integrity that normalized schemas enforce at the database level. Versioning with approval workflows maps naturally to relational version/revision tables. Role-based access control is a solved problem in relational databases with well-known patterns.

## Trade-offs

**Strengths:**
- Rock-solid referential integrity enforced at the DB level
- Excellent for complex reporting and compliance queries
- Mature tooling for migrations, backups, replication
- Simple to reason about for developers familiar with SQL
- Strong audit trail with foreign-keyed history tables

**Weaknesses:**
- Spatial anchor metadata (point clouds, transformation matrices) may feel awkward in flat columns
- Schema migrations required for every structural change to procedure step types
- Deep JOIN chains for full procedure retrieval (procedure -> version -> step -> media -> anchor)
- 3D model metadata and AR-specific configuration may require many columns or auxiliary tables
- Offline-first sync patterns are harder without document-oriented structures

## Schema Definition

```sql
-- =============================================================
-- ORGANIZATION & TENANCY
-- =============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        TEXT,  -- serialized org-level config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    location_label  VARCHAR(255),
    latitude        DOUBLE PRECISION,
    longitude       DOUBLE PRECISION,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sites_org ON sites(organization_id);

-- =============================================================
-- USERS & ACCESS CONTROL
-- =============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    auth_provider_id VARCHAR(255),
    locale          VARCHAR(10) NOT NULL DEFAULT 'en',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,  -- 'author', 'reviewer', 'operator', 'admin'
    description     TEXT,
    UNIQUE (organization_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    site_id         UUID REFERENCES sites(id) ON DELETE CASCADE,  -- NULL = org-wide
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id, site_id)
);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,  -- 'procedure.create', 'procedure.approve', etc.
    description     TEXT
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- =============================================================
-- SKILL TRACKING
-- =============================================================

CREATE TABLE skill_levels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,  -- 'novice', 'competent', 'expert'
    rank            INT NOT NULL,
    UNIQUE (organization_id, name)
);

CREATE TABLE user_skills (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    skill_level_id  UUID NOT NULL REFERENCES skill_levels(id) ON DELETE CASCADE,
    equipment_category_id UUID,  -- optional: skill per equipment category
    assessed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    assessed_by     UUID REFERENCES users(id),
    PRIMARY KEY (user_id, skill_level_id)
);

-- =============================================================
-- EQUIPMENT & SPATIAL ANCHORS
-- =============================================================

CREATE TABLE equipment_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    parent_id       UUID REFERENCES equipment_categories(id),
    UNIQUE (organization_id, name, parent_id)
);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
    category_id     UUID REFERENCES equipment_categories(id),
    name            VARCHAR(255) NOT NULL,
    serial_number   VARCHAR(255),
    manufacturer    VARCHAR(255),
    model           VARCHAR(255),
    commissioned_at DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_site ON equipment(site_id);
CREATE INDEX idx_equipment_category ON equipment(category_id);

CREATE TABLE spatial_anchors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id) ON DELETE CASCADE,
    anchor_type     VARCHAR(50) NOT NULL CHECK (anchor_type IN ('qr_code', 'marker', 'object_detection', 'point_cloud', 'manual')),
    label           VARCHAR(255),
    -- Transformation matrix (4x4) stored as individual floats
    transform_m00   DOUBLE PRECISION, transform_m01 DOUBLE PRECISION, transform_m02 DOUBLE PRECISION, transform_m03 DOUBLE PRECISION,
    transform_m10   DOUBLE PRECISION, transform_m11 DOUBLE PRECISION, transform_m12 DOUBLE PRECISION, transform_m13 DOUBLE PRECISION,
    transform_m20   DOUBLE PRECISION, transform_m21 DOUBLE PRECISION, transform_m22 DOUBLE PRECISION, transform_m23 DOUBLE PRECISION,
    transform_m30   DOUBLE PRECISION, transform_m31 DOUBLE PRECISION, transform_m32 DOUBLE PRECISION, transform_m33 DOUBLE PRECISION,
    -- QR / marker data
    marker_value    VARCHAR(500),
    -- Object detection model reference
    detection_model_id UUID,
    -- Point cloud reference (stored in object storage)
    point_cloud_url VARCHAR(1024),
    confidence_threshold DOUBLE PRECISION DEFAULT 0.85,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spatial_anchors_equipment ON spatial_anchors(equipment_id);

-- =============================================================
-- PROCEDURES & VERSIONING
-- =============================================================

CREATE TABLE procedures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    equipment_category_id UUID REFERENCES equipment_categories(id),
    code            VARCHAR(50) NOT NULL,  -- human-readable identifier e.g. 'MNT-042'
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    procedure_type  VARCHAR(50) NOT NULL CHECK (procedure_type IN ('maintenance', 'assembly', 'inspection', 'safety', 'calibration', 'custom')),
    complexity      VARCHAR(20) NOT NULL DEFAULT 'standard' CHECK (complexity IN ('basic', 'standard', 'advanced', 'expert')),
    min_skill_level_id UUID REFERENCES skill_levels(id),
    source_document_url VARCHAR(1024),  -- original PDF/Word for AI conversion
    default_language VARCHAR(10) NOT NULL DEFAULT 'en',
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

CREATE INDEX idx_procedures_org ON procedures(organization_id);
CREATE INDEX idx_procedures_type ON procedures(procedure_type);

CREATE TABLE procedure_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    procedure_id    UUID NOT NULL REFERENCES procedures(id) ON DELETE CASCADE,
    version_number  INT NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'in_review', 'approved', 'published', 'superseded', 'withdrawn')),
    change_summary  TEXT,
    authored_by     UUID NOT NULL REFERENCES users(id),
    reviewed_by     UUID REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (procedure_id, version_number)
);

CREATE INDEX idx_proc_versions_procedure ON procedure_versions(procedure_id);
CREATE INDEX idx_proc_versions_status ON procedure_versions(status);

-- =============================================================
-- STEPS, MEDIA & 3D OVERLAYS
-- =============================================================

CREATE TABLE procedure_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    procedure_version_id UUID NOT NULL REFERENCES procedure_versions(id) ON DELETE CASCADE,
    step_number     INT NOT NULL,
    title           VARCHAR(500) NOT NULL,
    instruction_text TEXT NOT NULL,
    step_type       VARCHAR(50) NOT NULL DEFAULT 'action' CHECK (step_type IN ('action', 'inspection', 'decision', 'warning', 'info', 'ai_check')),
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    estimated_duration_seconds INT,
    can_skip        BOOLEAN NOT NULL DEFAULT false,
    skip_condition  TEXT,  -- condition expression for adaptive procedures
    tool_requirements TEXT,
    safety_notes    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (procedure_version_id, step_number)
);

CREATE INDEX idx_steps_version ON procedure_steps(procedure_version_id);

CREATE TABLE step_translations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    step_id         UUID NOT NULL REFERENCES procedure_steps(id) ON DELETE CASCADE,
    language_code   VARCHAR(10) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    instruction_text TEXT NOT NULL,
    safety_notes    TEXT,
    translated_by   VARCHAR(50) NOT NULL DEFAULT 'manual',  -- 'manual', 'ai', 'professional'
    verified        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (step_id, language_code)
);

CREATE TABLE step_media (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    step_id         UUID NOT NULL REFERENCES procedure_steps(id) ON DELETE CASCADE,
    media_type      VARCHAR(30) NOT NULL CHECK (media_type IN ('image', 'video', 'audio', '3d_model', 'animation', 'pdf')),
    file_url        VARCHAR(1024) NOT NULL,
    thumbnail_url   VARCHAR(1024),
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    display_order   INT NOT NULL DEFAULT 0,
    caption         VARCHAR(500),
    duration_seconds DOUBLE PRECISION,  -- for video/audio
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_step_media_step ON step_media(step_id);

CREATE TABLE step_3d_overlays (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    step_id         UUID NOT NULL REFERENCES procedure_steps(id) ON DELETE CASCADE,
    spatial_anchor_id UUID NOT NULL REFERENCES spatial_anchors(id),
    model_url       VARCHAR(1024) NOT NULL,  -- glTF 2.0 URL
    model_format    VARCHAR(20) NOT NULL DEFAULT 'gltf' CHECK (model_format IN ('gltf', 'glb', 'usdz')),
    compression     VARCHAR(20) DEFAULT 'draco',
    -- Position offset from anchor (meters)
    offset_x        DOUBLE PRECISION NOT NULL DEFAULT 0,
    offset_y        DOUBLE PRECISION NOT NULL DEFAULT 0,
    offset_z        DOUBLE PRECISION NOT NULL DEFAULT 0,
    -- Rotation (quaternion)
    rotation_x      DOUBLE PRECISION NOT NULL DEFAULT 0,
    rotation_y      DOUBLE PRECISION NOT NULL DEFAULT 0,
    rotation_z      DOUBLE PRECISION NOT NULL DEFAULT 0,
    rotation_w      DOUBLE PRECISION NOT NULL DEFAULT 1,
    -- Scale
    scale_x         DOUBLE PRECISION NOT NULL DEFAULT 1,
    scale_y         DOUBLE PRECISION NOT NULL DEFAULT 1,
    scale_z         DOUBLE PRECISION NOT NULL DEFAULT 1,
    opacity         DOUBLE PRECISION NOT NULL DEFAULT 1.0,
    animation_name  VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_3d_overlays_step ON step_3d_overlays(step_id);
CREATE INDEX idx_3d_overlays_anchor ON step_3d_overlays(spatial_anchor_id);

-- =============================================================
-- AI INSPECTION CHECKS
-- =============================================================

CREATE TABLE ai_inspection_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    step_id         UUID NOT NULL REFERENCES procedure_steps(id) ON DELETE CASCADE,
    model_name      VARCHAR(255) NOT NULL,  -- 'torque_check_v2', 'alignment_verify', etc.
    model_version   VARCHAR(50),
    model_artifact_url VARCHAR(1024),
    pass_threshold  DOUBLE PRECISION NOT NULL DEFAULT 0.90,
    retry_allowed   BOOLEAN NOT NULL DEFAULT true,
    max_retries     INT NOT NULL DEFAULT 3,
    capture_type    VARCHAR(30) NOT NULL DEFAULT 'photo' CHECK (capture_type IN ('photo', 'video_clip', 'depth_scan')),
    reference_image_url VARCHAR(1024),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- EXECUTION & AUDIT
-- =============================================================

CREATE TABLE procedure_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    procedure_version_id UUID NOT NULL REFERENCES procedure_versions(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    operator_id     UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'in_progress' CHECK (status IN ('in_progress', 'completed', 'paused', 'aborted', 'failed')),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    total_duration_seconds INT,
    device_type     VARCHAR(50),  -- 'mobile_ios', 'mobile_android', 'hololens2', 'meta_quest', 'web'
    device_id       VARCHAR(255),
    is_offline      BOOLEAN NOT NULL DEFAULT false,
    synced_at       TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_executions_operator ON procedure_executions(operator_id);
CREATE INDEX idx_executions_equipment ON procedure_executions(equipment_id);
CREATE INDEX idx_executions_version ON procedure_executions(procedure_version_id);
CREATE INDEX idx_executions_status ON procedure_executions(status);
CREATE INDEX idx_executions_started ON procedure_executions(started_at);

CREATE TABLE step_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID NOT NULL REFERENCES procedure_executions(id) ON DELETE CASCADE,
    step_id         UUID NOT NULL REFERENCES procedure_steps(id),
    step_number     INT NOT NULL,
    status          VARCHAR(20) NOT NULL CHECK (status IN ('pending', 'in_progress', 'passed', 'failed', 'skipped', 'retried')),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    duration_seconds INT,
    was_skipped     BOOLEAN NOT NULL DEFAULT false,
    skip_reason     TEXT,
    operator_notes  TEXT,
    ai_check_result VARCHAR(20) CHECK (ai_check_result IN ('pass', 'fail', 'inconclusive', 'not_applicable')),
    ai_confidence   DOUBLE PRECISION,
    ai_capture_url  VARCHAR(1024),  -- captured image/video for audit
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_step_exec_execution ON step_executions(execution_id);
CREATE INDEX idx_step_exec_status ON step_executions(status);

-- =============================================================
-- ENTERPRISE INTEGRATION
-- =============================================================

CREATE TABLE webhook_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    target_url      VARCHAR(1024) NOT NULL,
    event_types     TEXT NOT NULL,  -- comma-separated: 'execution.completed,procedure.published'
    secret_hash     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_order_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID NOT NULL REFERENCES procedure_executions(id),
    external_system VARCHAR(50) NOT NULL,  -- 'sap', 'oracle', 'custom'
    external_id     VARCHAR(255) NOT NULL,
    external_url    VARCHAR(1024),
    linked_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- REMOTE EXPERT SESSIONS
-- =============================================================

CREATE TABLE remote_expert_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID REFERENCES procedure_executions(id),
    operator_id     UUID NOT NULL REFERENCES users(id),
    expert_id       UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'requested' CHECK (status IN ('requested', 'active', 'ended', 'missed')),
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    recording_url   VARCHAR(1024),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE expert_annotations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES remote_expert_sessions(id) ON DELETE CASCADE,
    annotation_type VARCHAR(30) NOT NULL CHECK (annotation_type IN ('arrow', 'circle', 'text', 'freehand', '3d_pointer')),
    spatial_anchor_id UUID REFERENCES spatial_anchors(id),
    position_x      DOUBLE PRECISION,
    position_y      DOUBLE PRECISION,
    position_z      DOUBLE PRECISION,
    content         TEXT,
    timestamp_ms    BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- ANOMALY DETECTION & ANALYTICS
-- =============================================================

CREATE TABLE anomaly_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    alert_type      VARCHAR(50) NOT NULL,  -- 'step_failure_spike', 'duration_drift', 'skip_pattern'
    severity        VARCHAR(20) NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    procedure_id    UUID REFERENCES procedures(id),
    step_id         UUID REFERENCES procedure_steps(id),
    description     TEXT NOT NULL,
    detection_data  TEXT,  -- serialized statistics
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_anomaly_alerts_org ON anomaly_alerts(organization_id);
CREATE INDEX idx_anomaly_alerts_severity ON anomaly_alerts(severity);

-- =============================================================
-- CAD IMPORT TRACKING
-- =============================================================

CREATE TABLE cad_imports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    source_system   VARCHAR(50) NOT NULL CHECK (source_system IN ('solidworks', 'siemens_nx', 'ptc_creo', 'step_file', 'manual_upload')),
    source_filename VARCHAR(500) NOT NULL,
    source_format   VARCHAR(30) NOT NULL,  -- 'sldprt', 'prt', 'step', 'iges'
    output_gltf_url VARCHAR(1024),
    conversion_status VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (conversion_status IN ('pending', 'processing', 'completed', 'failed')),
    polygon_count   BIGINT,
    file_size_bytes BIGINT,
    imported_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Scalability Considerations

- **Partitioning:** `step_executions` and `procedure_executions` should be range-partitioned by `started_at` for time-series queries and archival. At scale (millions of executions/month), move completed execution data to partitioned archive tables.
- **Read replicas:** Reporting and analytics queries (anomaly detection, compliance exports) should target read replicas to avoid impacting authoring and playback workloads.
- **Connection pooling:** Use PgBouncer or similar for the many short-lived connections from AR devices.
- **Offline sync:** The `is_offline` and `synced_at` columns on `procedure_executions` support conflict-detection on sync, but the application layer must implement last-write-wins or manual conflict resolution.

## Migration Path

This schema is straightforward to evolve using standard migration tools (Flyway, Alembic, Prisma Migrate). Adding new step types requires an ALTER to the CHECK constraint. If the rigid column structure for spatial anchors or 3D overlays becomes limiting, individual tables can be migrated to a hybrid JSONB approach (see Suggestion 3) without rewriting the rest of the schema.
