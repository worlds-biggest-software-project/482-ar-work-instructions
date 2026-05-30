# AR Work Instructions — Phased Development Plan

> Project: 482-ar-work-instructions · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a concrete, buildable roadmap. The database design follows **Data Model Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** — chosen because the organisational/workflow backbone (orgs → sites → equipment → procedures → versions → steps → executions) is strongly relational, while AR-specific content (spatial-anchor transforms, 3D overlay placement, AI-check parameters, multi-language translation bundles) is polymorphic and best held in validated JSONB.

---

## Product Summary

**What it does:** An open-source, AR-native platform that renders step-by-step maintenance, assembly, inspection, and safety procedures directly onto physical equipment in a worker's field of view, replacing paper manuals and static PDFs. Authors build procedures in a no-code web editor; operators play them back via WebAR on mobile or via WebXR on headsets, with spatial anchoring to equipment, offline caching, and timestamped completion records for audit.

**Primary personas:**
- **Process engineer / author** — creates and maintains procedures (web, no headset).
- **Reviewer / approver** — signs off procedure versions before publication.
- **Operator / technician** — executes procedures on the plant floor (mobile/headset).
- **Operations director** — consumes completion analytics and anomaly alerts.
- **Integrator / admin** — wires up MES/ERP webhooks, SSO, and deployment.

**Key differentiators (the AI-native + open advantage):**
1. Only credible open-source AR work-instruction platform — no vendor lock-in, self-hostable including air-gapped.
2. Built on open standards from day one: glTF 2.0 (ISO/IEC 12113:2022), WebXR (W3C), WebRTC (W3C/IETF), OpenAPI 3.1, OAuth 2.0 / OIDC, GS1 Digital Link, OPC UA.
3. Unified AI: SOP-to-procedure conversion, runtime translation, computer-vision step inspection, and completion-data anomaly detection in one stack rather than bolted-on add-ons.
4. WebAR delivery without app installation; portable, open procedure-content export format to fight the industry's interchange gap.

**Deployment model:** Self-hosted / cloud / hybrid, including air-gapped on-prem. Single Docker Compose stack for MVP; Helm chart for production.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Backend language | **Python 3.12** | AI workloads (SOP conversion, CV inspection, anomaly detection) dominate the differentiating roadmap; Python has the richest ML/LLM ecosystem. Keeps AI code in-process rather than a second polyglot service. |
| API framework | **FastAPI** | First-class OpenAPI 3.1 generation (required by standards.md), async for webhook/WebRTC-signalling/long AI calls, Pydantic validation that doubles as JSONB-payload schema enforcement. |
| Database | **PostgreSQL 16** | Hybrid relational + JSONB model (Suggestion 3). Single technology, GIN indexes for JSONB containment queries, partitioning for execution time-series, strong audit/referential integrity for ISO 9001 / GMP. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Mature async ORM with native JSONB support; Alembic gives the versioned, reviewable migrations a compliance product needs. |
| Task queue | **Celery + Redis** | Async workloads: AI SOP conversion, CAD→glTF transcode, video segmentation, translation, webhook delivery with retry, anomaly batch jobs. Redis doubles as cache and WebRTC-signalling pub/sub. |
| Object storage | **S3-compatible (MinIO bundled)** | glTF models, media, point clouds, AI capture images. MinIO ships in the self-hosted stack; swappable for AWS S3/Azure Blob via env. |
| Frontend (authoring) | **React 18 + TypeScript + Vite** | The no-code WYSIWYG editor is an SPA-heavy dashboard; React's ecosystem (react-three-fiber for 3D preview) fits. |
| 3D / AR rendering | **three.js + react-three-fiber; WebXR via three.js WebXR; model-viewer for glTF preview** | Open WebXR `immersive-ar` sessions (W3C AR Module L1), glTF 2.0 loader with Draco decode, hit-testing, and headset support — no proprietary engine, no app install. |
| Marker / QR detection | **zxing-wasm (QR / GS1 Digital Link) + MediaPipe / ONNX Runtime Web (object detection)** | Browser-side fiducial + marker-free detection; GS1 Digital Link decode maps a scanned code straight to an asset + procedure. |
| Remote expert video | **WebRTC (browser-native) + mediasoup-style SFU optional; aiortc-free signalling server in FastAPI** | Sub-300 ms peer video with AR annotation overlay; signalling over Redis pub/sub + WebSocket. |
| AI / LLM access | **Provider-abstracted client (OpenAI-compatible + Anthropic + local Ollama/vLLM)** | Air-gapped deployments must run fully local models; an abstraction layer lets the same code call a cloud or on-prem endpoint. |
| Auth | **OAuth 2.0 / OIDC (Authlib) + local password fallback; FIDO2/WebAuthn for sign-off MFA** | RFC 6749 + OIDC for enterprise SSO (Azure AD, Okta); NIST SP 800-63B AAL2/AAL3 sign-off via WebAuthn for GMP/CMMC. |
| Containerisation | **Docker + Docker Compose (dev/self-host), Helm (prod)** | Standard for self-hosted; air-gapped images can be exported as tarballs. |
| Testing | **pytest + pytest-asyncio + httpx (backend); Vitest + Playwright (frontend/E2E)** | Standard, async-aware; Playwright drives the authoring UI and WebAR fallback flows. |
| Code quality | **ruff (lint+format) + mypy (backend); eslint + prettier + tsc (frontend)** | Fast, opinionated, type-checked. |
| Package mgmt | **uv (Python) / pnpm (JS)** | Fast, lockfile-based, reproducible builds for air-gapped mirroring. |
| Real-time transport | **WebSockets (FastAPI) over Redis pub/sub** | Live execution sync, remote-expert signalling, annotation streams. |

### Project Structure

