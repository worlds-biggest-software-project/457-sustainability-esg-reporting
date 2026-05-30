# Sustainability & ESG Reporting — Phased Development Plan

> Project: 457-sustainability-esg-reporting · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a single buildable specification for an open-source, AI-native carbon accounting and multi-framework ESG disclosure platform targeting the underserved mid-market.

---

## Core Requirements (Synthesis)

**What it does.** Centralises GHG accounting (Scope 1/2/3 per the GHG Protocol), maps a single emissions/ESG dataset to multiple disclosure frameworks (GRI, ISSB S1/S2, CSRD/ESRS, CDP, SB 253, SEC climate), collects Scope 3 supplier data through a self-service portal, and produces audit-ready disclosures with full source-to-disclosure data lineage. AI augments document ingestion, emission-factor matching, anomaly detection, and narrative drafting.

**Primary users.** Sustainability managers (own the inventory), finance/compliance teams (own assurance and filing), procurement teams (own Scope 3 supplier engagement), and external auditors (read-only, lineage traversal). Suppliers are an external persona who submit data through the portal with no login to the core platform.

**Key differentiators.** (1) Open-source and self-hostable, breaking the $50k–$200k/yr enterprise lock-in; (2) transparent, versioned emission-factor library built on public sources (IPCC EFDB, US EPA, GHG Protocol) rather than a proprietary black box; (3) single-dataset multi-framework mapping with explicit crosswalks; (4) AI-native ingestion and factor matching; (5) restatement-aware recalculation when factors change.

**Deployment model.** Self-hosted via Docker Compose / Kubernetes; cloud-hosted and hybrid also supported. Single PostgreSQL database for the MVP (no polyglot persistence) to keep operations tractable for mid-market teams.

**Integration surface.** CSV import, ERP connectors (SAP, NetSuite — stubbed in MVP, one real connector in later phase), utility/energy APIs, LLM providers (OpenAI-compatible + local via Ollama), object storage (S3-compatible) for documents, SMTP for supplier reminders, and outbound CDP Disclosure API (later phase, ASP-gated).

**Standards the software must implement.** GHG Protocol Corporate Standard (calculation methodology), ISO 14064-1 alignment (assurance trail), IPCC AR5/AR6 GWP values, GRI / ESRS / IFRS S2 / CDP / SB 253 disclosure mappings, ESRS & GRI XBRL taxonomies (export), Inline XBRL / ESEF (CSRD filing), PACT Pathfinder schema (PCF exchange — backlog), Open Footprint Forum data model (interoperability export), OAuth 2.0 / OIDC (auth), GDPR (supplier/workforce PII), SOC 2 controls (audit logging, RBAC).

**Data model decision.** Adopt **Data Model 3 (Hybrid Relational + JSONB on PostgreSQL)** as the canonical schema. Rationale: the GHG calculation chain (source → activity → emission → factor) stays fully normalised for audit-grade traceability and fast aggregation, while framework-specific disclosures, polymorphic Scope 3 activity details, and diverse S/G metrics live in validated JSONB — letting new frameworks ship without schema migrations, which is the platform's core competitive edge. We borrow the **restatement-as-new-record** idea from Model 2 for emission-factor recalculation (without committing to full event sourcing), and reserve **TimescaleDB** (Model 4) for an optional real-time metering phase. Model 4's tri-database design is explicitly rejected for the MVP as operationally unsuitable for mid-market self-hosting.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | The product is LLM- and data-pipeline-heavy (OCR, factor matching, anomaly detection, narrative drafting). Python has the strongest ecosystem for these (`pydantic`, `pandas`, OpenAI SDK, `arelle` for XBRL) and is approachable for the sustainability-analyst contributors this open-source project wants to attract. |
| API framework | FastAPI | Async, first-class Pydantic v2 integration, and auto-generates the OpenAPI 3.1 spec that auditors and ERP integrators expect (per standards.md, all incumbents expose OpenAPI). |
| Data validation | Pydantic v2 | Single source of truth for request/response models *and* for validating JSONB payloads against framework schemas — directly addresses Model 3's biggest con (JSONB type safety). |
| Database | PostgreSQL 16 (+ `pg_trgm`, `btree_gin`, `pgcrypto`) | Hybrid relational+JSONB store per Data Model 3. `pg_trgm` powers emission-factor fuzzy name matching; GIN indexes make JSONB queryable; RLS enforces tenant isolation. One database = tractable self-hosting. |
| ORM / migrations | SQLAlchemy 2.0 + Alembic | Mature JSONB support, async engine, and versioned migrations (replaces Flyway suggestion to stay in-language). |
| Task queue | Celery + Redis | Async workloads: OCR/LLM document processing, bulk recalculation on factor updates, supplier reminder scheduling, XBRL generation. Redis doubles as cache for framework schemas/crosswalks. |
| Object storage | S3-compatible (MinIO for self-host, AWS S3 for cloud) via `boto3` | Source documents (utility bills, supplier evidence) must be stored durably with SHA-256 checksums for the audit trail. |
| LLM access | Provider-abstracted gateway (OpenAI-compatible API; Ollama for local/offline) | Self-hosted deployments must not be forced onto a paid API. All LLM calls go through one internal interface with a `null`/deterministic fallback for tests. |
| OCR | `pytesseract` (Tesseract) + LLM post-extraction | Open-source OCR baseline; LLM structures the OCR text into typed fields. No proprietary OCR dependency. |
| Frontend | Next.js 14 (App Router) + TypeScript + shadcn/ui + Tailwind + Recharts | Dashboard-centric SPA (the dominant UX pattern across Sweep/Watershed/Persefoni). The public supplier portal is a separate, login-less route group in the same app. |
| XBRL | `arelle` (Python) | Open-source XBRL processor for generating ESRS/GRI Inline XBRL filings (ESEF). |
| Auth | OAuth 2.0 / OIDC via Authlib; JWT sessions | Standards.md mandates OAuth 2.0/OIDC for SaaS and integrations; supports enterprise SSO. |
| Containerisation | Docker + Docker Compose (dev/self-host); Helm chart (k8s) | README commits to Docker/Kubernetes self-hosting. |
| Testing | pytest, pytest-asyncio, testcontainers, Playwright | Unit + integration (real Postgres via testcontainers) + E2E (Playwright for portal/dashboard flows). |
| Code quality | Ruff (lint+format), mypy (strict), pre-commit | Fast, single-tool lint/format; strict typing catches JSONB-shape drift early. |
| Package manager | uv (Python), pnpm (frontend) | Fast, reproducible installs. |
| Observability | OpenTelemetry + Prometheus/Grafana; structured JSON logs | SOC 2 expectations; trace ingestion → calculation → disclosure pipeline. |

