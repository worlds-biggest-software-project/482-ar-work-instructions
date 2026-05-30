# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A hybrid schema that uses PostgreSQL relational tables with strict foreign keys for core entities and relationships, while leveraging JSONB columns for inherently flexible, nested, or schema-variable data. This targets the sweet spot between the referential integrity of a fully normalized model and the flexibility of a document store. Spatial transforms, 3D overlay configurations, media metadata, AI model parameters, and translation bundles are stored as JSONB, while organizational hierarchy, users, procedures, and execution records remain relational.

## Why This Suits the Domain

AR Work Instructions has a dual nature: the **organizational and workflow structure** (orgs, sites, users, procedures, versions, approvals, executions) is highly relational with clear cardinality and referential constraints. But the **AR-specific content** — spatial anchor transforms, 3D overlay position/rotation/scale, AI inspection parameters, device capabilities, and multi-language translations — is deeply nested, varies by anchor type, and evolves rapidly as device capabilities change.

A pure relational model (Suggestion 1) forces 16 individual float columns for a 4x4 transformation matrix and separate tables for every media variant. A pure document model loses referential integrity on the workflow backbone. The hybrid approach keeps the best of both: foreign keys enforce that every step belongs to a valid procedure version, while JSONB absorbs the structural variety of AR configuration without schema migrations for every new device or overlay parameter.

Key domain fits:
1. **Spatial anchors** have radically different data shapes depending on type (QR code vs. point cloud vs. object detection). JSONB handles this polymorphism naturally.
2. **3D overlays** combine position, rotation, scale, animation state, and material overrides — a nested structure that reads and writes as a single unit.
3. **Multi-language translations** are a sparse map (not every step is translated to every language) best represented as a JSONB object keyed by language code.
4. **AI inspection configs** evolve rapidly as new models are deployed — JSONB avoids ALTER TABLE for every new parameter.
5. **Webhook and integration payloads** vary by target system and should not constrain the relational schema.

## Trade-offs

**Strengths:**
- Core referential integrity preserved: procedures, versions, steps, executions, users all have foreign keys
- Flexible AR content without schema migrations for new anchor types, overlay params, or device fields
- GIN indexes on JSONB columns enable fast containment and existence queries
- Single database technology (PostgreSQL) — no additional infrastructure
- JSONB columns can be individually migrated to relational columns if a structure stabilizes
- Native PostgreSQL JSON functions for reporting across flexible fields
- Offline sync payloads map naturally to JSONB execution records

**Weaknesses:**
- JSONB contents are not enforced by database constraints (must validate in application layer or with CHECK constraints)
- Deep JSONB nesting can make queries verbose (jsonb_path_query, ->> chains)
- GIN indexes are larger than B-tree indexes and slower to update under heavy write loads
- Partial indexes on JSONB fields require careful design
- Developers must know which data lives in columns vs. JSONB — the boundary is a design decision, not a given
- JSONB does not support foreign keys to other tables

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
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings: { "default_language": "en", "max_users": 50,
    --   "features": { "ai_inspection": true, "remote_expert": true },
    --   "branding": { "logo_url": "...", "primary_color": "#003366" } }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    location        JSONB NOT NULL DEFAULT '{}',
    -- location: { "label": "Building A", "latitude": 37.7749,
    --   "longitude": -122.4194, "address": "...", "floor_plan_url": "..." }
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
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
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile: { "phone": "...", "department": "Maintenance",
    --   "certifications": ["ISO-9001-auditor", "forklift"],
    --   "preferred_device": "hololens2" }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users(organization_id);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions: ["procedure.create", "procedure.approve", "execution.view",
    --   "equipment.manage", "user.manage", "analytics.view"]
    description     TEXT,
    UNIQUE (organization_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    site_id         UUID REFERENCES sites(id) ON DELETE CASCADE,  -- NULL = org-wide
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id, COALESCE(site_id, '00000000-0000-0000-0000-000000000000'))
);

-- =============================================================
-- SKILL TRACKING
-- =============================================================

CREATE TABLE user_skills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    equipment_category_id UUID,  -- NULL = general skill
    skill_level     VARCHAR(50) NOT NULL CHECK (skill_level IN ('novice', 'beginner', 'competent', 'proficient', 'expert')),
    assessment      JSONB NOT NULL DEFAULT '{}',
    -- assessment: { "assessed_by": "uuid", "method": "practical_exam",
    --   "score": 92, "notes": "...", "expiry_date": "2027-01-15" }
    assessed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, equipment_category_id)
);