```
ar-work-instructions/
├── README.md
├── docker-compose.yml                 # postgres, redis, minio, api, worker, web
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── pyproject.toml                     # uv-managed backend deps
├── alembic.ini
├── helm/                              # production chart
├── docs/
│   └── openapi.json                   # generated; committed for SDK gen
├── backend/
│   ├── arwi/
│   │   ├── main.py                    # FastAPI app factory, router mount
│   │   ├── config.py                  # pydantic-settings (env)
│   │   ├── db/
│   │   │   ├── session.py             # async engine, session factory
│   │   │   ├── base.py                # declarative base
│   │   │   └── models/                # SQLAlchemy models (one file per domain)
│   │   ├── schemas/                   # Pydantic request/response + JSONB validators
│   │   ├── api/
│   │   │   ├── deps.py                # auth, db, rbac dependencies
│   │   │   └── routes/                # auth, orgs, equipment, anchors,
│   │   │       │                      #   procedures, versions, steps, media,
│   │   │       │                      #   executions, webhooks, expert, ai, analytics
│   │   ├── services/                  # business logic (procedure, version,
│   │   │       │                      #   execution, rbac, sync, anomaly)
│   │   ├── ai/
│   │   │   ├── client.py              # provider-abstracted LLM/CV client
│   │   │   ├── sop_convert.py         # PDF/Word → structured draft
│   │   │   ├── translate.py           # runtime step translation
│   │   │   ├── inspection.py          # CV step pass/fail
│   │   │   └── anomaly.py             # completion-data anomaly detection
│   │   ├── integrations/
│   │   │   ├── webhooks.py            # MES/ERP inbound + outbound delivery
│   │   │   ├── cad.py                 # STEP/CAD → glTF transcode jobs
│   │   │   └── opcua.py               # live sensor overlay (backlog)
│   │   ├── realtime/
│   │   │   ├── ws.py                  # WebSocket manager
│   │   │   └── signalling.py          # WebRTC signalling
│   │   ├── storage/                   # S3/MinIO client
│   │   ├── tasks/                     # Celery tasks
│   │   └── export/                    # portable procedure package (.arwi)
│   └── tests/
│       ├── unit/
│       ├── integration/
│       └── fixtures/
├── web/
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── app/                       # routing, providers
│   │   ├── features/
│   │   │   ├── auth/
│   │   │   ├── procedures/            # list, editor (WYSIWYG), versioning
│   │   │   ├── editor3d/              # react-three-fiber anchor/overlay placement
│   │   │   ├── player/                # WebAR/WebXR runtime player
│   │   │   ├── equipment/
│   │   │   ├── analytics/
│   │   │   └── expert/                # remote expert call UI
│   │   ├── lib/                       # api client (generated from openapi.json)
│   │   ├── ar/                        # WebXR session, hit-test, anchor recovery
│   │   └── offline/                   # service worker, IndexedDB cache, sync
│   └── tests/
└── packages/
    └── arwi-content-spec/             # open procedure interchange format spec + JSON Schema
```

---

## Phase 1: Foundation — Project Skeleton, Config, Database Core

### Purpose
Stand up the runnable skeleton: containerised Postgres/Redis/MinIO, FastAPI app that boots and serves a health endpoint and an OpenAPI 3.1 spec, the SQLAlchemy declarative base, Alembic migrations, and the multi-tenant core tables (organizations, sites, users). After this phase the system runs, connects to the database, and exposes an empty-but-valid API surface. Everything else builds additively on this.

### Tasks

#### 1.1 — Repository scaffold, tooling, and Docker stack

**What:** Create the directory tree, dependency manifests, and a `docker-compose.yml` that brings up postgres, redis, minio, api, and worker.

**Design:**
- `pyproject.toml` deps: `fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`, `pydantic-settings`, `celery[redis]`, `redis`, `boto3`, `authlib`, `python-multipart`, `httpx`. Dev group: `pytest`, `pytest-asyncio`, `httpx`, `ruff`, `mypy`, `testcontainers`.
- `arwi/config.py` using `pydantic-settings`:

```python
class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://arwi:arwi@postgres:5432/arwi"
    redis_url: str = "redis://redis:6379/0"
    s3_endpoint: str = "http://minio:9000"
    s3_bucket: str = "arwi"
    s3_access_key: str = "minioadmin"
    s3_secret_key: str = "minioadmin"
    jwt_secret: str
    oidc_issuer: str | None = None
    ai_provider: Literal["openai", "anthropic", "ollama"] = "ollama"
    ai_base_url: str | None = None
    ai_api_key: str | None = None
    cors_origins: list[str] = ["http://localhost:5173"]
    model_config = SettingsConfigDict(env_prefix="ARWI_", env_file=".env")
```

- `docker-compose.yml` services with healthchecks; api depends_on postgres/redis/minio healthy.
- Makefile/`uv run` targets: `dev`, `test`, `lint`, `migrate`, `revision`.

**Testing:**
- `Unit: Settings loads from env vars with ARWI_ prefix → correct values; missing jwt_secret → ValidationError`.
- `Integration: docker compose up → GET /health returns 200 within healthcheck window`.
- `Integration: api container can open a TCP connection to postgres, redis, minio on boot`.

#### 1.2 — FastAPI app factory, health, and OpenAPI export

**What:** App factory that mounts routers, configures CORS, and emits OpenAPI 3.1.

**Design:**
- `create_app() -> FastAPI` with `openapi_version="3.1.0"`, title, version.
- `GET /health` → `{"status": "ok", "db": "ok|down", "redis": "ok|down"}` (pings each dependency).
- A `scripts/export_openapi.py` writes `docs/openapi.json` (consumed by the frontend client generator).
- Global exception handler producing RFC-7807-style problem JSON.

**Testing:**
- `Unit: GET /health with all deps up → 200, all "ok"`.
- `Unit (mocked): db ping raises → /health returns 200 body db="down", overall status "degraded"`.
- `Unit: exported openapi.json validates against OpenAPI 3.1 meta-schema`.

#### 1.3 — Database base, session management, Alembic

**What:** Async engine/session, declarative base with shared mixins, Alembic configured for autogenerate.

**Design:**

```python
class Base(DeclarativeBase): ...

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

class UUIDPKMixin:
    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, server_default=text("gen_random_uuid()"))
```

- `get_session()` async dependency yielding `AsyncSession`.
- Alembic env wired to async engine and `Base.metadata`.

**Testing:**
- `Integration (testcontainers postgres): create_all then a round-trip insert/select on a probe table`.
- `Integration: alembic upgrade head on empty DB → all core tables exist; downgrade base → clean`.

#### 1.4 — Tenancy core: organizations, sites, users

**What:** Implement the org/site/user tables and CRUD routes (auth wiring comes in Phase 2; here routes are open in dev only, gated by a feature flag).

**Design:** Follows Suggestion 3 JSONB-on-`settings` pattern.