### Project Structure

```
sustainability-esg-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml              # postgres, redis, minio, api, worker, web
├── docker-compose.prod.yml
├── alembic.ini
├── .env.example
├── README.md
├── deploy/
│   └── helm/                       # Kubernetes chart
├── data/
│   ├── emission_factors/           # seed CSVs: IPCC EFDB, US EPA, GHG Protocol
│   ├── frameworks/                 # framework disclosure schemas (JSON)
│   │   ├── gri.json
│   │   ├── esrs.json
│   │   ├── issb_s2.json
│   │   ├── cdp.json
│   │   └── sb253.json
│   └── crosswalks/
│       └── crosswalk-v1.json       # cross-framework equivalence map
├── src/
│   └── esg/
│       ├── __init__.py
│       ├── main.py                 # FastAPI app factory
│       ├── config.py               # Pydantic Settings
│       ├── db/
│       │   ├── session.py          # async engine, session, RLS context
│       │   ├── base.py             # declarative base, mixins (TenantMixin, TimestampMixin)
│       │   └── models/             # SQLAlchemy ORM (one file per aggregate area)
│       │       ├── org.py          # tenants, org_units, facilities
│       │       ├── factors.py      # emission_factor_libraries, emission_factors
│       │       ├── activity.py     # emission_sources, activity_data, emissions
│       │       ├── reporting.py    # reporting_periods, frameworks, disclosure_responses
│       │       ├── esg_metrics.py  # esg_metrics (S & G)
│       │       ├── supplier.py     # suppliers, requests, responses
│       │       ├── targets.py      # reduction_targets, target_progress
│       │       ├── documents.py    # source_documents
│       │       ├── governance.py   # users, approval_workflows, approval_steps
│       │       └── audit.py        # audit_log
│       ├── schemas/                # Pydantic request/response + JSONB shape schemas
│       │   ├── activity.py
│       │   ├── disclosure.py
│       │   ├── framework_schema.py # validators for JSONB disclosure payloads
│       │   └── ...
│       ├── api/
│       │   ├── deps.py             # auth, tenant context, pagination
│       │   ├── routers/
│       │   │   ├── org.py
│       │   │   ├── factors.py
│       │   │   ├── activity.py
│       │   │   ├── emissions.py
│       │   │   ├── reporting.py
│       │   │   ├── suppliers.py
│       │   │   ├── targets.py
│       │   │   ├── documents.py
│       │   │   ├── materiality.py
│       │   │   ├── copilot.py
│       │   │   └── portal.py       # login-less supplier portal endpoints
│       │   └── errors.py
│       ├── core/                   # domain logic (no FastAPI/SQLAlchemy imports)
│       │   ├── ghg/
│       │   │   ├── engine.py       # calculation engine
│       │   │   ├── gwp.py          # AR5/AR6 GWP tables
│       │   │   ├── units.py        # unit conversion
│       │   │   ├── scope2.py       # location- vs market-based
│       │   │   └── scope3/         # per-category calculators
│       │   ├── factors/
│       │   │   └── matcher.py      # NLP/embedding factor matching
│       │   ├── frameworks/
│       │   │   ├── mapper.py       # crosswalk → disclosure population
│       │   │   └── loader.py       # load framework JSON schemas
│       │   ├── export/
│       │   │   ├── xbrl.py
│       │   │   ├── pdf.py
│       │   │   └── open_footprint.py
│       │   ├── anomaly.py
│       │   └── materiality.py
│       ├── integrations/
│       │   ├── llm/                # gateway, providers, prompts
│       │   ├── ocr/
│       │   ├── storage/
│       │   ├── erp/                # SAP, NetSuite connectors
│       │   └── cdp/
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks/              # ingestion, recalculation, reminders, exports
│       ├── audit/
│       │   └── middleware.py       # writes audit_log + lineage on mutations
│       └── seed/
│           └── load_factors.py     # CLI to load factor libraries & framework schemas
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/                   # sample bills, supplier responses, golden XBRL
└── web/                            # Next.js app (dashboard + supplier portal)
    ├── package.json
    ├── app/
    ├── components/
    └── lib/api-client.ts           # generated from OpenAPI spec
```

---

## Phase 1: Foundation, Data Model & Tenancy

### Purpose
Establish the project skeleton, the canonical PostgreSQL schema (Data Model 3 core tables), database migrations, multi-tenant isolation, configuration, and the audit-log substrate that every later phase depends on. After this phase the application boots, connects to Postgres/Redis/MinIO, runs migrations, and enforces tenant scoping — but exposes no business endpoints yet.

### Tasks

#### 1.1 — Project scaffold, config, and containerised dev environment

**What**: A runnable FastAPI app with `uv`-managed deps, settings, and a `docker-compose.yml` bringing up Postgres 16, Redis, MinIO, the API, and a Celery worker.

**Design**:
- `config.py` — Pydantic `BaseSettings`:
  ```python
  class Settings(BaseSettings):
      database_url: PostgresDsn
      redis_url: RedisDsn
      s3_endpoint: str; s3_bucket: str = "esg-documents"
      s3_access_key: str; s3_secret_key: SecretStr
      llm_provider: Literal["openai", "ollama", "null"] = "null"
      llm_base_url: str | None = None
      llm_api_key: SecretStr | None = None
      jwt_secret: SecretStr
      gwp_default: Literal["AR5", "AR6"] = "AR6"
      environment: Literal["dev", "test", "prod"] = "dev"
      model_config = SettingsConfigDict(env_prefix="ESG_", env_file=".env")
  ```
