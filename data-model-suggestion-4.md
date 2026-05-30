# Data Model Suggestion 4: Graph Database Model (Neo4j)

## Approach

A property graph database (Neo4j) where every domain entity is a node and every relationship is a first-class, queryable, attributed edge. This model treats the connections between entities — equipment-to-procedure assignments, operator skill qualifications, step sequencing, spatial anchor attachments, execution traces, and approval workflows — as primary data rather than join-table afterthoughts. Cypher queries express multi-hop traversals (e.g., "find all procedures an operator is qualified for on equipment at this site") in a fraction of the complexity of equivalent SQL JOINs.

## Why This Suits the Domain

AR Work Instructions is deeply relationship-driven in ways that make graph modeling compelling:

1. **Equipment-procedure-operator qualification matrix.** The core operational question is: "Given this operator at this equipment, which published procedure versions is the operator qualified to execute?" Answering this requires traversing organization -> site -> equipment -> category -> procedure -> version (published) -> minimum skill level, then cross-referencing operator -> skill assessments -> skill level. In a relational model this is a 6-7 table JOIN with conditional logic. In a graph, it is a single pattern match.

2. **Spatial anchor hierarchies.** Equipment can have multiple spatial anchors, each referenced by multiple 3D overlays across different procedure steps. A graph naturally represents the anchor -> overlay -> step -> version -> procedure chain without intermediate tables.

3. **Adaptive procedure logic.** The system adapts step ordering and detail based on operator skill and real-time context. Graph traversals can efficiently walk an operator's execution history, skill levels, and procedure complexity to determine the optimal step sequence.

4. **Anomaly detection and impact analysis.** When an anomaly is detected on a specific step, graph queries can immediately traverse to find all affected equipment, all operators who executed that step recently, and all related procedures — impact analysis that would require complex recursive CTEs in SQL.

5. **Versioning as graph structure.** Procedure versions form a natural chain: version 1 -> SUPERSEDED_BY -> version 2 -> SUPERSEDED_BY -> version 3. Rollback is finding the previous node in the chain. Approval workflows are edges: version -[REVIEWED_BY]-> user, version -[APPROVED_BY]-> user.

6. **Enterprise integration topology.** Webhook subscriptions, MES/ERP links, and work order associations are naturally edges connecting execution nodes to external system nodes.

## Trade-offs

**Strengths:**
- Relationship traversals are O(1) per hop regardless of total data volume — no full-table scans for JOINs
- Complex qualification, impact analysis, and dependency queries are concise and fast
- Schema-flexible properties on both nodes and relationships accommodate AR-specific variability
- Natural fit for recommendation engines (suggest procedures, identify training gaps)
- Visual graph exploration tools aid debugging and data understanding
- Versioning and approval workflows modeled as graph edges with temporal properties

**Weaknesses:**
- Less mature ecosystem for enterprise deployments compared to PostgreSQL (fewer DBAs, fewer managed services)
- Aggregation queries (count executions by month, average duration by procedure type) are slower than columnar/relational approaches
- No native ACID across multiple graph operations without explicit transaction management (Neo4j does support ACID transactions, but patterns differ from SQL)
- Bulk data loading (millions of execution records) requires careful batching with UNWIND or neo4j-admin import
- Full-text search requires separate index configuration (Neo4j has built-in full-text indexes but they are not as feature-rich as Elasticsearch)
- Backup and point-in-time recovery tooling is less standardized than PostgreSQL
- Team skill requirements: Cypher query language has a learning curve
- Less suitable for the high-write-throughput execution logging workload (consider a sidecar time-series store for raw telemetry)

## Schema Definition

### Node Labels and Properties