CREATE INDEX idx_user_skills_user ON user_skills(user_id);

-- =============================================================
-- EQUIPMENT & SPATIAL ANCHORS
-- =============================================================

CREATE TABLE equipment_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES equipment_categories(id),
    description     TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata: { "industry": "automotive", "maintenance_interval_days": 90,
    --   "required_certifications": ["hydraulics-level-2"] }
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
    is_active       BOOLEAN NOT NULL DEFAULT true,
    specs           JSONB NOT NULL DEFAULT '{}',
    -- specs: { "commissioned_at": "2024-03-15", "warranty_expiry": "2027-03-15",
    --   "firmware_version": "3.2.1", "weight_kg": 450,
    --   "dimensions": { "length_mm": 2400, "width_mm": 800, "height_mm": 1200 },
    --   "hazards": ["high_voltage", "pinch_points"] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_site ON equipment(site_id);
CREATE INDEX idx_equipment_category ON equipment(category_id);
CREATE INDEX idx_equipment_specs ON equipment USING GIN (specs);

CREATE TABLE spatial_anchors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id) ON DELETE CASCADE,
    anchor_type     VARCHAR(50) NOT NULL CHECK (anchor_type IN (
        'qr_code', 'marker', 'object_detection', 'point_cloud', 'image_target', 'manual'
    )),
    label           VARCHAR(255),
    transform       JSONB NOT NULL,
    -- transform: 4x4 matrix as nested array:
    -- [[1,0,0,tx],[0,1,0,ty],[0,0,1,tz],[0,0,0,1]]
    anchor_config   JSONB NOT NULL DEFAULT '{}',
    -- anchor_config varies by anchor_type:
    --   qr_code:          { "value": "EQUIP-042", "size_mm": 150, "error_correction": "H" }
    --   marker:           { "dictionary": "aruco_4x4", "marker_id": 42, "size_mm": 100 }
    --   object_detection: { "model_id": "uuid", "model_url": "...", "confidence_threshold": 0.85,
    --                        "object_class": "hydraulic_press_v2" }
    --   point_cloud:      { "scan_url": "s3://...", "format": "ply", "point_count": 250000,
    --                        "bounding_box": { "min": [0,0,0], "max": [2.4,0.8,1.2] } }
    --   image_target:     { "reference_image_url": "...", "physical_width_mm": 300 }
    --   manual:           { "instructions": "Place device at marked position on floor" }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spatial_anchors_equipment ON spatial_anchors(equipment_id);
CREATE INDEX idx_spatial_anchors_type ON spatial_anchors(anchor_type);
CREATE INDEX idx_spatial_anchors_config ON spatial_anchors USING GIN (anchor_config);

-- =============================================================
-- PROCEDURES & VERSIONING
-- =============================================================

CREATE TABLE procedures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    equipment_category_id UUID REFERENCES equipment_categories(id),
    code            VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    procedure_type  VARCHAR(50) NOT NULL CHECK (procedure_type IN (
        'maintenance', 'assembly', 'inspection', 'safety', 'calibration', 'troubleshooting', 'custom'
    )),
    complexity      VARCHAR(20) NOT NULL DEFAULT 'standard' CHECK (complexity IN (
        'basic', 'standard', 'advanced', 'expert'
    )),
    min_skill_level VARCHAR(50),
    default_language VARCHAR(10) NOT NULL DEFAULT 'en',
    source_document_url VARCHAR(1024),
    tags            JSONB NOT NULL DEFAULT '[]',
    -- tags: ["preventive", "quarterly", "safety-critical", "line-A"]
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

CREATE INDEX idx_procedures_org ON procedures(organization_id);
CREATE INDEX idx_procedures_type ON procedures(procedure_type);
CREATE INDEX idx_procedures_tags ON procedures USING GIN (tags);