- `main.py` — `create_app()` factory mounting routers, OTel middleware, exception handlers; `/healthz` returns `{"status":"ok","db":bool,"redis":bool}`.
- `docker-compose.yml` services: `postgres` (with `pg_trgm`, `btree_gin`, `pgcrypto` enabled via init SQL), `redis`, `minio`, `api`, `worker`.

**Testing**:
- `Unit: Settings loads from env with ESG_ prefix; missing required field → ValidationError naming the field.`
- `Integration: docker-compose up → GET /healthz returns 200 with db=true, redis=true.`
- `Unit: create_app() registers exception handlers and /healthz route.`

#### 1.2 — Canonical schema & Alembic migrations (Data Model 3 core)

**What**: ORM models and the initial migration for the normalised core plus JSONB columns from Data Model 3.

**Design**: Implement these tables exactly per `data-model-suggestion-3.md`: `tenants`, `organizational_units`, `facilities`, `emission_factor_libraries`, `emission_factors`, `emission_sources`, `activity_data`, `emissions`, `reporting_periods`, `reporting_frameworks`, `disclosure_responses`, `esg_metrics`, `suppliers`, `supplier_data_requests`, `supplier_responses`, `source_documents`, `audit_log`, `framework_crosswalks`, `users`, `approval_workflows`, `approval_steps`. Plus a `reduction_targets` table (from Model 1) and a `factor_recalculations` table (restatement record borrowed from Model 2):
```sql
CREATE TABLE factor_recalculations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    emission_id     UUID NOT NULL REFERENCES emissions(id),
    reason          VARCHAR(30) NOT NULL CHECK (reason IN ('factor_update','methodology_change','data_correction')),
    old_co2e_tonnes NUMERIC(16,6) NOT NULL,
    new_co2e_tonnes NUMERIC(16,6) NOT NULL,
    delta_co2e_tonnes NUMERIC(16,6) NOT NULL,
    old_emission_factor_id UUID REFERENCES emission_factors(id),
    new_emission_factor_id UUID REFERENCES emission_factors(id),
    triggered_by    UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
Apply all CHECK constraints, GIN indexes on JSONB columns (`facilities.attributes`, `activity_data.details`, `disclosure_responses.response_data`, `esg_metrics.metric_value`, `source_documents.extracted_data`, `framework_crosswalks.mappings`), `pg_trgm` GIN index on `emission_factors.name`, and the JSONB CHECK + `validate_activity_details` trigger from Model 3.

**Testing**:
- `Integration (testcontainers): alembic upgrade head creates all tables; downgrade base drops them cleanly.`
- `Integration: inserting emission_sources with scope=4 → CHECK violation.`
- `Integration: inserting activity_data for a stationary_combustion source without details.fuel_type → trigger raises.`
- `Integration: GIN index used for details @> '{"fuel_type":"natural_gas"}' (EXPLAIN shows index scan on a seeded table).`

#### 1.3 — Multi-tenancy, RLS, and tenant context

**What**: Row-level security enforcing `tenant_id` isolation, plus a request-scoped tenant context.

**Design**:
- Every tenant-scoped table gets an RLS policy: `USING (tenant_id = current_setting('esg.tenant_id')::uuid)`.
- `db/session.py` — `get_session(tenant_id)` sets `SET LOCAL esg.tenant_id = :tid` at transaction start.
- A privileged "system" role bypasses RLS for seeding/admin tasks (factor library loads, framework schema loads).

**Testing**:
- `Integration: two tenants insert facilities; tenant A's session SELECT returns only A's rows.`
- `Integration: attempt to UPDATE another tenant's row under RLS → 0 rows affected.`
- `Integration: system role loads global emission_factors (no tenant_id) successfully.`

#### 1.4 — Audit log & data-lineage substrate

**What**: A mutation interceptor that writes `audit_log` rows and maintains `disclosure_responses.data_sources` lineage arrays.

**Design**:
- SQLAlchemy event listeners on `after_insert/after_update/after_delete` for audited models emit `audit_log` rows: `{old, new, changed_fields}` JSONB (Model 3 shape), `changed_by` from request context, `correlation_id` from the OTel trace.
- A `lineage` helper records which `activity_data`/`esg_metric` IDs fed a given `emissions` row and a `disclosure_response`.

**Testing**:
- `Unit: diff_jsonb(old,new) returns only changed keys.`
- `Integration: updating activity_data.quantity writes one audit_log row with changed_fields=["quantity"] and correct old/new.`
- `Integration: deleting a facility writes an audit_log DELETE row with the prior values.`

---

## Phase 2: GHG Calculation Engine

### Purpose
Build the heart of the product: a deterministic, GHG-Protocol-aligned engine that turns activity data + emission factors into per-gas and CO2e emissions, with correct unit conversion, GWP application, and Scope 2 dual-method handling. This is the core value proposition and ships early. After this phase, given seeded factors and activity data, the system computes auditable emissions.

### Tasks

#### 2.1 — GWP tables and unit conversion

**What**: AR5 and AR6 GWP100 tables and a unit-conversion module.

**Design**:
```python
GWP100: dict[str, dict[str, float]] = {
    "AR5": {"CO2":1, "CH4":28, "N2O":265, "SF6":23500, "NF3":16100},
    "AR6": {"CO2":1, "CH4":27.9, "N2O":273, "SF6":25200, "NF3":17400},
}  # HFC/PFC families handled per-species via factor metadata
def to_base_unit(quantity: Decimal, unit: str, dimension: str) -> Decimal: ...
# dimensions: energy(kWh), volume(L), mass(kg), distance(km), money(currency-agnostic passthrough)
```
All arithmetic uses `Decimal` to preserve audit precision.