```cypher
// =============================================================
// ORGANIZATION & TENANCY
// =============================================================

// Organization node
CREATE CONSTRAINT org_id_unique FOR (o:Organization) REQUIRE o.id IS UNIQUE;
CREATE CONSTRAINT org_slug_unique FOR (o:Organization) REQUIRE o.slug IS UNIQUE;

// Properties:
// :Organization {
//   id: UUID (string),
//   name: String,
//   slug: String,
//   subscription_tier: String ("free"|"pro"|"enterprise"),
//   default_language: String,
//   created_at: DateTime,
//   updated_at: DateTime
// }

// Site node
CREATE CONSTRAINT site_id_unique FOR (s:Site) REQUIRE s.id IS UNIQUE;

// :Site {
//   id: UUID,
//   name: String,
//   location_label: String,
//   latitude: Float,
//   longitude: Float,
//   timezone: String,
//   is_active: Boolean,
//   created_at: DateTime,
//   updated_at: DateTime
// }

// Relationship: Organization -[:HAS_SITE]-> Site


// =============================================================
// USERS & ACCESS CONTROL
// =============================================================

CREATE CONSTRAINT user_id_unique FOR (u:User) REQUIRE u.id IS UNIQUE;
CREATE CONSTRAINT user_email_unique FOR (u:User) REQUIRE u.email IS UNIQUE;

// :User {
//   id: UUID,
//   email: String,
//   display_name: String,
//   password_hash: String,
//   auth_provider: String,
//   auth_provider_id: String,
//   locale: String,
//   is_active: Boolean,
//   department: String,
//   last_login_at: DateTime,
//   created_at: DateTime,
//   updated_at: DateTime
// }

// Relationships:
// Organization -[:HAS_USER]-> User
// User -[:HAS_ROLE {site_id, granted_at, granted_by}]-> Role

CREATE CONSTRAINT role_id_unique FOR (r:Role) REQUIRE r.id IS UNIQUE;

// :Role {
//   id: UUID,
//   name: String ("author"|"reviewer"|"operator"|"admin"),
//   description: String
// }

// :Permission {
//   name: String ("procedure.create"|"procedure.approve"|"execution.view"|...)
// }

CREATE CONSTRAINT perm_name_unique FOR (p:Permission) REQUIRE p.name IS UNIQUE;

// Role -[:GRANTS]-> Permission


// =============================================================
// SKILL TRACKING
// =============================================================

CREATE CONSTRAINT skill_id_unique FOR (sl:SkillLevel) REQUIRE sl.id IS UNIQUE;

// :SkillLevel {
//   id: UUID,
//   name: String ("novice"|"beginner"|"competent"|"proficient"|"expert"),
//   rank: Integer
// }

// Relationships:
// User -[:HAS_SKILL {
//   assessed_at: DateTime,
//   assessed_by: UUID,
//   method: String,
//   score: Float,
//   expiry_date: Date
// }]-> SkillLevel
//
// The HAS_SKILL edge can optionally connect through an EquipmentCategory:
// User -[:HAS_SKILL_FOR {assessed_at, assessed_by}]-> EquipmentCategory
//   and EquipmentCategory -[:AT_LEVEL]-> SkillLevel
// OR more simply, the HAS_SKILL edge carries an equipment_category_id property.


// =============================================================
// EQUIPMENT & SPATIAL ANCHORS
// =============================================================

CREATE CONSTRAINT equip_id_unique FOR (e:Equipment) REQUIRE e.id IS UNIQUE;

// :Equipment {
//   id: UUID,
//   name: String,
//   serial_number: String,
//   manufacturer: String,
//   model: String,
//   is_active: Boolean,
//   commissioned_at: Date,
//   firmware_version: String,
//   created_at: DateTime,
//   updated_at: DateTime
// }

// Relationships:
// Site -[:HAS_EQUIPMENT]-> Equipment
// Equipment -[:IN_CATEGORY]-> EquipmentCategory

CREATE CONSTRAINT eqcat_id_unique FOR (ec:EquipmentCategory) REQUIRE ec.id IS UNIQUE;

// :EquipmentCategory {
//   id: UUID,
//   name: String,
//   description: String
// }

// EquipmentCategory -[:PARENT_CATEGORY]-> EquipmentCategory  (hierarchical)
// Organization -[:HAS_CATEGORY]-> EquipmentCategory

CREATE CONSTRAINT anchor_id_unique FOR (a:SpatialAnchor) REQUIRE a.id IS UNIQUE;

// :SpatialAnchor {
//   id: UUID,
//   anchor_type: String ("qr_code"|"marker"|"object_detection"|"point_cloud"|"image_target"|"manual"),
//   label: String,
//   transform_matrix: List<Float>,  -- flattened 4x4 = 16 floats
//   is_active: Boolean,
//   created_at: DateTime,
//   updated_at: DateTime,
//   -- Type-specific properties (graph nodes are schema-flexible):
//   marker_value: String,           -- for qr_code / marker
//   marker_size_mm: Float,          -- for qr_code / marker
//   detection_model_url: String,    -- for object_detection
//   confidence_threshold: Float,    -- for object_detection
//   point_cloud_url: String,        -- for point_cloud
//   point_count: Integer,           -- for point_cloud
//   reference_image_url: String     -- for image_target
// }

// Equipment -[:HAS_ANCHOR]-> SpatialAnchor


// =============================================================
// PROCEDURES & VERSIONING
// =============================================================

CREATE CONSTRAINT proc_id_unique FOR (p:Procedure) REQUIRE p.id IS UNIQUE;
CREATE INDEX proc_code_idx FOR (p:Procedure) ON (p.code);

// :Procedure {
//   id: UUID,
//   code: String,
//   title: String,
//   description: String,
//   procedure_type: String ("maintenance"|"assembly"|"inspection"|"safety"|"calibration"|"custom"),
//   complexity: String ("basic"|"standard"|"advanced"|"expert"),
//   default_language: String,
//   source_document_url: String,
//   tags: List<String>,
//   is_archived: Boolean,
//   created_at: DateTime,
//   updated_at: DateTime
// }

// Relationships:
// Organization -[:HAS_PROCEDURE]-> Procedure
// Procedure -[:FOR_CATEGORY]-> EquipmentCategory
// Procedure -[:REQUIRES_SKILL]-> SkillLevel
// Procedure -[:CREATED_BY]-> User

CREATE CONSTRAINT procver_id_unique FOR (pv:ProcedureVersion) REQUIRE pv.id IS UNIQUE;

// :ProcedureVersion {
//   id: UUID,
//   version_number: Integer,
//   status: String ("draft"|"in_review"|"approved"|"published"|"superseded"|"withdrawn"),
//   change_summary: String,
//   published_at: DateTime,
//   created_at: DateTime,
//   updated_at: DateTime
// }

// Relationships:
// Procedure -[:HAS_VERSION]-> ProcedureVersion
// ProcedureVersion -[:SUPERSEDED_BY]-> ProcedureVersion  (version chain)
// ProcedureVersion -[:AUTHORED_BY]-> User
// ProcedureVersion -[:REVIEWED_BY {decision, comments, reviewed_at}]-> User
// ProcedureVersion -[:APPROVED_BY {approved_at}]-> User


// =============================================================
// STEPS, MEDIA & 3D OVERLAYS
// =============================================================

CREATE CONSTRAINT step_id_unique FOR (s:ProcedureStep) REQUIRE s.id IS UNIQUE;

// :ProcedureStep {
//   id: UUID,
//   step_number: Integer,
//   title: String,
//   instruction_text: String,
//   step_type: String ("action"|"inspection"|"decision"|"warning"|"info"|"ai_check"|"measurement"),
//   is_critical: Boolean,
//   estimated_duration_seconds: Integer,
//   can_skip: Boolean,
//   skip_condition: String,
//   safety_notes: String,
//   tool_requirements: String,
//   created_at: DateTime,
//   updated_at: DateTime
// }

// Relationships:
// ProcedureVersion -[:HAS_STEP {step_number}]-> ProcedureStep
// ProcedureStep -[:NEXT_STEP]-> ProcedureStep  (linked list for traversal)
// ProcedureStep -[:DECISION_BRANCH {condition, label}]-> ProcedureStep  (for decision steps)

CREATE CONSTRAINT media_id_unique FOR (m:Media) REQUIRE m.id IS UNIQUE;

// :Media {
//   id: UUID,
//   media_type: String ("image"|"video"|"audio"|"3d_model"|"animation"|"pdf"),
//   file_url: String,
//   thumbnail_url: String,
//   file_size_bytes: Integer,
//   mime_type: String,
//   caption: String,
//   duration_seconds: Float,
//   created_at: DateTime
// }

// ProcedureStep -[:HAS_MEDIA {display_order}]-> Media

CREATE CONSTRAINT overlay_id_unique FOR (ov:Overlay3D) REQUIRE ov.id IS UNIQUE;

// :Overlay3D {
//   id: UUID,
//   model_url: String,
//   model_format: String ("gltf"|"glb"|"usdz"),
//   compression: String ("draco"|"meshopt"|"none"),
//   offset_x: Float, offset_y: Float, offset_z: Float,
//   rotation_x: Float, rotation_y: Float, rotation_z: Float, rotation_w: Float,
//   scale_x: Float, scale_y: Float, scale_z: Float,
//   opacity: Float,
//   animation_name: String,
//   highlight_color: String,
//   created_at: DateTime
// }

// ProcedureStep -[:HAS_OVERLAY]-> Overlay3D
// Overlay3D -[:ANCHORED_TO]-> SpatialAnchor

// Translation node (one per language per step)
// :Translation {
//   language_code: String,
//   title: String,
//   instruction_text: String,
//   safety_notes: String,
//   translated_by: String ("manual"|"ai"|"professional"),
//   verified: Boolean,
//   created_at: DateTime
// }

// ProcedureStep -[:HAS_TRANSLATION]-> Translation


// =============================================================
// AI INSPECTION
// =============================================================

CREATE CONSTRAINT ai_config_id_unique FOR (ai:AIInspectionConfig) REQUIRE ai.id IS UNIQUE;

// :AIInspectionConfig {
//   id: UUID,
//   model_name: String,
//   model_version: String,
//   model_artifact_url: String,
//   pass_threshold: Float,
//   retry_allowed: Boolean,
//   max_retries: Integer,
//   capture_type: String ("photo"|"video_clip"|"depth_scan"),
//   reference_image_url: String,
//   created_at: DateTime
// }

// ProcedureStep -[:HAS_AI_INSPECTION]-> AIInspectionConfig


// =============================================================
// EXECUTION & AUDIT
// =============================================================

CREATE CONSTRAINT exec_id_unique FOR (ex:Execution) REQUIRE ex.id IS UNIQUE;
CREATE INDEX exec_started_idx FOR (ex:Execution) ON (ex.started_at);

// :Execution {
//   id: UUID,
//   status: String ("in_progress"|"completed"|"paused"|"aborted"|"failed"),
//   started_at: DateTime,
//   completed_at: DateTime,
//   total_duration_seconds: Integer,
//   device_type: String,
//   device_id: String,
//   os_version: String,
//   app_version: String,
//   is_offline: Boolean,
//   synced_at: DateTime,
//   notes: String,
//   created_at: DateTime
// }

// Relationships:
// Execution -[:OF_VERSION]-> ProcedureVersion
// Execution -[:ON_EQUIPMENT]-> Equipment
// Execution -[:PERFORMED_BY]-> User

CREATE CONSTRAINT stepexec_id_unique FOR (se:StepExecution) REQUIRE se.id IS UNIQUE;

// :StepExecution {
//   id: UUID,
//   step_number: Integer,
//   status: String ("pending"|"in_progress"|"passed"|"failed"|"skipped"|"retried"),
//   started_at: DateTime,
//   completed_at: DateTime,
//   duration_seconds: Integer,
//   was_skipped: Boolean,
//   skip_reason: String,
//   operator_notes: String,
//   ai_check_result: String ("pass"|"fail"|"inconclusive"),
//   ai_confidence: Float,
//   ai_capture_url: String,
//   created_at: DateTime
// }

// Execution -[:HAS_STEP_EXECUTION]-> StepExecution
// StepExecution -[:FOR_STEP]-> ProcedureStep
// StepExecution -[:NEXT_EXECUTION]-> StepExecution  (execution sequence chain)


// =============================================================
// REMOTE EXPERT SESSIONS
// =============================================================

CREATE CONSTRAINT session_id_unique FOR (rs:RemoteExpertSession) REQUIRE rs.id IS UNIQUE;

// :RemoteExpertSession {
//   id: UUID,
//   status: String ("requested"|"active"|"ended"|"missed"),
//   started_at: DateTime,
//   ended_at: DateTime,
//   recording_url: String,
//   created_at: DateTime
// }

// Execution -[:HAD_EXPERT_SESSION]-> RemoteExpertSession
// RemoteExpertSession -[:OPERATOR]-> User
// RemoteExpertSession -[:EXPERT]-> User

// :Annotation {
//   id: UUID,
//   annotation_type: String ("arrow"|"circle"|"text"|"freehand"|"3d_pointer"),
//   position_x: Float, position_y: Float, position_z: Float,
//   content: String,
//   color: String,
//   timestamp_ms: Integer,
//   created_at: DateTime
// }

// RemoteExpertSession -[:HAS_ANNOTATION]-> Annotation
// Annotation -[:AT_ANCHOR]-> SpatialAnchor


// =============================================================
// ENTERPRISE INTEGRATION
// =============================================================

// :ExternalSystem {
//   name: String ("sap"|"oracle"|"mes_custom"),
//   base_url: String,
//   system_type: String ("erp"|"mes"|"cmms")
// }

CREATE CONSTRAINT extsys_name_unique FOR (es:ExternalSystem) REQUIRE es.name IS UNIQUE;

// :WorkOrder {
//   id: UUID,
//   external_id: String,
//   external_url: String,
//   linked_at: DateTime
// }

// Execution -[:LINKED_TO_WORK_ORDER]-> WorkOrder
// WorkOrder -[:FROM_SYSTEM]-> ExternalSystem

// :WebhookConfig {
//   id: UUID,
//   name: String,
//   target_url: String,
//   event_types: List<String>,
//   secret_hash: String,
//   is_active: Boolean,
//   created_at: DateTime
// }

// Organization -[:HAS_WEBHOOK]-> WebhookConfig


// =============================================================
// ANOMALY DETECTION
// =============================================================

CREATE CONSTRAINT anomaly_id_unique FOR (a:AnomalyAlert) REQUIRE a.id IS UNIQUE;

// :AnomalyAlert {
//   id: UUID,
//   alert_type: String ("step_failure_spike"|"duration_drift"|"skip_pattern"|"quality_degradation"),
//   severity: String ("info"|"warning"|"critical"),
//   description: String,
//   metric_value: Float,
//   baseline_value: Float,
//   sample_size: Integer,
//   acknowledged_at: DateTime,
//   resolution_notes: String,
//   created_at: DateTime
// }

// Organization -[:HAS_ANOMALY]-> AnomalyAlert
// AnomalyAlert -[:AFFECTS_PROCEDURE]-> Procedure
// AnomalyAlert -[:AFFECTS_STEP]-> ProcedureStep
// AnomalyAlert -[:ACKNOWLEDGED_BY]-> User


// =============================================================
// CAD IMPORTS
// =============================================================

CREATE CONSTRAINT cad_id_unique FOR (c:CADImport) REQUIRE c.id IS UNIQUE;

// :CADImport {
//   id: UUID,
//   source_system: String,
//   source_filename: String,
//   source_format: String,
//   output_gltf_url: String,
//   conversion_status: String ("pending"|"processing"|"completed"|"failed"),
//   polygon_count: Integer,
//   file_size_bytes: Integer,
//   created_at: DateTime
// }

// Organization -[:HAS_CAD_IMPORT]-> CADImport
// CADImport -[:IMPORTED_BY]-> User
// CADImport -[:PRODUCES_MODEL]-> Media  (links to resulting 3D model)
```