CREATE TABLE procedure_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    procedure_id    UUID NOT NULL REFERENCES procedures(id) ON DELETE CASCADE,
    version_number  INT NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'in_review', 'approved', 'published', 'superseded', 'withdrawn'
    )),
    change_summary  TEXT,
    authored_by     UUID NOT NULL REFERENCES users(id),
    reviewed_by     UUID REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    review_notes    JSONB NOT NULL DEFAULT '[]',
    -- review_notes: [{ "reviewer_id": "uuid", "comment": "...",
    --   "timestamp": "2026-05-20T14:30:00Z", "status": "approved" }]
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (procedure_id, version_number)
);

CREATE INDEX idx_proc_versions_procedure ON procedure_versions(procedure_id);
CREATE INDEX idx_proc_versions_status ON procedure_versions(status);

-- =============================================================
-- STEPS (core relational + JSONB for AR content)
-- =============================================================

CREATE TABLE procedure_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    procedure_version_id UUID NOT NULL REFERENCES procedure_versions(id) ON DELETE CASCADE,
    step_number     INT NOT NULL,
    title           VARCHAR(500) NOT NULL,
    instruction_text TEXT NOT NULL,
    step_type       VARCHAR(50) NOT NULL DEFAULT 'action' CHECK (step_type IN (
        'action', 'inspection', 'decision', 'warning', 'info', 'ai_check', 'measurement'
    )),
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    estimated_duration_seconds INT,
    can_skip        BOOLEAN NOT NULL DEFAULT false,
    skip_condition  TEXT,
    safety_notes    TEXT,
    tool_requirements TEXT,

    -- JSONB: media attachments (images, video, audio, documents)
    media           JSONB NOT NULL DEFAULT '[]',
    -- media: [{ "id": "uuid", "type": "image", "url": "s3://...",
    --   "thumbnail_url": "...", "mime_type": "image/jpeg",
    --   "file_size_bytes": 245000, "caption": "Align bracket with mark",
    --   "display_order": 1, "duration_seconds": null }]

    -- JSONB: 3D overlay configurations
    overlays_3d     JSONB NOT NULL DEFAULT '[]',
    -- overlays_3d: [{ "id": "uuid", "anchor_id": "uuid",
    --   "model_url": "s3://models/bracket_highlight.glb",
    --   "format": "glb", "compression": "draco",
    --   "position": { "x": 0.15, "y": 0.30, "z": -0.05 },
    --   "rotation": { "x": 0, "y": 0, "z": 0, "w": 1 },
    --   "scale": { "x": 1, "y": 1, "z": 1 },
    --   "opacity": 0.8, "animation": "pulse",
    --   "highlight_color": "#00FF00" }]

    -- JSONB: AI inspection configuration (null if step is not an AI check)
    ai_inspection   JSONB,
    -- ai_inspection: { "model_name": "torque_check_v2", "model_version": "1.3",
    --   "model_artifact_url": "s3://models/torque_v2.onnx",
    --   "pass_threshold": 0.92, "retry_allowed": true, "max_retries": 3,
    --   "capture_type": "photo", "reference_image_url": "s3://ref/torque_ref.jpg",
    --   "regions_of_interest": [{ "x": 100, "y": 200, "w": 300, "h": 300 }] }

    -- JSONB: translations keyed by language code
    translations    JSONB NOT NULL DEFAULT '{}',
    -- translations: {
    --   "es": { "title": "...", "instruction_text": "...", "safety_notes": "...",
    --           "translated_by": "ai", "verified": false },
    --   "de": { "title": "...", "instruction_text": "...", "safety_notes": "...",
    --           "translated_by": "professional", "verified": true }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (procedure_version_id, step_number)
);

CREATE INDEX idx_steps_version ON procedure_steps(procedure_version_id);
CREATE INDEX idx_steps_type ON procedure_steps(step_type);
CREATE INDEX idx_steps_media ON procedure_steps USING GIN (media);
CREATE INDEX idx_steps_overlays ON procedure_steps USING GIN (overlays_3d);
CREATE INDEX idx_steps_translations ON procedure_steps USING GIN (translations);

-- =============================================================
-- EXECUTION & AUDIT
-- =============================================================