**Testing**:
- `Unit: to_base_unit(1, "MWh", "energy") == 1000 kWh; to_base_unit(1, "gal_US", "volume") == 3.785411784 L.`
- `Unit: unknown unit → UnitConversionError.`
- `Unit: GWP100["AR6"]["CH4"] == 27.9.`

#### 2.2 — Factor selection

**What**: Given an emission source + activity date + region, select the applicable active emission factor.

**Design**:
```python
def select_factor(session, *, source_type, region, activity_date, unit_denominator,
                  gas_type="CO2e", preferred_library_id=None) -> EmissionFactor:
    # filter: category match, region match (fallback to broader region/global),
    # valid_from <= activity_date <= valid_to (or NULL), unit_denominator match,
    # is_active; order by region specificity, then library preference, then recency.
```
Raise `NoFactorFoundError` (becomes HTTP 422 with guidance) if none match.

**Testing**:
- `Unit (seeded): electricity kWh in region GB on 2025-03-01 → returns the GB grid factor, not the global default.`
- `Unit: region with no specific factor falls back to global factor.`
- `Unit: activity date outside all factor validity windows → NoFactorFoundError.`

#### 2.3 — Calculation engine (Scope 1 & 2)

**What**: `calculate(activity_data) -> EmissionResult` producing per-gas tonnes, CO2e, biogenic CO2, and Scope 2 location/market figures.

**Design**:
```python
@dataclass(frozen=True)
class EmissionResult:
    co2_tonnes: Decimal; ch4_tonnes: Decimal; n2o_tonnes: Decimal
    hfc_tonnes: Decimal; pfc_tonnes: Decimal; sf6_tonnes: Decimal; nf3_tonnes: Decimal
    co2e_tonnes: Decimal; biogenic_co2_tonnes: Decimal
    scope2_method: Literal["location_based","market_based"] | None
    emission_factor_id: UUID; calculation_method: str
    intermediate_steps: list[dict]   # stored in emissions.calculation_details
ENGINE_VERSION = "1.0.0"
```
Flow: convert activity to factor's denominator unit → multiply by factor → split per-gas if factor is multi-gas else apply CO2e directly → apply GWP → sum CO2e. Scope 2: compute location-based always; compute market-based when contractual instruments (RECs/REGOs) present in `activity_data.details`. Record each arithmetic step in `intermediate_steps` for lineage.

**Testing**:
- `Unit: 1000 kWh × 0.233 kgCO2e/kWh → 0.233 tCO2e (location-based); intermediate_steps has unit_conversion + factor_application.`
- `Unit: natural gas with separate CH4/N2O factors → per-gas tonnes populated, CO2e = Σ(gas×GWP).`
- `Unit: Scope 2 with REGO covering 100% → market_based co2e == 0, location_based > 0.`
- `Unit: biogenic CO2 reported separately, excluded from co2e_tonnes.`

#### 2.4 — Scope 3 calculators (spend-based & activity-based)

**What**: Per-category Scope 3 calculation supporting spend-based and activity-based methods.

**Design**: A registry mapping `scope3_category (1–15) → Calculator`. MVP implements the highest-impact categories: 1 (purchased goods, spend-based via EEIO-style factor `kgCO2e/currency`), 4 (upstream transport, activity-based tonne-km), 6 (business travel, passenger-km by mode/class), 11 (use of sold products). Each calculator reads its required fields from `activity_data.details` (validated by source-type JSON Schema) and returns an `EmissionResult`. PCAF-style `data_quality_score` recorded in `calculation_details`.
```python
class Scope3Calculator(Protocol):
    category: int
    required_detail_fields: set[str]
    def calculate(self, activity, factor_lookup) -> EmissionResult: ...
```

**Testing**:
- `Unit: spend-based Cat 1: $125,000 × 0.42 kgCO2e/USD → 52.5 tCO2e.`
- `Unit: Cat 6 air travel LHR→JFK economy round-trip → expected passenger-km × factor.`
- `Unit: missing required detail field (e.g. spend_amount) → ValidationError before calculation.`
- `Unit: unsupported category → CategoryNotImplementedError (clear message).`

#### 2.5 — Recalculation & restatement

**What**: When a factor is superseded or activity data corrected, recompute affected emissions and record the delta.

**Design**: `recalculate(emission_id, reason) ` reruns the engine with the current applicable factor, writes a new `emissions` row state, and inserts a `factor_recalculations` row capturing `old/new/delta_co2e`. Original figure is preserved in the audit log; the restatement row drives the auditor's year-over-year view.

**Testing**:
- `Integration: publish a new GB grid factor → recalculate → factor_recalculations row with correct delta and new_emission_factor_id.`
- `Integration: correcting activity quantity 1200→1250 kWh recomputes co2e proportionally and records reason='data_correction'.`

---

## Phase 3: Emission Factor Library & Seeding

### Purpose
Deliver the transparent, versioned, open emission-factor library that differentiates the platform. After this phase, public factor datasets (IPCC EFDB, US EPA, GHG Protocol cross-sector) are loadable, versioned, queryable, and searchable, and the calculation engine has real factors to use.

### Tasks

#### 3.1 — Factor ingestion CLI & library versioning

**What**: `esg-seed factors --source EPA --version 2025 --file data/emission_factors/epa-2025.csv`.

**Design**: CSV→`emission_factors` loader keyed by `(source, version)` in `emission_factor_libraries`. Each load creates a new immutable library version; prior versions retain `valid_to`. Records `region`, `gas_type`, `factor_value`, `unit_numerator/denominator`, `gwp_source`, `data_quality_score`, `source_url`, and category technical details in JSONB. Idempotent on re-run (upsert by `factor_code` within library).

**Testing**:
- `Integration: load EPA fixture (50 rows) → 50 emission_factors under one library; re-run → still 50 (idempotent).`
- `Unit: malformed row (non-numeric factor_value) → row rejected, error report lists line number, load continues.`
- `Integration: loading EPA 2026 sets EPA 2025 library valid_to and marks it inactive per policy.`

#### 3.2 — Factor search & browse API