### Full Relationship Summary

```
Organization -[:HAS_SITE]-> Site
Organization -[:HAS_USER]-> User
Organization -[:HAS_CATEGORY]-> EquipmentCategory
Organization -[:HAS_PROCEDURE]-> Procedure
Organization -[:HAS_WEBHOOK]-> WebhookConfig
Organization -[:HAS_ANOMALY]-> AnomalyAlert
Organization -[:HAS_CAD_IMPORT]-> CADImport

Site -[:HAS_EQUIPMENT]-> Equipment

User -[:HAS_ROLE]-> Role                          {site_id, granted_at, granted_by}
User -[:HAS_SKILL]-> SkillLevel                   {equipment_category_id, assessed_at, assessed_by, method, score}
Role -[:GRANTS]-> Permission

EquipmentCategory -[:PARENT_CATEGORY]-> EquipmentCategory
Equipment -[:IN_CATEGORY]-> EquipmentCategory
Equipment -[:HAS_ANCHOR]-> SpatialAnchor

Procedure -[:FOR_CATEGORY]-> EquipmentCategory
Procedure -[:REQUIRES_SKILL]-> SkillLevel
Procedure -[:CREATED_BY]-> User
Procedure -[:HAS_VERSION]-> ProcedureVersion

ProcedureVersion -[:SUPERSEDED_BY]-> ProcedureVersion
ProcedureVersion -[:AUTHORED_BY]-> User
ProcedureVersion -[:REVIEWED_BY]-> User            {decision, comments, reviewed_at}
ProcedureVersion -[:APPROVED_BY]-> User            {approved_at}
ProcedureVersion -[:HAS_STEP]-> ProcedureStep      {step_number}

ProcedureStep -[:NEXT_STEP]-> ProcedureStep
ProcedureStep -[:DECISION_BRANCH]-> ProcedureStep  {condition, label}
ProcedureStep -[:HAS_MEDIA]-> Media                {display_order}
ProcedureStep -[:HAS_OVERLAY]-> Overlay3D
ProcedureStep -[:HAS_TRANSLATION]-> Translation
ProcedureStep -[:HAS_AI_INSPECTION]-> AIInspectionConfig

Overlay3D -[:ANCHORED_TO]-> SpatialAnchor

Execution -[:OF_VERSION]-> ProcedureVersion
Execution -[:ON_EQUIPMENT]-> Equipment
Execution -[:PERFORMED_BY]-> User
Execution -[:HAS_STEP_EXECUTION]-> StepExecution
Execution -[:HAD_EXPERT_SESSION]-> RemoteExpertSession
Execution -[:LINKED_TO_WORK_ORDER]-> WorkOrder

StepExecution -[:FOR_STEP]-> ProcedureStep
StepExecution -[:NEXT_EXECUTION]-> StepExecution

RemoteExpertSession -[:OPERATOR]-> User
RemoteExpertSession -[:EXPERT]-> User
RemoteExpertSession -[:HAS_ANNOTATION]-> Annotation
Annotation -[:AT_ANCHOR]-> SpatialAnchor

WorkOrder -[:FROM_SYSTEM]-> ExternalSystem
CADImport -[:IMPORTED_BY]-> User
CADImport -[:PRODUCES_MODEL]-> Media

AnomalyAlert -[:AFFECTS_PROCEDURE]-> Procedure
AnomalyAlert -[:AFFECTS_STEP]-> ProcedureStep
AnomalyAlert -[:ACKNOWLEDGED_BY]-> User
```