CREATE TABLE procedure_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    procedure_version_id UUID NOT NULL REFERENCES procedure_versions(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    operator_id     UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'in_progress' CHECK (status IN (
        'in_progress', 'completed', 'paused', 'aborted', 'failed'
    )),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    total_duration_seconds INT,
    device_info     JSONB NOT NULL DEFAULT '{}',
    -- device_info: { "type": "mobile_ios", "device_id": "...",
    --   "os_version": "iOS 19.1", "app_version": "2.4.0",
    --   "ar_capabilities": ["lidar", "webxr"],
    --   "screen_resolution": "2778x1284" }
    is_offline      BOOLEAN NOT NULL DEFAULT false,
    synced_at       TIMESTAMPTZ,
    notes           TEXT,
    work_order      JSONB,
    -- work_order: { "external_system": "sap", "external_id": "WO-2026-4421",
    --   "external_url": "https://sap.example.com/wo/4421", "linked_at": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_executions_operator ON procedure_executions(operator_id);
CREATE INDEX idx_executions_equipment ON procedure_executions(equipment_id);
CREATE INDEX idx_executions_version ON procedure_executions(procedure_version_id);
CREATE INDEX idx_executions_status ON procedure_executions(status);
CREATE INDEX idx_executions_started ON procedure_executions(started_at);
-- Partial index for active executions (dashboard queries)
CREATE INDEX idx_executions_active ON procedure_executions(operator_id, started_at)
    WHERE status IN ('in_progress', 'paused');

CREATE TABLE step_executions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID NOT NULL REFERENCES procedure_executions(id) ON DELETE CASCADE,
    step_id         UUID NOT NULL REFERENCES procedure_steps(id),
    step_number     INT NOT NULL,
    status          VARCHAR(20) NOT NULL CHECK (status IN (
        'pending', 'in_progress', 'passed', 'failed', 'skipped', 'retried'
    )),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    duration_seconds INT,
    was_skipped     BOOLEAN NOT NULL DEFAULT false,
    skip_reason     TEXT,
    operator_notes  TEXT,
    ai_result       JSONB,
    -- ai_result: { "check_result": "pass", "confidence": 0.97,
    --   "capture_url": "s3://captures/exec-123/step-5.jpg",
    --   "model_name": "torque_check_v2", "model_version": "1.3",
    --   "inference_time_ms": 340,
    --   "detections": [{ "label": "bolt_torqued", "confidence": 0.97,
    --     "bounding_box": { "x": 120, "y": 200, "w": 80, "h": 80 } }] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_step_exec_execution ON step_executions(execution_id);
CREATE INDEX idx_step_exec_status ON step_executions(status);
CREATE INDEX idx_step_exec_ai ON step_executions USING GIN (ai_result)
    WHERE ai_result IS NOT NULL;

-- =============================================================
-- REMOTE EXPERT SESSIONS
-- =============================================================

CREATE TABLE remote_expert_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id    UUID REFERENCES procedure_executions(id),
    operator_id     UUID NOT NULL REFERENCES users(id),
    expert_id       UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'requested' CHECK (status IN (
        'requested', 'active', 'ended', 'missed'
    )),
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    recording_url   VARCHAR(1024),
    annotations     JSONB NOT NULL DEFAULT '[]',
    -- annotations: [{ "id": "uuid", "type": "arrow", "anchor_id": "uuid",
    --   "position": { "x": 0.5, "y": 1.2, "z": -0.1 },
    --   "content": "Loosen this bolt first", "timestamp_ms": 34500,
    --   "color": "#FF0000", "created_at": "..." }]
    session_metadata JSONB NOT NULL DEFAULT '{}',
    -- session_metadata: { "webrtc_quality": "good", "bandwidth_kbps": 2400,
    --   "latency_ms": 45, "resolution": "720p" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_expert_sessions_execution ON remote_expert_sessions(execution_id);
CREATE INDEX idx_expert_sessions_status ON remote_expert_sessions(status);

-- =============================================================
-- ENTERPRISE INTEGRATION
-- =============================================================

CREATE TABLE webhook_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    target_url      VARCHAR(1024) NOT NULL,
    event_types     JSONB NOT NULL DEFAULT '[]',
    -- event_types: ["execution.completed", "procedure.published", "anomaly.critical"]
    headers         JSONB NOT NULL DEFAULT '{}',
    -- headers: { "Authorization": "Bearer ...", "X-Custom": "value" }
    secret_hash     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    retry_policy    JSONB NOT NULL DEFAULT '{"max_retries": 3, "backoff_seconds": [5, 30, 300]}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhooks_org ON webhook_configs(organization_id);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    webhook_id      UUID NOT NULL REFERENCES webhook_configs(id) ON DELETE CASCADE,
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    response_code   INT,
    response_body   TEXT,
    attempt_number  INT NOT NULL DEFAULT 1,
    delivered_at    TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL CHECK (status IN ('pending', 'success', 'failed', 'retrying')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhook_deliveries_webhook ON webhook_deliveries(webhook_id);
CREATE INDEX idx_webhook_deliveries_status ON webhook_deliveries(status);

-- =============================================================
-- ANOMALY DETECTION & ANALYTICS
-- =============================================================

CREATE TABLE anomaly_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    procedure_id    UUID REFERENCES procedures(id),
    step_id         UUID REFERENCES procedure_steps(id),
    description     TEXT NOT NULL,
    detection_data  JSONB NOT NULL DEFAULT '{}',
    -- detection_data: { "metric": "step_failure_rate", "current_value": 0.35,
    --   "baseline_value": 0.05, "window_hours": 24,
    --   "sample_size": 142, "z_score": 4.2,
    --   "affected_executions": ["uuid1", "uuid2"] }
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    resolution_notes TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_anomaly_org ON anomaly_alerts(organization_id);
CREATE INDEX idx_anomaly_severity ON anomaly_alerts(severity);
CREATE INDEX idx_anomaly_type ON anomaly_alerts(alert_type);
CREATE INDEX idx_anomaly_unacked ON anomaly_alerts(organization_id, created_at)
    WHERE acknowledged_at IS NULL;

-- =============================================================
-- CAD IMPORT TRACKING
-- =============================================================

CREATE TABLE cad_imports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    source_system   VARCHAR(50) NOT NULL CHECK (source_system IN (
        'solidworks', 'siemens_nx', 'ptc_creo', 'step_file', 'iges_file', 'manual_upload'
    )),
    source_filename VARCHAR(500) NOT NULL,
    output_gltf_url VARCHAR(1024),
    conversion_status VARCHAR(30) NOT NULL DEFAULT 'pending' CHECK (conversion_status IN (
        'pending', 'processing', 'completed', 'failed'
    )),
    import_details  JSONB NOT NULL DEFAULT '{}',
    -- import_details: { "source_format": "sldprt", "file_size_bytes": 15400000,
    --   "polygon_count": 48000, "mesh_count": 12,
    --   "compression_applied": "draco", "compressed_size_bytes": 2100000,
    --   "conversion_time_seconds": 34, "warnings": ["high polygon count on part 3"],
    --   "lod_levels_generated": 3 }
    imported_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cad_imports_org ON cad_imports(organization_id);
CREATE INDEX idx_cad_imports_status ON cad_imports(conversion_status);

-- =============================================================
-- OFFLINE SYNC QUEUE
-- =============================================================

CREATE TABLE sync_queue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id       VARCHAR(255) NOT NULL,
    operator_id     UUID NOT NULL REFERENCES users(id),
    payload_type    VARCHAR(50) NOT NULL CHECK (payload_type IN (
        'execution', 'step_execution', 'expert_session', 'annotation'
    )),
    payload         JSONB NOT NULL,
    client_timestamp TIMESTAMPTZ NOT NULL,
    server_received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed       BOOLEAN NOT NULL DEFAULT false,
    processed_at    TIMESTAMPTZ,
    conflict_detected BOOLEAN NOT NULL DEFAULT false,
    conflict_resolution JSONB,
    -- conflict_resolution: { "strategy": "client_wins", "conflicting_field": "status",
    --   "server_value": "completed", "client_value": "in_progress" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sync_queue_device ON sync_queue(device_id);
CREATE INDEX idx_sync_queue_unprocessed ON sync_queue(processed, server_received_at)
    WHERE processed = false;
```

## Validation Strategy for JSONB Columns

Since PostgreSQL JSONB columns lack schema enforcement, validation must be handled at multiple layers:

1. **Application-layer validation:** Use JSON Schema validation (e.g., Ajv in Node.js, jsonschema in Python) before INSERT/UPDATE. Define schemas for each JSONB column and validate on write.

2. **CHECK constraints for critical fields:**
```sql
-- Ensure transform is a valid 4x4 matrix (array of 4 arrays, each with 4 numbers)
ALTER TABLE spatial_anchors ADD CONSTRAINT chk_transform_shape
    CHECK (jsonb_array_length(transform) = 4
       AND jsonb_array_length(transform->0) = 4
       AND jsonb_array_length(transform->1) = 4
       AND jsonb_array_length(transform->2) = 4
       AND jsonb_array_length(transform->3) = 4);

-- Ensure media is always an array
ALTER TABLE procedure_steps ADD CONSTRAINT chk_media_is_array
    CHECK (jsonb_typeof(media) = 'array');

-- Ensure translations is always an object
ALTER TABLE procedure_steps ADD CONSTRAINT chk_translations_is_object
    CHECK (jsonb_typeof(translations) = 'object');
```

3. **Trigger-based validation** for complex business rules on JSONB content when application-layer validation alone is insufficient.

## Query Examples

```sql
-- Find all procedures tagged "safety-critical" for a specific org
SELECT id, code, title FROM procedures
WHERE organization_id = $1 AND tags ? 'safety-critical';

-- Get a step with its Spanish translation
SELECT s.title, s.instruction_text,
       s.translations->'es'->>'title' AS title_es,
       s.translations->'es'->>'instruction_text' AS text_es
FROM procedure_steps s
WHERE s.id = $1;

-- Find all spatial anchors using object detection with confidence > 0.9
SELECT sa.id, sa.label, e.name AS equipment_name
FROM spatial_anchors sa
JOIN equipment e ON e.id = sa.equipment_id
WHERE sa.anchor_type = 'object_detection'
  AND (sa.anchor_config->>'confidence_threshold')::DOUBLE PRECISION > 0.9;

-- Dashboard: recent AI check failures with detection details
SELECT se.step_number, se.ai_result->>'check_result' AS result,
       (se.ai_result->>'confidence')::DOUBLE PRECISION AS confidence,
       se.ai_result->>'capture_url' AS capture
FROM step_executions se
WHERE se.execution_id = $1
  AND se.ai_result->>'check_result' = 'fail'
ORDER BY se.step_number;
```

## Scalability Considerations

- **Table partitioning:** Partition `procedure_executions` and `step_executions` by `started_at` using PostgreSQL native range partitioning. Monthly partitions for high-volume deployments, quarterly for smaller installations.
- **JSONB size awareness:** Monitor average JSONB column sizes. The `media` and `overlays_3d` arrays on `procedure_steps` should rarely exceed 10-20 items per step. If steps accumulate very large JSONB payloads, consider extracting to separate tables.
- **GIN index maintenance:** GIN indexes on JSONB are excellent for read-heavy workloads but add overhead on writes. For `step_executions.ai_result`, a partial GIN index (WHERE ai_result IS NOT NULL) avoids indexing the majority of rows that have no AI result.
- **Connection pooling:** Essential for AR device connections. Use PgBouncer in transaction-pooling mode.
- **Read replicas:** Route analytics queries (anomaly detection, compliance reporting, skill assessments) to read replicas.
- **TOAST compression:** PostgreSQL automatically compresses large JSONB values via TOAST. For very large point cloud metadata or annotation histories, consider storing references to object storage (S3/GCS) rather than inline JSONB.

## Migration Path

- **From normalized relational (Suggestion 1):** Straightforward migration. Combine the individual float columns (transform_m00..m33) into a JSONB `transform` column. Merge `step_media`, `step_3d_overlays`, `step_translations`, and `ai_inspection_configs` tables into JSONB columns on `procedure_steps`. Run as a phased migration: add JSONB columns, backfill data, update application to read from JSONB, drop old tables.
- **From event-sourced (Suggestion 2):** The read projections in Suggestion 2 already use JSONB for denormalized data. This schema can serve as the primary data store replacing the projection layer, while keeping the event store for audit if desired.
- **To fully relational:** If any JSONB column's structure stabilizes and needs database-level constraints, extract it to dedicated relational tables. For example, if `translations` grows to require per-language audit trails, extract to a `step_translations` table with foreign keys.
- **Migration tooling:** Standard tools (Flyway, Alembic, Prisma Migrate, Knex) handle JSONB columns natively. New fields within JSONB require no schema migration — just application code changes with backward-compatible defaults.