**What**: `GET /api/factors?q=&category=&region=&source=&date=` with fuzzy text and filters.

**Design**: Trigram similarity on `name` (`pg_trgm`) combined with category/region/validity filters; paginated. Response includes provenance (`source`, `version`, `source_url`) to satisfy transparency goals.
```
GET /api/factors?q=natural+gas&region=GB&date=2025-06-01
→ 200 [{id, name, category, region, factor_value, unit_numerator, unit_denominator,
        gwp_source, source, version, source_url, data_quality_score, similarity}]
```

**Testing**:
- `Integration: q="electricty" (typo) still returns the electricity factor via trigram similarity.`
- `Integration: region=GB&date filter excludes US factors and expired factors.`
- `Integration (mocked auth): unauthenticated request → 401.`

#### 3.3 — Factor update notifications

**What**: When a new library version supersedes factors in use, flag affected tenants.

**Design**: On library publish, a Celery task finds `emissions` rows referencing superseded factors and creates per-tenant notifications listing affected emission count and the recommended `recalculate` action (links to Phase 2.5). No automatic mutation — recalculation is user-approved (assurance requirement).

**Testing**:
- `Integration: publish new factor superseding one in use by tenant A → notification created for A only, with correct affected count.`
- `Integration: tenants not using the factor get no notification.`

---

## Phase 4: Data Ingestion (CSV, API, Documents)

### Purpose
Let users get activity data into the system at scale. After this phase, sustainability teams can bulk-import via CSV templates, push via REST, and upload documents that are stored with checksums — wiring directly into the engine so emissions calculate on ingest.

### Tasks

#### 4.1 — Activity data REST API

**What**: CRUD for emission sources and activity data, calculating emissions on create/update.

**Design**:
```
POST /api/sources                 → create emission_source (scope, scope3_category, source_type, configuration)
POST /api/activity                → create activity_data; validates details vs source-type JSON Schema;
                                     enqueues/inline-runs engine; returns emission_id + co2e
GET  /api/emissions?period=&scope= → aggregated + line-item emissions
```
Reject writes to a `locked` reporting period (412). `details` validated by the per-source-type Pydantic/JSON Schema registry from Phase 2.4.

**Testing**:
- `Integration: POST valid electricity activity → 201, emissions row created, co2e matches engine.`
- `Integration: POST activity to locked period → 412, no row written.`
- `Integration: POST with details failing source-type schema → 422 naming the missing field.`

#### 4.2 — CSV bulk import

**What**: Template-driven CSV import with row-level validation and an import report.

**Design**: `POST /api/imports/activity` (multipart). Uses Postgres `COPY` into a staging table, validates each row (source resolution, unit, date within period), calculates emissions for valid rows, returns `{accepted, rejected:[{row,errors}]}`. Published downloadable templates per source type.

**Testing**:
- `Integration: 100-row CSV with 3 bad rows → 97 accepted, 3 in rejected with reasons; 97 emissions created.`
- `Integration: CSV referencing unknown source name → those rows rejected with "source not found".`
- `Fixture: golden CSV template round-trips.`

#### 4.3 — Document upload & storage

**What**: Upload source documents to S3/MinIO with SHA-256 checksums and metadata.

**Design**: `POST /api/documents` streams to object storage, computes checksum, stores `source_documents` row (`file_path`, `checksum_sha256`, `mime_type`, `document_type`). Links to `activity_data.source_document_id` for lineage.

**Testing**:
- `Integration (MinIO container): upload PDF → stored, checksum matches local hash, row created.`
- `Integration: re-upload identical file → checksum matches; configurable dedupe returns existing doc.`
- `Integration: oversized/disallowed mime → 415.`

#### 4.4 — ERP/utility connector interface (one reference connector)

**What**: A pluggable connector interface plus one working reference connector (NetSuite-style REST, mockable).

**Design**:
```python
class Connector(Protocol):
    name: str
    def authenticate(self, creds) -> None: ...    # OAuth 2.0 client-credentials
    def pull_activity(self, since: date) -> Iterable[RawActivity]: ...
```
A field-mapping config translates connector records → `activity_data` (+`details`). SAP/NetSuite are registered; the reference connector hits a mock server in tests.

**Testing**:
- `Integration (mock server): pull_activity since date → N RawActivity mapped to activity_data with correct details.`
- `Unit: field mapping config maps source columns → activity_data fields.`
- `Integration: connector auth failure → ConnectorAuthError, no partial import.`

---

## Phase 5: Multi-Framework Reporting & Disclosure Mapping

### Purpose
Deliver the platform's headline capability: map one dataset to many frameworks via explicit crosswalks and produce disclosure responses. After this phase, a tenant selects applicable frameworks and the system auto-populates equivalent disclosures (e.g. GRI 305-1, ESRS E1-6, IFRS S2.29(a), CDP C6.1) from the same emissions data.

### Tasks

#### 5.1 — Framework schema & crosswalk loader

**What**: Load framework disclosure schemas (`data/frameworks/*.json`) into `reporting_frameworks.disclosure_schema` and the crosswalk into `framework_crosswalks.mappings`.

**Design**: JSON files define each framework's disclosures (code, name, type, unit, fields) per Model 3's `disclosure_schema` shape. The crosswalk (Model 3 `mappings` shape) maps a canonical concept → per-framework `{disclosure, field}` plus `source_table`, `source_filter`, `aggregation`. Loaded by the system role; cached in Redis.

**Testing**:
- `Integration: load gri.json + esrs.json → reporting_frameworks rows with valid schemas; invalid JSON → load aborts with path of bad framework.`
- `Unit: crosswalk validator rejects a mapping referencing a disclosure code absent from any loaded framework.`

#### 5.2 — Disclosure population engine

**What**: Generate `disclosure_responses` for a tenant/period/framework set from emissions + ESG metrics via the crosswalk.