### Key Query Examples

```cypher
// ---------------------------------------------------------------
// 1. Find all published procedures an operator is qualified to run
//    on a specific piece of equipment
// ---------------------------------------------------------------
MATCH (u:User {id: $operatorId})-[:HAS_SKILL]->(sl:SkillLevel)
MATCH (e:Equipment {id: $equipmentId})-[:IN_CATEGORY]->(ec:EquipmentCategory)
MATCH (p:Procedure)-[:FOR_CATEGORY]->(ec)
MATCH (p)-[:REQUIRES_SKILL]->(reqSkill:SkillLevel)
WHERE sl.rank >= reqSkill.rank
MATCH (p)-[:HAS_VERSION]->(pv:ProcedureVersion {status: 'published'})
RETURN p.code, p.title, pv.version_number
ORDER BY p.code;

// ---------------------------------------------------------------
// 2. Load a full procedure for AR playback (single traversal)
// ---------------------------------------------------------------
MATCH (pv:ProcedureVersion {id: $versionId})-[:HAS_STEP]->(step:ProcedureStep)
OPTIONAL MATCH (step)-[:HAS_MEDIA]->(m:Media)
OPTIONAL MATCH (step)-[:HAS_OVERLAY]->(ov:Overlay3D)-[:ANCHORED_TO]->(anchor:SpatialAnchor)
OPTIONAL MATCH (step)-[:HAS_AI_INSPECTION]->(ai:AIInspectionConfig)
OPTIONAL MATCH (step)-[:HAS_TRANSLATION]->(t:Translation)
RETURN step, collect(DISTINCT m) AS media,
       collect(DISTINCT {overlay: ov, anchor: anchor}) AS overlays,
       ai, collect(DISTINCT t) AS translations
ORDER BY step.step_number;

// ---------------------------------------------------------------
// 3. Impact analysis: which operators are affected by a step defect?
// ---------------------------------------------------------------
MATCH (s:ProcedureStep {id: $stepId})<-[:FOR_STEP]-(se:StepExecution)
      <-[:HAS_STEP_EXECUTION]-(ex:Execution)-[:PERFORMED_BY]->(u:User)
WHERE ex.started_at > datetime() - duration('P30D')
RETURN DISTINCT u.display_name, u.email, count(se) AS execution_count
ORDER BY execution_count DESC;

// ---------------------------------------------------------------
// 4. Operator skill gap analysis
// ---------------------------------------------------------------
MATCH (u:User {id: $operatorId})
OPTIONAL MATCH (u)-[hs:HAS_SKILL]->(sl:SkillLevel)
WITH u, collect({level: sl.name, rank: sl.rank, category: hs.equipment_category_id}) AS skills
MATCH (p:Procedure)-[:REQUIRES_SKILL]->(reqSkill:SkillLevel)
MATCH (p)-[:FOR_CATEGORY]->(ec:EquipmentCategory)
WHERE NOT any(s IN skills WHERE s.rank >= reqSkill.rank
      AND (s.category IS NULL OR s.category = ec.id))
RETURN p.code, p.title, ec.name AS category,
       reqSkill.name AS required_level;

// ---------------------------------------------------------------
// 5. Version history with approval chain
// ---------------------------------------------------------------
MATCH (p:Procedure {code: $procedureCode})-[:HAS_VERSION]->(pv:ProcedureVersion)
OPTIONAL MATCH (pv)-[:AUTHORED_BY]->(author:User)
OPTIONAL MATCH (pv)-[rev:REVIEWED_BY]->(reviewer:User)
OPTIONAL MATCH (pv)-[app:APPROVED_BY]->(approver:User)
OPTIONAL MATCH (pv)-[:SUPERSEDED_BY]->(next:ProcedureVersion)
RETURN pv.version_number, pv.status,
       author.display_name AS author,
       reviewer.display_name AS reviewer, rev.decision,
       approver.display_name AS approver, app.approved_at,
       next.version_number AS superseded_by
ORDER BY pv.version_number;

// ---------------------------------------------------------------
// 6. Equipment with all anchors and linked procedures (AR device bootstrap)
// ---------------------------------------------------------------
MATCH (e:Equipment {id: $equipmentId})
MATCH (e)-[:HAS_ANCHOR]->(a:SpatialAnchor)
MATCH (e)-[:IN_CATEGORY]->(ec:EquipmentCategory)<-[:FOR_CATEGORY]-(p:Procedure)
MATCH (p)-[:HAS_VERSION]->(pv:ProcedureVersion {status: 'published'})
RETURN e, collect(DISTINCT a) AS anchors,
       collect(DISTINCT {procedure: p, version: pv}) AS procedures;
```