```sql
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(100) NOT NULL UNIQUE,
  subscription_tier VARCHAR(50) NOT NULL DEFAULT 'free',
  settings JSONB NOT NULL DEFAULT '{}',   -- default_language, features{}, branding{}
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE sites (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  location JSONB NOT NULL DEFAULT '{}',    -- {label, lat, lon, timezone}
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sites_org ON sites(organization_id);
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  email VARCHAR(320) NOT NULL,
  display_name VARCHAR(255) NOT NULL,
  password_hash VARCHAR(255),
  auth_provider VARCHAR(50) NOT NULL DEFAULT 'local',
  auth_provider_id VARCHAR(255),
  locale VARCHAR(10) NOT NULL DEFAULT 'en',
  is_active BOOLEAN NOT NULL DEFAULT true,
  last_login_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (organization_id, email)
);
```

- Pydantic `OrganizationCreate/Read`, `SiteCreate/Read`, `UserCreate/Read`. A `SettingsModel` Pydantic class validates the `settings` JSONB on write.
- Routes: `POST/GET/PATCH /orgs`, `/orgs/{id}/sites`, `/orgs/{id}/users`.

**Testing:**
- `Unit: OrganizationCreate with duplicate slug → service raises ConflictError → 409`.
- `Unit: settings JSONB with unknown feature flag → SettingsModel rejects → 422`.
- `Integration: create org, then create site under it; deleting org cascades sites`.
- `Integration: create user with email duplicate within same org → 409; same email different org → 201`.

---

## Phase 2: Identity, RBAC, and Skills

### Purpose
Make the platform multi-user and secure. Add OAuth 2.0 / OIDC login (RFC 6749) with local-password fallback, JWT session issuance, role-based access control (author / reviewer / operator / admin) scoped per-site, and the skills model that later drives skills-based procedure assignment. After this phase every API route is authorised, and the matrix of who-can-do-what is enforced.

### Tasks

#### 2.1 — Authentication (OIDC + local) and sessions

**What:** Login via configured OIDC provider or local credentials; issue short-lived access JWT + refresh token.

**Design:**
- `Authlib` OIDC client keyed on `ARWI_OIDC_ISSUER`. Local fallback hashes with `argon2`.
- `POST /auth/login` (local) → `{access_token, refresh_token, expires_in}`. `GET /auth/oidc/start` → redirect; `GET /auth/oidc/callback` → exchanges code, upserts user by `auth_provider_id`.
- `POST /auth/refresh`, `POST /auth/logout` (revokes refresh in Redis).
- Access JWT claims: `sub`(user id), `org`, `roles`(list), `aal`(1|2|3).

**Testing:**
- `Unit: valid local creds → tokens issued; wrong password → 401, no token`.
- `Integration (mocked OIDC): callback with valid code → user upserted, tokens issued`.
- `Integration: expired access token → 401; valid refresh → new access token`.
- `Integration: revoked refresh token reuse → 401`.

#### 2.2 — RBAC model and enforcement

**What:** Roles, permissions, user_roles (site-scoped), and a FastAPI dependency that enforces a required permission.

**Design:**

```sql
CREATE TABLE roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL,  -- author|reviewer|operator|admin
  description TEXT,
  UNIQUE (organization_id, name)
);
CREATE TABLE permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL UNIQUE,  -- procedure.create, procedure.approve, execution.run ...
  description TEXT
);
CREATE TABLE role_permissions (
  role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
  permission_id UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
  PRIMARY KEY (role_id, permission_id)
);
CREATE TABLE user_roles (
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
  site_id UUID REFERENCES sites(id) ON DELETE CASCADE,  -- NULL = org-wide
  granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  granted_by UUID REFERENCES users(id),
  PRIMARY KEY (user_id, role_id, site_id)
);
```

- Dependency: `require(permission: str, site_param: str | None = None)` → resolves the caller's effective permissions (union of role permissions, filtered by site scope) and raises 403 if missing.
- Seed migration inserts the canonical permission set and default roles per new org.

**Testing:**
- `Unit: operator calls procedure.approve → 403`.
- `Unit: reviewer with site-scoped role approves procedure at another site → 403`.
- `Unit: admin (org-wide) passes any permission check`.
- `Integration: newly created org auto-seeds author/reviewer/operator/admin roles`.

#### 2.3 — Skills model

**What:** Skill levels and per-user skill assessments, foundation for adaptive/assignment features.

**Design:**

```sql
CREATE TABLE skill_levels (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL, rank INT NOT NULL,
  UNIQUE (organization_id, name)
);
CREATE TABLE user_skills (
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  skill_level_id UUID NOT NULL REFERENCES skill_levels(id) ON DELETE CASCADE,
  equipment_category_id UUID,
  assessed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  assessed_by UUID REFERENCES users(id),
  PRIMARY KEY (user_id, skill_level_id)
);
```

- Routes: `GET/POST /orgs/{id}/skill-levels`, `POST /users/{id}/skills`.

**Testing:**
- `Unit: assign skill above defined rank range → 422`.
- `Integration: assess user skill, read back ordered by rank`.

---

## Phase 3: Equipment, Spatial Anchors, and the Open Content Model

### Purpose
Model the physical world and how procedures attach to it. Add equipment with categories, polymorphic spatial anchors (QR / GS1 Digital Link / marker / object-detection / point-cloud), and define the open procedure-content interchange format (`arwi-content-spec`) that fights the industry interchange gap. This phase makes "what equipment exists and how do we recognise it" a first-class, queryable concept that procedures (Phase 4) and the player (Phase 5/6) bind to.

### Tasks

#### 3.1 — Equipment and categories

**What:** Equipment registry with self-referential category tree.

**Design:**

```sql
CREATE TABLE equipment_categories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL, description TEXT,
  parent_id UUID REFERENCES equipment_categories(id),
  UNIQUE (organization_id, name, parent_id)
);
CREATE TABLE equipment (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  site_id UUID NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
  category_id UUID REFERENCES equipment_categories(id),
  name VARCHAR(255) NOT NULL,
  identifiers JSONB NOT NULL DEFAULT '{}',  -- {serial, manufacturer, model, gtin}
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_equipment_site ON equipment(site_id);
CREATE INDEX idx_equipment_gtin ON equipment ((identifiers->>'gtin'));
```

**Testing:**
- `Unit: create category cycle (A parent of B, B parent of A) → service rejects → 422`.
- `Integration: lookup equipment by GTIN uses the expression index`.

#### 3.2 — Spatial anchors (polymorphic, JSONB)

**What:** Anchors of varying shape per type, with a 4×4 transform stored as a JSONB array (not 16 columns — explicitly improving on Suggestion 1).

**Design:**