**Design**: For each canonical concept in the crosswalk, run its `source_filter` aggregation against `emissions`/`esg_metrics`, then write the value into every applicable framework's disclosure `response_data` at the mapped JSON path. Each response records its `data_sources` (lineage). Quantitative disclosures auto-fill; narrative disclosures created as `draft` placeholders for Phase 8 AI drafting.
```
POST /api/reports/populate {period_id, frameworks:["GRI","CSRD_ESRS","ISSB_S2","CDP"]}
→ 202 {created: n, frameworks: {...completion %...}}
```

**Testing**:
- `Integration: seeded emissions → populate GRI+ESRS+ISSB → all three Scope 1 disclosures carry the same total; data_sources populated.`
- `Integration: location- vs market-based Scope 2 mapped to correct ESRS E1-6 fields.`
- `Integration: re-populate is idempotent (UNIQUE on tenant/period/framework/disclosure upserts).`

#### 5.3 — Report export (PDF, JSON, Open Footprint)

**What**: Export disclosure responses as audit-ready PDF, machine-readable JSON, and an Open Footprint Forum-shaped interoperability export.

**Design**: `GET /api/reports/{period}/export?format=pdf|json|openfootprint&framework=`. PDF embeds data-lineage references per figure (assurance requirement). Open Footprint export maps emissions to the OFF data model for portability.

**Testing**:
- `Integration: export JSON → schema-valid, every figure has a lineage reference.`
- `Integration: PDF export renders without error and contains the Scope 1/2/3 totals (text extraction check).`
- `Unit: Open Footprint mapper produces OFF-schema-valid output for a sample inventory.`

#### 5.4 — XBRL / Inline-XBRL export (ESRS & GRI)

**What**: Generate ESEF Inline XBRL for CSRD/ESRS and XBRL for GRI using `arelle`.

**Design**: Map `disclosure_responses` to the ESRS XBRL taxonomy concepts and emit iXBRL; map to GRI taxonomy for GRI filings. Validate output against the taxonomy with `arelle`.

**Testing**:
- `Integration: generate ESRS iXBRL for a sample E1-6 set → arelle validation passes against the taxonomy fixture.`
- `Fixture: golden iXBRL comparison for a known input (normalised diff).`
- `Unit: missing mandatory tagged concept → export raises with the concept name.`

---

## Phase 6: Supplier Engagement & Scope 3 Portal

### Purpose
Solve the hardest operational problem in ESG — Scope 3 supplier data collection — with a login-less supplier portal, request/reminder workflows, and data-quality scoring. After this phase, procurement teams send requests, suppliers submit data via a tokenised link, and submissions flow into Scope 3 emissions.

### Tasks

#### 6.1 — Supplier registry & data requests

**What**: Manage suppliers and create/send data requests for a period.

**Design**: `suppliers` (+ JSONB `profile`), `supplier_data_requests` with `request_template` JSONB defining requested scopes/metrics. `POST /api/suppliers/{id}/requests {period_id, template}` → generates a signed, expiring portal token (JWT, scoped to the request).

**Testing**:
- `Integration: create request → status='draft'; send → 'sent', signed token generated, decodes to the request id.`
- `Unit: portal token expired/tampered → 401 on portal access.`

#### 6.2 — Login-less supplier portal

**What**: Public portal endpoints (and Next.js route group) where a supplier opens the tokenised link, sees the questionnaire, uploads evidence, and submits.

**Design**: `GET /portal/{token}` returns the request template; `POST /portal/{token}/submit` validates `response_data` against the template schema, stores `supplier_responses`, attaches uploaded evidence document. No supplier account required (GDPR-minimal: only contact + submitted data).

**Testing**:
- `E2E (Playwright): open portal link → fill questionnaire → upload PDF → submit → confirmation; response stored.`
- `Integration: submit with response failing template schema → 422 listing offending fields.`
- `Integration: submit after due date still accepted but flagged late.`

#### 6.3 — Reminder workflow & data-quality scoring

**What**: Automated reminders for outstanding requests and a quality score on submissions.

**Design**: Celery beat scans `supplier_data_requests` for due/overdue items, sends SMTP reminders, increments `reminder_count`. Quality scoring rubric (completeness, third-party verification flag, methodology disclosed, recency) writes `data_quality_score` (1–5) to the response.

**Testing**:
- `Integration (mocked SMTP): overdue request → reminder sent, reminder_count++, last_reminder_at set.`
- `Unit: scoring rubric — verified + full methodology + current year → score 5; sparse self-reported → low score.`
- `Integration: validated supplier response promotes into a Scope 3 emissions entry (supplier-specific method).`

#### 6.4 — Procurement dashboard data

**What**: Aggregate endpoints for supplier engagement status and Scope 3 hotspots.

**Design**: `GET /api/suppliers/engagement?period=` → per-supplier status, spend, reported emissions, quality, risk; `GET /api/scope3/hotspots?period=` ranks suppliers/categories by emissions and emission-intensity-per-spend.

**Testing**:
- `Integration: engagement endpoint returns correct status counts (sent/submitted/overdue).`
- `Integration: hotspots ranks the highest-emitting supplier first with intensity-per-spend computed.`

---

## Phase 7: Governance, RBAC & Approval Workflows

### Purpose
Make the platform assurance-ready with authentication, role-based access, four-eyes approval, and period locking. After this phase, data and disclosures move through draft → review → approval → published with enforced segregation of duties, and periods lock for filing.

### Tasks

#### 7.1 — Auth (OAuth 2.0 / OIDC) & users

**What**: OIDC login, JWT sessions, and the seven-role user model.

**Design**: Authlib OIDC; roles `admin, sustainability_manager, data_entry, reviewer, approver, auditor, viewer` (Model 1). `api/deps.py` provides `require_role(...)` and injects tenant + user into request context (feeds audit log).

**Testing**:
- `Integration (mock OIDC): valid code exchange → JWT issued with tenant+role claims.`
- `Unit: require_role('approver') rejects a 'data_entry' token → 403.`
- `Integration: auditor role has read access to lineage endpoints but 403 on writes.`

#### 7.2 — Four-eyes approval workflow

**What**: Submit→review→approve flow on activity data, emissions, and disclosures.