## Indexing Strategy

```cypher
// Composite indexes for frequent query patterns
CREATE INDEX exec_status_date FOR (ex:Execution) ON (ex.status, ex.started_at);
CREATE INDEX step_exec_status FOR (se:StepExecution) ON (se.status);
CREATE INDEX proc_type_archived FOR (p:Procedure) ON (p.procedure_type, p.is_archived);
CREATE INDEX anchor_type FOR (a:SpatialAnchor) ON (a.anchor_type);
CREATE INDEX anomaly_severity FOR (a:AnomalyAlert) ON (a.severity, a.created_at);

// Full-text indexes for natural language search across procedure libraries
CREATE FULLTEXT INDEX procedure_search FOR (p:Procedure)
    ON EACH [p.title, p.description, p.code];
CREATE FULLTEXT INDEX step_search FOR (s:ProcedureStep)
    ON EACH [s.title, s.instruction_text, s.safety_notes, s.tool_requirements];
```

## Scalability Considerations

- **Read scaling:** Neo4j supports causal clustering with read replicas. AR playback queries (read-heavy) route to replicas; authoring and execution writes go to the leader.
- **Execution volume:** Execution and StepExecution nodes will dominate the graph by volume. Consider time-based subgraphs or archival: move completed execution nodes older than N months to a separate archive database or export to a data lake for analytics.
- **Sharding:** Neo4j Fabric (Enterprise) allows sharding by organization/tenant, distributing large multi-tenant deployments across graph shards. Each organization's subgraph is largely self-contained.
- **Caching:** Neo4j's page cache should be sized to hold the working set (procedure definitions, active equipment, recent executions). Procedure content is read far more often than written.
- **Batch imports:** For initial data migration or bulk execution imports, use `neo4j-admin database import` or `CALL apoc.periodic.iterate` for batched Cypher imports rather than individual CREATE statements.
- **Analytics offload:** For heavy aggregation (monthly reports, KPI dashboards), export execution data to a columnar store (ClickHouse, BigQuery) or maintain a PostgreSQL analytics replica. The graph excels at traversal, not aggregation.