```sql
CREATE TABLE spatial_anchors (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  equipment_id UUID NOT NULL REFERENCES equipment(id) ON DELETE CASCADE,
  anchor_type VARCHAR(50) NOT NULL
    CHECK (anchor_type IN ('qr_code','gs1_digital_link','marker','object_detection','point_cloud','manual')),
  label VARCHAR(255),
  config JSONB NOT NULL DEFAULT '{}',
  -- qr_code/gs1: {value, ecc}; marker: {marker_id, dictionary, size_mm};
  -- object_detection: {model_url, class_label, confidence_threshold};
  -- point_cloud: {url, feature_count}; all: {transform: [16 floats], confidence_threshold}
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_anchors_equipment ON spatial_anchors(equipment_id);
CREATE INDEX idx_anchors_config_gin ON spatial_anchors USING gin (config);
```

- Pydantic discriminated-union validator `AnchorConfig` keyed on `anchor_type`; rejects payloads whose `config` does not match the type's schema (the JSONB validation strategy from Suggestion 3).
- Route `POST /equipment/{id}/anchors` validates `config` against the matching member.
- GS1 Digital Link decode helper maps a scanned URI → `{gtin, serial}` → equipment lookup.

**Testing:**
- `Unit: anchor_type=qr_code with config missing 'value' → 422 naming the field`.
- `Unit: anchor_type=marker with object_detection config fields → 422`.
- `Unit: transform array length != 16 → 422`.
- `Unit: GS1 Digital Link URI parsed to correct GTIN/serial`.

#### 3.3 — Open procedure-content interchange format (`.arwi`)

**What:** A documented, versioned JSON-Schema-backed package format for exporting/importing a full procedure (metadata, steps, anchors-by-reference, media manifest) — the open-standard differentiator.

**Design:**
- `packages/arwi-content-spec/schema.json` (JSON Schema 2020-12). Top level:

```json
{
  "spec_version": "1.0",
  "procedure": { "code": "...", "title": "...", "type": "maintenance", "default_language": "en" },
  "steps": [ { "step_number": 1, "title": "...", "instruction_text": "...",
               "step_type": "action", "media": [{"role":"image","sha256":"...","path":"media/01.png"}],
               "overlays": [{"anchor_ref":"a1","model":"models/bracket.glb","transform":[...]}],
               "translations": {"de": {"title":"...","instruction_text":"..."}} } ],
  "anchors": [ {"ref":"a1","anchor_type":"qr_code","config":{...}} ]
}
```

- Package = zip containing `procedure.json` + `media/` + `models/`. SHA-256 manifest for integrity (and air-gapped transfer).
- Backend `export/package.py` (`build_package(version_id) -> bytes`) and `import_package(bytes, org_id) -> Procedure`.

**Testing:**
- `Unit: a sample procedure exported then re-imported → semantically equal (round-trip)`.
- `Unit: package with media hash mismatch → ImportError naming the file`.
- `Unit: procedure.json validates against schema.json; malformed step_type → ValidationError`.

---

## Phase 4: Procedure Authoring, Versioning, and Approval Workflow

### Purpose
The heart of the authoring side. Implement procedures, their versions with a draft→review→approved→published state machine, ordered steps with typed content, step media (to object storage), step-level 3D overlays bound to anchors, and multi-language translation bundles. After this phase an author can build a complete, media-rich, spatially-anchored procedure and a reviewer can approve and publish it — fulfilling the MVP authoring requirement.

### Tasks

#### 4.1 — Procedures and versions with state machine

**What:** Procedure records and immutable-after-publish versions.

**Design:**

```sql
CREATE TABLE procedures (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  equipment_category_id UUID REFERENCES equipment_categories(id),
  code VARCHAR(50) NOT NULL,
  title VARCHAR(500) NOT NULL,
  description TEXT,
  procedure_type VARCHAR(50) NOT NULL
    CHECK (procedure_type IN ('maintenance','assembly','inspection','safety','calibration','custom')),
  complexity VARCHAR(20) NOT NULL DEFAULT 'standard'
    CHECK (complexity IN ('basic','standard','advanced','expert')),
  min_skill_level_id UUID REFERENCES skill_levels(id),
  default_language VARCHAR(10) NOT NULL DEFAULT 'en',
  metadata JSONB NOT NULL DEFAULT '{}',   -- {source_document_url, standards_tags:[...]}
  is_archived BOOLEAN NOT NULL DEFAULT false,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (organization_id, code)
);
CREATE TABLE procedure_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  procedure_id UUID NOT NULL REFERENCES procedures(id) ON DELETE CASCADE,
  version_number INT NOT NULL,
  status VARCHAR(30) NOT NULL DEFAULT 'draft'
    CHECK (status IN ('draft','in_review','approved','published','superseded','withdrawn')),
  change_summary TEXT,
  authored_by UUID NOT NULL REFERENCES users(id),
  reviewed_by UUID REFERENCES users(id),
  approved_by UUID REFERENCES users(id),
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (procedure_id, version_number)
);
CREATE INDEX idx_proc_versions_status ON procedure_versions(status);
```

- State machine (service-enforced):
  - `draft → in_review` (author submits; requires `procedure.submit`)
  - `in_review → draft` (reviewer rejects)
  - `in_review → approved` (requires `procedure.approve`; **approver ≠ author**)
  - `approved → published` (sets `published_at`; supersedes prior published version of same procedure)
  - `published → superseded` (automatic on newer publish)
  - any → `withdrawn` (admin)
- Editing steps is allowed only while `draft`. Publishing a new version clones the previous version's steps as the editable starting point.
- Endpoints: `POST /procedures`, `POST /procedures/{id}/versions`, `POST /versions/{id}/transition` (body `{action}`).

**Testing:**
- `Unit: author who created version attempts approve → 403 (separation of duties)`.
- `Unit: edit step on published version → 409 (immutable)`.
- `Unit: illegal transition draft→published directly → 422`.
- `Integration: publish v2 → v1 auto-set superseded; latest published resolves to v2`.

#### 4.2 — Steps, typed content, and translations

**What:** Ordered steps with type, criticality, adaptive skip conditions, and a JSONB translation bundle.

**Design:**