**Design**: `approval_workflows` + `approval_steps`. Invariant (from Model 2): the reviewer/approver must differ from the submitter. State machine: `pending → in_review → approved | rejected | escalated`.
```
POST /api/approvals {entity_type, entity_id} → workflow pending
POST /api/approvals/{id}/decision {decision, comments}
```

**Testing**:
- `Integration: submitter cannot approve own submission → 403.`
- `Integration: approve transitions entity status to 'approved'; reject returns it with comments.`
- `Unit: state machine rejects illegal transition (approved → in_review).`

#### 7.3 — Period locking & restatement gate

**What**: Lock a period for filing; require approvals; control unlock.

**Design**: Lock requires all mandatory disclosures `approved` (Model 2 invariant). Locked periods reject data writes. Unlock requires `approver`+ role and a reason; unlocking a published period forces the restatement path (Phase 2.5).

**Testing**:
- `Integration: lock with an unapproved mandatory disclosure → 409 listing the gap.`
- `Integration: lock then attempt activity write → 412.`
- `Integration: unlock by data_entry → 403; by approver with reason → audit_log records it.`

---

## Phase 8: AI-Native Layer

### Purpose
Deliver the AI-native advantages that differentiate the platform: document extraction, NLP factor matching, anomaly detection, narrative drafting, double-materiality assistance, and a Copilot. After this phase, manual data entry drops sharply and data quality improves before audit. All AI is provider-abstracted with deterministic test fallbacks.

### Tasks

#### 8.1 — LLM gateway & prompt registry

**What**: One internal interface for all LLM/embedding calls with provider abstraction.

**Design**:
```python
class LLMGateway(Protocol):
    def complete(self, system: str, user: str, *, json_schema: dict | None=None) -> dict | str: ...
    def embed(self, texts: list[str]) -> list[list[float]]: ...
```
Providers: `openai` (OpenAI-compatible), `ollama` (local), `null` (deterministic stub returning fixtures — used in tests). Prompts live in `integrations/llm/prompts/` and are versioned.

**Testing**:
- `Unit: null provider returns the registered fixture for a prompt id (deterministic).`
- `Integration (mocked HTTP): openai provider parses structured JSON output against json_schema.`

#### 8.2 — Document ingestion (OCR + LLM extraction)

**What**: Turn an uploaded utility bill/invoice into structured `extracted_data` and a proposed `activity_data` draft.

**Design**: Celery task: Tesseract OCR → LLM structures fields (`supplier, billing_period, total_kwh, total_cost, meter_readings`) per a JSON schema → writes `source_documents.extracted_data` (with `ocr_confidence`, `classification`) → creates a *draft* activity entry for human confirmation. Prompt template included in the registry.

**Testing**:
- `Integration (null LLM + fixture OCR text): sample electricity bill → extracted_data has total_kwh and billing_period; draft activity created (status pending).`
- `Unit: low OCR confidence (<0.6) flags the document for manual review, no draft created.`
- `Integration: classification routes a fuel receipt to mobile_combustion source type.`

#### 8.3 — NLP emission-factor matching

**What**: Map free-text activity descriptions to the correct factor/category.

**Design**: Embedding similarity over factor `name`+`category` (vectors cached) combined with `pg_trgm` lexical match; returns ranked candidates with confidence. Low-confidence matches require human confirmation.
```
POST /api/factors/match {description:"Aluminium ingots Grade A", region, date}
→ [{factor_id, name, confidence}]  (ranked)
```

**Testing**:
- `Unit (null embeddings/fixtures): "diesel for backup generator" → top candidate is the diesel stationary-combustion factor.`
- `Integration: ambiguous description → multiple candidates, top confidence < threshold → flagged for review.`

#### 8.4 — Anomaly detection

**What**: Flag outliers in emissions data before submission.

**Design**: Per source/scope, compute z-score vs trailing-12-period mean (the windowed approach from Model 4, run in app/SQL). `z>2 → ANOMALY`, `1.5–2 → WARNING`. Surfaced on `activity_data.status='flagged'` and in a review queue. (No TimescaleDB dependency in MVP.)

**Testing**:
- `Unit: a 10× spike vs stable history → ANOMALY; a 1.6σ deviation → WARNING; normal → none.`
- `Integration: flagged entries appear in GET /api/review/anomalies.`

#### 8.5 — Narrative drafting & double-materiality assist

**What**: LLM drafts for narrative disclosures (grounded in structured data) and assisted ESRS double-materiality scoring.

**Design**: Narrative drafting passes the relevant approved quantitative figures + disclosure requirement text to the LLM, returns a draft into `disclosure_responses.response_data` as `draft` (always human-reviewed). Materiality assist scores topics on financial- and impact-materiality (0–1) from tenant/industry context, producing a draft matrix for `materiality` endpoints.

**Testing**:
- `Integration (null LLM fixture): draft GRI 305 narrative references the actual Scope 1 total from the dataset (no hallucinated number — number injected, not generated).`
- `Unit: every AI-generated disclosure is persisted with status='draft' (never auto-approved).`
- `Integration: materiality assist returns scored topics; result stored as draft requiring sign-off.`

#### 8.6 — Copilot (NL query)

**What**: Natural-language Q&A over the tenant's structured ESG data.

**Design**: `POST /api/copilot {question}` → LLM translates to a constrained, parameterised query against a read-only view set (never free SQL), executes, and returns a grounded answer with the figures and their lineage. Tenant-scoped; respects RBAC.

**Testing**:
- `Integration (null LLM fixture): "What were our Scope 2 market-based emissions for FY2025?" → returns the stored value with lineage.`
- `Unit: copilot cannot access another tenant's data (RLS + tenant context).`
- `Unit: query generation is constrained to the allowlisted views (no arbitrary SQL).`

---

## Phase 9: Frontend (Dashboard + Portal)

### Purpose
Provide the dashboard-centric UI that incumbents set as the table-stakes UX, plus the supplier portal screens. After this phase, all personas have a working web experience over the API.

### Tasks