## Recommended Companion Architecture

For production deployment, pair Neo4j with:
1. **PostgreSQL or ClickHouse** for time-series execution analytics and compliance reporting (aggregation-heavy queries).
2. **Object storage (S3/GCS)** for binary assets (3D models, point clouds, captured images, recordings).
3. **Redis** for session caching, real-time execution state, and WebRTC signaling.
4. **Message queue (NATS/RabbitMQ)** for webhook delivery, CAD conversion jobs, and anomaly detection pipeline.

This polyglot approach uses each database for its strength: Neo4j for relationship traversal and operational queries, a columnar store for analytics, and object storage for large binaries.

## Migration Path

- **From relational (Suggestions 1 or 3):** Use `LOAD CSV` or the APOC JDBC connector to import relational tables into graph nodes. Foreign key relationships become graph edges. The migration is conceptually straightforward: each table becomes a node label, each foreign key becomes a relationship type. Run both systems in parallel during transition with a sync layer.
- **From event-sourced (Suggestion 2):** Replay the event store to build the graph. Each event creates or updates nodes and relationships. The event stream becomes the source of truth; the graph is a materialized view. This is a natural fit since event-sourced systems already separate writes from reads.
- **To hybrid:** If aggregation queries prove too slow, keep Neo4j for operational/traversal queries and add a PostgreSQL read replica (populated via change data capture from Neo4j triggers or application-level dual writes) for reporting. This is the most common production pattern for graph-backed applications.
- **Exit strategy:** Export the graph to CSV/JSON using `CALL apoc.export.*` procedures. Node properties map directly to table columns; relationship types map to foreign keys or join tables. The graph model is fully reversible to relational.