```sql
CREATE TABLE procedure_steps (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  procedure_version_id UUID NOT NULL REFERENCES procedure_versions(id) ON DELETE CASCADE,
  step_number INT NOT NULL,
  title VARCHAR(500) NOT NULL,
  instruction_text TEXT NOT NULL,
  step_type VARCHAR(50) NOT NULL DEFAULT 'action'
    CHECK (step_type IN ('action','inspection','decision','warning','info','ai_check')),
  is_critical BOOLEAN NOT NULL DEFAULT false,
  can_skip BOOLEAN NOT NULL DEFAULT false,
  config JSONB NOT NULL DEFAULT '{}',  -- {estimated_duration_s, skip_condition, tools:[...], safety_notes}
  translations JSONB NOT NULL DEFAULT '{}',  -- {"de":{"title","instruction_text","safety_notes","source":"ai|manual","verified":bool}}
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (procedure_version_id, step_number)
);
CREATE INDEX idx_steps_version ON procedure_steps(procedure_version_id);
```

- Reorder endpoint `PATCH /versions/{id}/steps/reorder` (body: ordered list of step ids) renumbers atomically.
- `skip_condition` is a small safe expression (e.g. `operator.skill_rank >= 3`) evaluated by a sandboxed evaluator at runtime (Phase 6 adaptive).

**Testing:**
- `Unit: insert step with duplicate step_number in same version → 409`.
- `Unit: reorder with a step id not in version → 422`.
- `Unit: translations bundle with malformed language key → 422`.

#### 4.3 — Step media (object storage) and presigned upload

**What:** Attach images/video/audio/3D/pdf to steps; store binaries in S3/MinIO.

**Design:**

```sql
CREATE TABLE step_media (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  step_id UUID NOT NULL REFERENCES procedure_steps(id) ON DELETE CASCADE,
  media_type VARCHAR(30) NOT NULL
    CHECK (media_type IN ('image','video','audio','3d_model','animation','pdf')),
  object_key VARCHAR(1024) NOT NULL,
  meta JSONB NOT NULL DEFAULT '{}',  -- {thumbnail_key,size_bytes,mime,duration_s,caption,display_order}
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- `POST /steps/{id}/media/presign` → returns a presigned PUT URL + object_key; client uploads directly to MinIO/S3, then `POST /steps/{id}/media` registers the record. Server validates mime against `media_type`.
- glTF uploads validated as glTF 2.0 (magic bytes / JSON chunk); Draco compression recommended in response warnings if uncompressed > threshold.

**Testing:**
- `Unit: presign returns URL scoped to the org's bucket prefix`.
- `Integration (MinIO testcontainer): presign → PUT bytes → register → GET object exists`.
- `Unit: register video object with image mime → 422`.

#### 4.4 — Step 3D overlays bound to anchors

**What:** Place glTF overlays relative to a spatial anchor with transform/opacity/animation.

**Design:**

```sql
CREATE TABLE step_3d_overlays (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  step_id UUID NOT NULL REFERENCES procedure_steps(id) ON DELETE CASCADE,
  spatial_anchor_id UUID NOT NULL REFERENCES spatial_anchors(id),
  model_object_key VARCHAR(1024) NOT NULL,
  placement JSONB NOT NULL DEFAULT '{}',
  -- {format:'glb', compression:'draco', offset:[x,y,z], rotation:[x,y,z,w], scale:[x,y,z], opacity, animation_name}
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_overlays_step ON step_3d_overlays(step_id);
```

- Pydantic `Placement` validator: rotation quaternion length 4, scale all > 0, opacity 0–1.

**Testing:**
- `Unit: placement rotation with 3 elements → 422`.
- `Unit: overlay referencing anchor on different equipment than the procedure's category → warning surfaced (non-blocking)`.

---

## Phase 5: AR/WebAR Player, Offline Cache, and Execution Tracking

### Purpose
Deliver the operator experience and the compliance backbone. Build the WebAR/WebXR runtime player that resolves a scanned anchor to the right published procedure, renders step cards and 3D overlays spatially, and works offline with sync-on-reconnect. Record per-step timestamped execution data (operator, pass/fail, skips) — the audit trail required for ISO 9001 / GMP. After this phase the MVP is functionally complete: author → publish → scan → execute → audit.

### Tasks

#### 5.1 — Execution and step-execution records (partitioned)

**What:** Persist runs and per-step outcomes; time-partition for scale.

**Design:**

```sql
CREATE TABLE procedure_executions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  procedure_version_id UUID NOT NULL REFERENCES procedure_versions(id),
  equipment_id UUID NOT NULL REFERENCES equipment(id),
  operator_id UUID NOT NULL REFERENCES users(id),
  status VARCHAR(30) NOT NULL DEFAULT 'in_progress'
    CHECK (status IN ('in_progress','completed','paused','aborted','failed')),
  started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at TIMESTAMPTZ,
  total_duration_seconds INT,
  device JSONB NOT NULL DEFAULT '{}',  -- {type:'mobile_ios|hololens2|web', device_id, ua}
  is_offline BOOLEAN NOT NULL DEFAULT false,
  synced_at TIMESTAMPTZ,
  client_execution_id UUID,            -- idempotency key from offline client
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (started_at);
-- monthly partitions created by a maintenance task
CREATE TABLE step_executions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  execution_id UUID NOT NULL,
  step_id UUID NOT NULL REFERENCES procedure_steps(id),
  step_number INT NOT NULL,
  status VARCHAR(20) NOT NULL
    CHECK (status IN ('pending','in_progress','passed','failed','skipped','retried')),
  started_at TIMESTAMPTZ, completed_at TIMESTAMPTZ, duration_seconds INT,
  skip_reason TEXT, operator_notes TEXT,
  ai_result JSONB,  -- {result:'pass|fail|inconclusive', confidence, capture_key}
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_step_exec_execution ON step_executions(execution_id);
```

- `client_execution_id` makes offline replay idempotent (Suggestion 3 offline-sync note).

**Testing:**
- `Integration: monthly partition created; insert lands in correct partition`.
- `Unit: complete execution → total_duration_seconds computed from started/completed`.

#### 5.2 — Player content API and anchor resolution

**What:** Endpoints the player calls to fetch a fully-resolved, language-localised published procedure and to resolve a scanned code to a procedure.