#### 9.1 — App shell, auth, and API client

**What**: Next.js app with OIDC login, role-aware navigation, and a typed API client generated from the OpenAPI spec.

**Design**: `lib/api-client.ts` generated from `/openapi.json`; auth context stores JWT; route groups `(dashboard)` and `(portal)`.

**Testing**:
- `E2E: login → land on dashboard; logout clears session.`
- `Unit: nav hides admin routes for viewer role.`

#### 9.2 — Emissions dashboard & data entry

**What**: Scope 1/2/3 overview with hotspots, drill-down, and activity entry/import screens.

**Design**: Recharts visualisations from `GET /api/emissions`; CSV import UI with the row-error report; factor-search picker (Phase 3.2) integrated into manual entry.

**Testing**:
- `E2E: import CSV → dashboard totals update to match engine output.`
- `E2E: manual entry with factor picker → emission shown with provenance.`

#### 9.3 — Reporting & disclosure workspace

**What**: Framework selection, populate, review/approve, and export.

**Design**: Completion tracker per framework (from populate response); disclosure editor showing auto-filled figures + lineage; export buttons (PDF/JSON/XBRL); approval actions for reviewer/approver roles.

**Testing**:
- `E2E: select GRI+ESRS → populate → completion % shown → approve → export PDF downloads.`
- `E2E: data_entry user cannot see approve action.`

#### 9.4 — Supplier & procurement screens

**What**: Supplier list, request creation, engagement dashboard, and the public portal pages.

**Design**: Procurement engagement dashboard (status heatmap, hotspots from Phase 6.4); public portal questionnaire/upload screens (Phase 6.2).

**Testing**:
- `E2E: create + send supplier request → status 'sent'; open portal link in a fresh context → submit → status 'submitted'.`

---

## Phase 10: Packaging, Observability & Deployment

### Purpose
Make the platform installable and operable: production containers, Helm chart, observability, backups, and seed data. After this phase a new operator can stand up the full stack and load public factor libraries and framework schemas.

### Tasks

#### 10.1 — Production images & Helm chart

**What**: Multi-stage Dockerfiles, `docker-compose.prod.yml`, and a Helm chart for k8s.

**Design**: Separate API/worker/web images; chart templates for Postgres, Redis, MinIO (or external), API, worker, web, and a one-shot seed Job.

**Testing**:
- `Integration: docker-compose.prod.yml up → /healthz green; seed job loads factors + frameworks.`
- `CI: helm template renders; helm lint passes.`

#### 10.2 — Observability & SOC 2 controls

**What**: OTel tracing across ingest→calculate→disclose, Prometheus metrics, structured logs, and audit-log retention.

**Design**: Trace spans tagged with `tenant_id` and `correlation_id`; metrics for calculation latency, queue depth, factor-match confidence; log shipping config; documented audit-log retention (7–10 yr).

**Testing**:
- `Integration: a single activity ingest produces one trace spanning ingest → calc → audit.`
- `Integration: /metrics exposes calculation_count and queue_depth.`

#### 10.3 — Backup, restore & seed bundle

**What**: Backup/restore scripts and a documented seed bundle (factor libraries + framework schemas + crosswalk + sample tenant).

**Design**: `pg_dump`/`pg_restore` wrappers; MinIO bucket sync; `esg-seed all` loads the public-data bundle and an optional demo tenant.

**Testing**:
- `Integration: backup → drop → restore → row counts and a known emission value match.`
- `Integration: esg-seed all on an empty DB yields a queryable demo tenant with non-zero emissions.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Data Model & Tenancy        ── required by everything
    │
Phase 2: GHG Calculation Engine                  ── requires 1
    │
Phase 3: Emission Factor Library & Seeding       ── requires 1,2 (gives 2 real factors)
    │
Phase 4: Data Ingestion (CSV/API/Docs)           ── requires 2,3
    │
    ├── Phase 5: Multi-Framework Reporting        ── requires 4
    ├── Phase 6: Supplier Engagement & Scope 3    ── requires 4 (can parallel 5)
    └── Phase 7: Governance, RBAC & Approvals     ── requires 1; integrates with 4,5,6 (can parallel 5,6)
         │
Phase 8: AI-Native Layer                         ── requires 3,4,5 (matching, drafting, copilot)
    │
Phase 9: Frontend (Dashboard + Portal)           ── requires 4,5,6,7 (consumes their APIs)
    │
Phase 10: Packaging, Observability & Deployment  ── requires all; final hardening
```

**Parallelism opportunities:**
- After Phase 4, **Phases 5, 6, and 7** can be built concurrently by separate contributors (reporting, supplier portal, governance) — they touch largely disjoint tables.
- Within Phase 8, factor matching (8.3), anomaly detection (8.4), and document ingestion (8.2) are independent once the LLM gateway (8.1) exists.
- Frontend (Phase 9) sub-tasks can begin against mocked endpoints as soon as each backend API's OpenAPI schema is stable.

**Estimated scope: large** (10 phases, 38 tasks; full-stack platform with AI pipeline, multi-framework mapping, and a public portal).

---

## Definition of Done (per phase)

A phase is complete only when:

1. All tasks implemented and merged.
2. All unit and integration tests pass (`pytest`); integration tests run against a real Postgres via testcontainers.
3. E2E tests pass where the phase adds user-facing flows (Playwright).
4. `ruff check` and `ruff format --check` pass.
5. `mypy --strict` passes for `src/esg`.
6. `docker build` succeeds for all affected images; `docker-compose up` boots with `/healthz` green.
7. The phase's feature works end-to-end against the running stack (demonstrated via the seed/demo tenant).
8. New config options documented in `.env.example` and README.
9. New/changed API endpoints appear in the generated OpenAPI 3.1 spec, and the frontend API client regenerates without type errors.
10. Alembic migration created, and `upgrade head` / `downgrade` both succeed cleanly.
11. New mutations are covered by audit-log assertions (lineage preserved).
12. Tenant isolation verified (no cross-tenant leakage) for any new tenant-scoped table or endpoint.
```