**Design:**
- `GET /play/resolve?code=<qr|gs1>&equipment_hint=` → `{equipment_id, procedure_id, version_id}` (matches anchor `config.value` / GS1 GTIN, picks latest published version for the equipment's category).
- `GET /play/versions/{id}?lang=de` → denormalised JSON: steps with localised text (falls back to default_language if no translation), media presigned GET URLs, overlays with placement, anchors. This is the single payload the offline cache stores.
- `POST /play/executions` (idempotent on `client_execution_id`), `POST /play/executions/{id}/steps`, `POST /play/executions/{id}/complete`.

**Testing:**
- `Unit: resolve unknown code → 404`.
- `Unit: lang=de missing translation for step → returns default_language text with flag fallback=true`.
- `Integration: only published versions are resolvable; draft/approved → not returned`.
- `Integration: re-POST execution with same client_execution_id → same execution, not a duplicate`.

#### 5.3 — WebXR / WebAR runtime player (frontend)

**What:** Browser player using WebXR `immersive-ar` (W3C AR Module L1) with QR/marker scanning, glTF overlay rendering, hit-test placement, and step-card UI.

**Design:**
- `web/src/ar/session.ts`: requests `navigator.xr.requestSession('immersive-ar', {requiredFeatures:['hit-test'], optionalFeatures:['dom-overlay','light-estimation']})`; graceful fallback to camera-passthrough 2D card mode where WebXR AR is unsupported (note in standards.md: iOS Safari lacks `immersive-ar` outside visionOS).
- QR/GS1 scan via `zxing-wasm`; on detect → call `/play/resolve` → load cached or fetched procedure.
- three.js `GLTFLoader` + `DRACOLoader`; overlay placed at anchor transform; instruction cards rendered as DOM overlay tethered to anchor.
- Progressive gating: cannot advance past a `is_critical` or `ai_check` step until its condition/result is satisfied.

**Testing:**
- `E2E (Playwright, WebXR mocked): scan fixture QR → procedure loads → advance through steps → completion posted`.
- `E2E: device without immersive-ar → falls back to 2D card mode, still records execution`.
- `Unit: critical step gates next button until marked passed`.

#### 5.4 — Offline-first cache and sync

**What:** Service worker + IndexedDB cache of resolved procedures and queued execution records; sync on reconnect.

**Design:**
- Service worker caches the player app shell and glTF/media for downloaded procedures.
- `web/src/offline/store.ts`: IndexedDB stores `procedures`, `pendingExecutions`, `pendingSteps`. Each pending record carries `client_execution_id`.
- On `online` event, flush queue to `/play/...` endpoints; server idempotency dedupes. Conflict policy: last-write-wins on step records keyed by `(execution_id, step_number)`.

**Testing:**
- `E2E: download procedure, go offline (network throttled to offline), execute fully, go online → records appear server-side exactly once`.
- `Unit: queue flush with one failed POST retries that record, does not re-send succeeded ones`.

---

## Phase 6: AI-Native Capabilities

### Purpose
Deliver the unifying differentiator. Add the provider-abstracted AI layer and four capabilities the research calls out as fragmented across incumbents: SOP-to-procedure conversion, runtime multi-language translation, computer-vision step inspection, and adaptive step ordering. All run against either a cloud LLM/CV endpoint or a fully local one for air-gapped sites.

### Tasks

#### 6.1 — Provider-abstracted AI client

**What:** One interface over OpenAI-compatible, Anthropic, and local (Ollama/vLLM) backends for chat, vision, and embeddings.

**Design:**

```python
class AIClient(Protocol):
    async def complete(self, system: str, user: str, *, json_schema: dict | None = None) -> dict | str: ...
    async def vision(self, prompt: str, image_bytes: bytes, *, json_schema: dict) -> dict: ...
    async def embed(self, texts: list[str]) -> list[list[float]]: ...

def get_ai_client(settings: Settings) -> AIClient: ...   # factory keyed on ai_provider
```

- Structured outputs requested via JSON schema; client validates and retries once on schema failure.

**Testing:**
- `Unit (mocked transport): complete returns schema-valid JSON → parsed; invalid then valid on retry → succeeds`.
- `Unit: factory returns local client when ai_provider='ollama'`.

#### 6.2 — SOP-to-procedure conversion (Celery)

**What:** Upload a PDF/Word SOP; AI produces a structured draft procedure (steps, types, safety notes).

**Design:**
- `POST /ai/sop-convert` (multipart) → enqueues Celery task → returns `{job_id}`. `GET /ai/jobs/{id}` polls.
- Pipeline: extract text (pdfminer/python-docx) → chunk → LLM with system prompt:
  > "You convert an industrial SOP into a structured procedure. Output JSON matching the provided schema. Each discrete physical action becomes one step. Classify step_type. Extract safety_notes and required tools verbatim. Do not invent steps."
- Output validated against the step schema; written as a new `draft` version. Original file stored; `metadata.source_document_url` set.

**Testing:**
- `Integration (mocked AI): sample 2-page PDF → draft procedure with ≥1 step, types classified, source recorded`.
- `Unit: AI returns extra invented step not grounded in text → flagged in job warnings (heuristic: step with no source span)`.
- `Unit: corrupt upload → job fails with clear error, no draft created`.

#### 6.3 — Runtime translation

**What:** Translate step content into a target language on demand, caching into the step `translations` JSONB.

**Design:**
- `POST /versions/{id}/translate` (body `{target_languages:[...]}`) → Celery task translates each step, preserving domain terminology (glossary passed in prompt), marks `source:'ai', verified:false`.
- Player requests `?lang=` (Phase 5.2) and gets cached translations; unverified translations rendered with an advisory badge.

**Testing:**
- `Integration (mocked AI): translate to de+fr → both bundles present, verified=false`.
- `Unit: re-translate verified language → skipped unless force=true`.

#### 6.4 — Computer-vision step inspection

**What:** For `ai_check` steps, capture an image at runtime and return pass/fail with confidence.

**Design:**

```sql
CREATE TABLE ai_inspection_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  step_id UUID NOT NULL REFERENCES procedure_steps(id) ON DELETE CASCADE,
  config JSONB NOT NULL DEFAULT '{}',
  -- {model:'alignment_verify', pass_threshold:0.9, capture_type:'photo',
  --  reference_image_key, prompt, max_retries:3}
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- `POST /play/executions/{id}/steps/{step_id}/inspect` (image) → `AIClient.vision` with structured schema `{result, confidence, reasons[]}` → compares to `pass_threshold`. Capture image stored to S3, key recorded in `step_executions.ai_result.capture_key` for audit.
- Below threshold and retries remaining → `inconclusive`, operator re-captures; else `failed` blocks a critical step.

**Testing:**
- `Integration (mocked vision): confidence 0.95 ≥ 0.9 → pass; capture stored`.
- `Unit: confidence 0.6 with retries left → inconclusive, retry_count incremented`.
- `Unit: failed inspection on critical step → next step gated`.

#### 6.5 — Adaptive step ordering/skipping

**What:** Evaluate `can_skip`/`skip_condition` against operator skill and context at runtime.

**Design:**
- Safe expression evaluator (allowlisted names: `operator.skill_rank`, `equipment.category`, `context.shift`). Returned in the player payload as a resolved `should_skip` per step so offline players can apply it deterministically.

**Testing:**
- `Unit: skip_condition 'operator.skill_rank >= 3' with rank 4 → should_skip true`.
- `Unit: expression referencing disallowed name → rejected at author save time (422)`.

---

## Phase 7: Enterprise Integration — Webhooks, MES/ERP, Versioned Audit Export

### Purpose
Connect the platform to the systems that drive real plant work. Inbound webhooks from MES/ERP (SAP, Oracle) auto-assign procedures to operators/workstations; outbound webhooks notify external systems of executions and publications; signed audit exports satisfy compliance. After this phase the platform participates in the enterprise workflow rather than living in isolation.

### Tasks

#### 7.1 — Outbound webhooks with signed, retried delivery

**What:** Configurable webhook subscriptions delivered reliably via Celery.

**Design:**

```sql
CREATE TABLE webhook_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  target_url VARCHAR(1024) NOT NULL,
  event_types JSONB NOT NULL DEFAULT '[]',  -- ['execution.completed','procedure.published']
  secret_hash VARCHAR(255),
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE webhook_deliveries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  webhook_config_id UUID NOT NULL REFERENCES webhook_configs(id) ON DELETE CASCADE,
  event_type VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending','delivered','failed')),
  attempts INT NOT NULL DEFAULT 0, last_error TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- Domain events (execution.completed, procedure.published, anomaly.raised) emitted via an in-process event bus → enqueue delivery tasks. HMAC-SHA256 signature header using the per-config secret. Exponential backoff, max 6 attempts.

**Testing:**
- `Integration (mocked HTTP): execution.completed → delivery POSTed with valid HMAC header`.
- `Unit: receiver returns 500 → retried with backoff; after max attempts → status failed`.

#### 7.2 — Inbound work-order webhooks and assignment

**What:** Receive MES/ERP work orders and auto-assign the right procedure to the right operator/workstation.

**Design:**

```sql
CREATE TABLE work_order_links (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  execution_id UUID REFERENCES procedure_executions(id),
  external_system VARCHAR(50) NOT NULL,   -- sap|oracle|custom
  external_id VARCHAR(255) NOT NULL,
  external_url VARCHAR(1024),
  payload JSONB NOT NULL DEFAULT '{}',
  linked_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE procedure_assignments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  procedure_id UUID NOT NULL REFERENCES procedures(id),
  equipment_id UUID NOT NULL REFERENCES equipment(id),
  operator_id UUID REFERENCES users(id),
  work_order_link_id UUID REFERENCES work_order_links(id),
  due_at TIMESTAMPTZ, status VARCHAR(20) NOT NULL DEFAULT 'open',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- `POST /integrations/work-orders` (HMAC-verified) → maps `{equipment GTIN/serial, operation code, operator}` → resolves equipment + procedure → creates assignment. Adapter layer normalises SAP/Oracle payload shapes (mapping config in `webhook_configs.settings`).

**Testing:**
- `Integration (mocked): SAP-shaped payload → assignment created for matched equipment + procedure`.
- `Unit: invalid HMAC → 401, no assignment`.
- `Unit: GTIN with no matching equipment → 422 with diagnostic`.

#### 7.3 — Compliance audit export

**What:** Signed, immutable export of execution history for an equipment/procedure/date range.

**Design:**
- `GET /audit/export?equipment_id=&from=&to=&format=json|csv` → streams every execution + step record + AI captures manifest, with a detached signature (Ed25519) over the payload hash. Includes procedure version, approver identity, and per-step timestamps to evidence ISO 9001 / GMP control.

**Testing:**
- `Integration: export for a date range → record count matches DB; signature verifies`.
- `Unit: tampering with one byte → signature verification fails`.

---

## Phase 8: Remote Expert Assistance (WebRTC)

### Purpose
Add live remote expert help with AR annotation — the feature incumbents charge separately for (Vuforia Chalk). An off-site expert joins the operator's session, sees the camera feed, and draws annotations spatially anchored in the operator's view, over WebRTC (sub-300 ms).

### Tasks

#### 8.1 — Signalling and session model

**What:** WebSocket signalling over Redis pub/sub; session lifecycle records.

**Design:**

```sql
CREATE TABLE remote_expert_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  execution_id UUID REFERENCES procedure_executions(id),
  operator_id UUID NOT NULL REFERENCES users(id),
  expert_id UUID NOT NULL REFERENCES users(id),
  status VARCHAR(30) NOT NULL DEFAULT 'requested'
    CHECK (status IN ('requested','active','ended','missed')),
  started_at TIMESTAMPTZ, ended_at TIMESTAMPTZ,
  recording_key VARCHAR(1024),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE expert_annotations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id UUID NOT NULL REFERENCES remote_expert_sessions(id) ON DELETE CASCADE,
  annotation_type VARCHAR(30) NOT NULL
    CHECK (annotation_type IN ('arrow','circle','text','freehand','3d_pointer')),
  spatial JSONB NOT NULL DEFAULT '{}',  -- {anchor_id, position:[x,y,z], content}
  timestamp_ms BIGINT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- WS `/ws/expert/{session_id}` relays SDP offers/answers and ICE candidates via Redis pub/sub (multi-worker safe). Annotations broadcast over the same channel and persisted.
- STUN/TURN config via env (`ARWI_TURN_URL`); air-gapped deployments run coturn in-stack.

**Testing:**
- `Integration: two WS clients exchange offer/answer/ICE through the relay`.
- `Unit: request session → operator notified; expert joins → status active; both leave → ended`.

#### 8.2 — Expert UI and annotation overlay

**What:** Expert-side React view of the operator feed with annotation tools; operator-side overlay rendering.

**Design:**
- Expert draws → annotation message `{type, spatial}` → operator player renders it tethered to the active anchor in the AR scene; persisted for audit.

**Testing:**
- `E2E (mocked media): expert places arrow → operator client receives and renders within the AR scene; annotation row persisted`.

---

## Phase 9: Analytics, Anomaly Detection, and Dashboards

### Purpose
Turn the execution audit trail into operational insight. Provide completion analytics (time-per-step, skip rates, pass/fail trends) and AI-driven anomaly detection that surfaces process drift before it becomes a defect — an underserved area the research highlights.

### Tasks

#### 9.1 — Analytics aggregation and dashboard API

**What:** Materialised aggregates over executions for engineer dashboards.

**Design:**
- Nightly Celery job populates `step_stats_daily` (avg/median duration, fail rate, skip rate per step). API: `GET /analytics/procedures/{id}` → trends; `GET /analytics/operators/{id}` → throughput. Queries target a read replica in production.

**Testing:**
- `Integration: seed executions → aggregation produces correct median duration and fail rate per step`.

#### 9.2 — Anomaly detection

**What:** Detect step-failure spikes, duration drift, and abnormal skip patterns.

**Design:**

```sql
CREATE TABLE anomaly_alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  alert_type VARCHAR(50) NOT NULL,  -- step_failure_spike|duration_drift|skip_pattern
  severity VARCHAR(20) NOT NULL CHECK (severity IN ('info','warning','critical')),
  procedure_id UUID REFERENCES procedures(id),
  step_id UUID REFERENCES procedure_steps(id),
  description TEXT NOT NULL,
  detection JSONB NOT NULL DEFAULT '{}',  -- {baseline, observed, z_score, window}
  acknowledged_by UUID REFERENCES users(id), acknowledged_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- Detector compares rolling window stats to a trailing baseline; z-score over threshold → alert; emits `anomaly.raised` event (→ webhooks Phase 7). `POST /analytics/anomalies/{id}/ack`.

**Testing:**
- `Unit: inject a fail-rate spike (baseline 2%, observed 30%) → critical alert raised`.
- `Unit: stable data within baseline → no alert`.
- `Integration: anomaly.raised triggers a webhook delivery`.

---

## Phase 10: Hardening, Air-Gapped Deployment, and SDK/Docs

### Purpose
Make it deployable and trustworthy in regulated/defence contexts. Add WebAuthn sign-off, OWASP hardening, the production Helm chart, an air-gapped image bundle, and generated client SDK + docs from the OpenAPI spec.

### Tasks

#### 10.1 — WebAuthn step sign-off (AAL2/AAL3)

**What:** Phishing-resistant MFA for critical-step or version-approval sign-off (NIST SP 800-63B).

**Design:**
- WebAuthn registration/assertion endpoints; a `signoff_events` table records `{user_id, target_type, target_id, credential_id, signed_at}`. Critical step completion / version approval can require an assertion; the resulting `aal` is recorded on the execution/version.

**Testing:**
- `Integration (mocked authenticator): approval requiring AAL2 without assertion → 403; with valid assertion → succeeds and recorded`.

#### 10.2 — Security hardening (OWASP)

**What:** Address OWASP Top 10 (web) and IoT Top 10 (devices) risks.

**Design:**
- Rate limiting (Redis), strict CORS, secure headers, input validation already via Pydantic, signed/expiring presigned URLs, audit logging of auth events, dependency scanning in CI. Device/API auth uses bearer tokens with short TTL; no non-expiring tokens.

**Testing:**
- `Integration: anonymous access to any /procedures route → 401`.
- `Integration: SQLi-style payloads in query params → rejected/parameterised, no error leakage`.
- `Security: CI runs dependency + container scan; build fails on high severity`.

#### 10.3 — Helm chart, air-gapped bundle, and SDK/docs

**What:** Production deploy artefacts and developer ergonomics.

**Design:**
- `helm/` chart for api/worker/web/postgres/redis/minio/coturn with values for external managed stores. `scripts/airgap-bundle.sh` saves all images + local AI model weights to a tarball with an offline install script. `scripts/gen-sdk.sh` runs `openapi-generator` against `docs/openapi.json` to emit a TypeScript client (used by `web/src/lib`) and a Python client. Docs site from the OpenAPI spec (Redoc/Scalar).

**Testing:**
- `Integration: helm template renders valid manifests; chart lint passes`.
- `E2E (CI): air-gapped bundle installed in a network-isolated container → /health ok, can author and play a procedure with local AI`.
- `Unit: generated TS client compiles against current openapi.json`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (skeleton, DB core)         ─── required by everything
    │
Phase 2: Identity, RBAC, Skills                 ─── requires 1
    │
Phase 3: Equipment, Anchors, Content Format     ─── requires 1,2
    │
Phase 4: Authoring, Versioning, Approval        ─── requires 2,3   ◀── MVP authoring
    │
Phase 5: AR Player, Offline, Execution          ─── requires 4     ◀── MVP complete
    ├── Phase 6: AI-Native Capabilities          ─── requires 4,5
    ├── Phase 7: Enterprise Integration          ─── requires 4,5 (parallel with 6)
    ├── Phase 8: Remote Expert (WebRTC)          ─── requires 5     (parallel with 6,7)
    └── Phase 9: Analytics & Anomaly Detection   ─── requires 5 (anomaly→7 webhooks)
         │
Phase 10: Hardening, Air-Gap, SDK/Docs          ─── requires all prior
```

**Parallelism opportunities:**
- After Phase 5, Phases 6, 7, 8, and 9 are largely independent and can be built concurrently by separate streams. Phase 9's `anomaly.raised` event consumes Phase 7's webhook delivery, so wire that integration after both land.
- Frontend authoring (Phase 4 UI) and the player (Phase 5 UI) share the generated API client; keep `docs/openapi.json` regenerated whenever backend routes change.

**Estimated scope:** Large (10 phases, ~38 tasks; full-stack platform with AR runtime, AI, real-time, and enterprise integration).

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests for the phase pass (`pytest`, `vitest`); E2E tests pass where specified (`playwright`).
3. Backend linting/formatting passes (`ruff check`, `ruff format --check`) and type checking passes (`mypy`).
4. Frontend linting/type checking passes (`eslint`, `tsc --noEmit`) where the phase touches `web/`.
5. `docker compose up` builds and starts the full stack; `GET /health` returns ok.
6. Alembic migrations created for all schema changes; `alembic upgrade head` then `downgrade base` round-trips cleanly on an empty DB.
7. New/changed API routes appear in the regenerated `docs/openapi.json`, which validates against the OpenAPI 3.1 meta-schema.
8. New config options documented in `README.md`/`.env.example` with defaults.
9. The phase's headline capability works end-to-end against the running stack (manual smoke or E2E).
10. New JSONB columns have a corresponding Pydantic validator enforcing their shape (per the Suggestion-3 validation strategy).
11. Relevant standards honoured and referenced in code/docs where applicable (glTF 2.0, WebXR AR Module L1, WebRTC, OAuth 2.0/OIDC, OpenAPI 3.1, GS1 Digital Link, NIST SP 800-63B).
```
