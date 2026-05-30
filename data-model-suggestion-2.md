# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Sustainability & ESG Reporting (457)
> Model: Event Sourcing with CQRS on PostgreSQL (event store) + read-optimised projections
> Generated: 2026-05-25

---

## Design Philosophy

ESG reporting is fundamentally an audit-trail problem. Regulators, assurance providers, and internal governance teams all demand the ability to answer: "What was the state of this data at any point in time? Who changed it, when, and why?" Event sourcing answers this by design -- rather than storing current state and logging changes as an afterthought, the event log _is_ the source of truth, and current state is derived by replaying events.

This model stores every business action as an immutable event in an append-only event store. Read models (projections) are materialised from the event stream to serve dashboards, reports, and API queries. The CQRS pattern separates the write path (commands that produce events) from the read path (queries against projections), allowing each to be optimized independently.

This architecture is particularly well-suited to the ESG domain because:
- **Assurance readiness is built-in.** The event log provides a tamper-evident, complete history that auditors can traverse. No separate audit log table is needed.
- **Emission factor version changes** can be replayed: when a new IPCC AR6 factor is published, the system can re-project emissions using updated factors and show the delta.
- **Multi-framework reporting** becomes a projection concern: the same event stream generates GRI, CSRD, ISSB, and CDP views without duplicating source data.
- **Regulatory corrections** (restatements) are modelled as new events rather than overwrites, preserving the original record.

---

## Event Store Schema

```sql
-- The central event store: append-only, immutable
CREATE TABLE event_store (
    event_id            BIGSERIAL PRIMARY KEY,
    stream_id           UUID NOT NULL,              -- aggregate root identifier
    stream_type         VARCHAR(50) NOT NULL,       -- aggregate type
    event_type          VARCHAR(100) NOT NULL,      -- domain event name
    event_version       INTEGER NOT NULL,           -- position within stream
    payload             JSONB NOT NULL,             -- event data
    metadata            JSONB NOT NULL DEFAULT '{}',-- correlation_id, causation_id, user_id, tenant_id
    tenant_id           UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Optimistic concurrency: each stream has monotonically increasing versions
    UNIQUE(stream_id, event_version)
);

-- Primary access patterns
CREATE INDEX idx_es_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_es_tenant ON event_store(tenant_id, created_at);
CREATE INDEX idx_es_type ON event_store(event_type, created_at);
CREATE INDEX idx_es_stream_type ON event_store(stream_type, created_at);

-- Partition by month for performance at scale
-- CREATE TABLE event_store_y2026m01 PARTITION OF event_store
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Snapshots for aggregates with long event histories
CREATE TABLE event_snapshots (
    stream_id           UUID PRIMARY KEY,
    stream_type         VARCHAR(50) NOT NULL,
    snapshot_version    INTEGER NOT NULL,
    state               JSONB NOT NULL,
    tenant_id           UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Outbox table for reliable event publishing to message brokers
CREATE TABLE event_outbox (
    id                  BIGSERIAL PRIMARY KEY,
    event_id            BIGINT NOT NULL REFERENCES event_store(event_id),
    destination         VARCHAR(100) NOT NULL,      -- topic/queue name
    published           BOOLEAN DEFAULT FALSE,
    published_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_outbox_unpublished ON event_outbox(published, created_at) WHERE NOT published;
```

---

## Domain Event Types

### Organisation & Structure Events

```
OrganisationRegistered
  { tenant_id, name, legal_name, industry_code, country_code, boundary_approach, base_year }

OrganisationalUnitCreated
  { unit_id, tenant_id, parent_unit_id, name, unit_type, ownership_pct }

OrganisationalUnitRestructured
  { unit_id, old_parent_id, new_parent_id, reason, effective_date }

FacilityRegistered
  { facility_id, tenant_id, org_unit_id, name, type, address, coordinates, floor_area_m2 }

FacilityDecommissioned
  { facility_id, decommission_date, reason }

FacilityDetailsUpdated
  { facility_id, changed_fields: { field: { old, new } } }
```

### Emission Source & Activity Data Events

```
EmissionSourceDefined
  { source_id, tenant_id, facility_id, scope, scope3_category, source_type, name, description }

EmissionSourceDeactivated
  { source_id, reason, effective_date }

ActivityDataRecorded
  { activity_data_id, source_id, period_id, activity_date, quantity, unit,
    data_origin, data_quality_score, source_document_id }

ActivityDataCorrected
  { activity_data_id, original_quantity, corrected_quantity, original_unit,
    corrected_unit, correction_reason, corrected_by }

ActivityDataValidated
  { activity_data_id, validated_by, validation_result, flags[], comments }

ActivityDataRejected
  { activity_data_id, rejected_by, rejection_reason }

ActivityDataApproved
  { activity_data_id, approved_by, approval_level }
```

### Emissions Calculation Events

```
EmissionsCalculated
  { emission_id, activity_data_id, emission_factor_id, emission_factor_version,
    scope, scope3_category, calculation_method,
    co2_tonnes, ch4_tonnes, n2o_tonnes, hfc_tonnes, pfc_tonnes, sf6_tonnes, nf3_tonnes,
    co2e_tonnes, scope2_method, biogenic_co2_tonnes,
    calculation_engine_version }

EmissionsRecalculated
  { emission_id, original_emission_id, reason: 'factor_update' | 'methodology_change' | 'data_correction',
    old_co2e_tonnes, new_co2e_tonnes, delta_co2e_tonnes,
    old_emission_factor_id, new_emission_factor_id }

EmissionsRestatement
  { restatement_id, tenant_id, period_id, scope, reason,
    original_total_co2e, restated_total_co2e, explanation }
```

### Emission Factor Events

```
EmissionFactorLibraryPublished
  { library_id, name, source, version, publication_date, factor_count }

EmissionFactorCreated
  { factor_id, library_id, name, category, region, gas_type, factor_value,
    unit_numerator, unit_denominator, gwp_source, valid_from, valid_to }

EmissionFactorDeprecated
  { factor_id, superseded_by_id, deprecation_reason, effective_date }
```

### Supplier Engagement Events

```
SupplierOnboarded
  { supplier_id, tenant_id, name, industry_code, country_code, tier, annual_spend }

SupplierDataRequested
  { request_id, supplier_id, period_id, request_type, due_date }

SupplierReminderSent
  { request_id, reminder_number, sent_at, channel: 'email' | 'portal' }

SupplierDataSubmitted
  { request_id, response_id, scope1_co2e, scope2_co2e, scope3_co2e, total_co2e,
    methodology, data_quality_score, supporting_document_id }

SupplierDataValidated
  { response_id, validated_by, quality_score, flags[] }

SupplierDataRejected
  { response_id, rejected_by, rejection_reason }

SupplierScoreUpdated
  { supplier_id, old_risk_score, new_risk_score, scoring_factors }
```

### Reporting & Disclosure Events

```
ReportingPeriodOpened
  { period_id, tenant_id, name, period_type, start_date, end_date }

ReportingPeriodLocked
  { period_id, locked_by, lock_reason }

ReportingPeriodUnlocked
  { period_id, unlocked_by, unlock_reason, approval_reference }

FrameworkApplicabilityDeclared
  { tenant_id, framework_id, period_id, is_mandatory, is_voluntary }

DisclosureResponseDrafted
  { response_id, tenant_id, period_id, framework_disclosure_id,
    response_text, response_numeric, unit }

DisclosureResponseReviewed
  { response_id, reviewer_id, decision: 'approved' | 'returned' | 'rejected', comments }

DisclosureResponsePublished
  { response_id, published_by, publication_format: 'xbrl' | 'pdf' | 'html' | 'api' }

ReductionTargetSet
  { target_id, tenant_id, target_type, scope_coverage, baseline_year,
    baseline_co2e, target_year, target_reduction_pct, sbti_validated }

ReductionTargetProgressRecorded
  { target_id, period_id, actual_co2e, reduction_pct_achieved, on_track }
```

### Source Document Events

```
SourceDocumentUploaded
  { document_id, tenant_id, document_type, file_name, file_path, checksum_sha256, uploaded_by }

SourceDocumentOcrProcessed
  { document_id, ocr_confidence, extracted_fields, processing_engine_version }

SourceDocumentClassified
  { document_id, classification_result, confidence_score, model_version }
```

---

## Read Model Projections (CQRS Query Side)

### Projection: Current Emissions Summary

```sql
-- Materialised from EmissionsCalculated and EmissionsRecalculated events
CREATE TABLE rm_emissions_summary (
    tenant_id           UUID NOT NULL,
    reporting_period_id UUID NOT NULL,
    scope               SMALLINT NOT NULL,
    scope3_category     SMALLINT,
    scope2_method       VARCHAR(20),
    total_co2e_tonnes   NUMERIC(16,6) NOT NULL DEFAULT 0,
    total_co2_tonnes    NUMERIC(16,6) NOT NULL DEFAULT 0,
    total_ch4_tonnes    NUMERIC(16,6) NOT NULL DEFAULT 0,
    total_n2o_tonnes    NUMERIC(16,6) NOT NULL DEFAULT 0,
    total_biogenic_co2  NUMERIC(16,6) NOT NULL DEFAULT 0,
    emission_count      INTEGER NOT NULL DEFAULT 0,
    last_calculated_at  TIMESTAMPTZ,
    last_event_id       BIGINT NOT NULL,         -- tracks projection progress
    PRIMARY KEY (tenant_id, reporting_period_id, scope, COALESCE(scope3_category, 0),
                 COALESCE(scope2_method, 'none'))
);
```

### Projection: Facility-Level Emissions Detail

```sql
CREATE TABLE rm_facility_emissions (
    tenant_id           UUID NOT NULL,
    facility_id         UUID NOT NULL,
    facility_name       VARCHAR(255),
    reporting_period_id UUID NOT NULL,
    scope               SMALLINT NOT NULL,
    source_type         VARCHAR(50),
    co2e_tonnes         NUMERIC(16,6) NOT NULL DEFAULT 0,
    activity_count      INTEGER NOT NULL DEFAULT 0,
    avg_data_quality     NUMERIC(3,1),
    last_event_id       BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, facility_id, reporting_period_id, scope, COALESCE(source_type, 'all'))
);
```

### Projection: Supplier Engagement Dashboard

```sql
CREATE TABLE rm_supplier_engagement (
    tenant_id           UUID NOT NULL,
    supplier_id         UUID NOT NULL,
    supplier_name       VARCHAR(255),
    reporting_period_id UUID NOT NULL,
    tier                SMALLINT,
    annual_spend        NUMERIC(16,2),
    request_status      VARCHAR(20),
    request_sent_at     TIMESTAMPTZ,
    due_date            DATE,
    reminder_count      INTEGER DEFAULT 0,
    reported_co2e       NUMERIC(16,6),
    data_quality_score  SMALLINT,
    risk_score          NUMERIC(5,2),
    last_event_id       BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, supplier_id, reporting_period_id)
);
```

### Projection: Framework Disclosure Completion

```sql
CREATE TABLE rm_disclosure_completion (
    tenant_id           UUID NOT NULL,
    reporting_period_id UUID NOT NULL,
    framework_code      VARCHAR(30) NOT NULL,
    framework_name      VARCHAR(255),
    total_disclosures   INTEGER NOT NULL DEFAULT 0,
    drafted             INTEGER NOT NULL DEFAULT 0,
    reviewed            INTEGER NOT NULL DEFAULT 0,
    approved            INTEGER NOT NULL DEFAULT 0,
    published           INTEGER NOT NULL DEFAULT 0,
    completion_pct      NUMERIC(5,1) DEFAULT 0,
    last_event_id       BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, reporting_period_id, framework_code)
);
```

### Projection: Audit Trail View

```sql
-- Flattened audit trail for compliance queries -- no separate audit log needed
CREATE TABLE rm_audit_trail (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    event_id            BIGINT NOT NULL,
    event_type          VARCHAR(100) NOT NULL,
    stream_type         VARCHAR(50) NOT NULL,
    stream_id           UUID NOT NULL,
    actor_id            UUID,
    actor_email         VARCHAR(255),
    action_summary      TEXT NOT NULL,          -- human-readable description
    affected_entity     VARCHAR(100),
    affected_entity_id  UUID,
    timestamp           TIMESTAMPTZ NOT NULL,
    correlation_id      UUID,
    ip_address          INET
);
CREATE INDEX idx_rm_audit_tenant ON rm_audit_trail(tenant_id, timestamp DESC);
CREATE INDEX idx_rm_audit_entity ON rm_audit_trail(affected_entity, affected_entity_id);
CREATE INDEX idx_rm_audit_actor ON rm_audit_trail(actor_id, timestamp DESC);
```

### Projection: Emissions Time Series (for trend analysis)

```sql
CREATE TABLE rm_emissions_timeseries (
    tenant_id           UUID NOT NULL,
    period_month        DATE NOT NULL,           -- first day of month
    scope               SMALLINT NOT NULL,
    scope3_category     SMALLINT,
    co2e_tonnes         NUMERIC(16,6) NOT NULL DEFAULT 0,
    cumulative_co2e     NUMERIC(16,6) NOT NULL DEFAULT 0,
    yoy_change_pct      NUMERIC(7,2),
    last_event_id       BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, period_month, scope, COALESCE(scope3_category, 0))
);
```

---

## Event Processing Architecture

### Command Handlers (Write Side)

```
                    ┌──────────────┐
                    │   REST API   │
                    │  (Commands)  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   Command    │
                    │   Handler    │
                    └──────┬───────┘
                           │
              ┌────────────▼────────────┐
              │     Aggregate Root      │
              │  (Domain Validation)    │
              │  - OrganisationAgg      │
              │  - EmissionSourceAgg    │
              │  - SupplierAgg          │
              │  - ReportingPeriodAgg   │
              └────────────┬────────────┘
                           │
                    ┌──────▼───────┐
                    │  Event Store │
                    │ (PostgreSQL) │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   Outbox     │
                    │  Publisher   │
                    └──────────────┘
```

### Event Projectors (Read Side)

```
    ┌──────────────┐     ┌───────────────────────────────────────────┐
    │  Event Store │     │          Event Projectors                 │
    │              ├────►│                                           │
    │  (Polling /  │     │  ┌──────────────────────────────────┐    │
    │   CDC)       │     │  │ EmissionsSummaryProjector        │    │
    │              │     │  │ → rm_emissions_summary            │    │
    │              │     │  └──────────────────────────────────┘    │
    │              │     │  ┌──────────────────────────────────┐    │
    │              │     │  │ FacilityEmissionsProjector       │    │
    │              │     │  │ → rm_facility_emissions           │    │
    │              │     │  └──────────────────────────────────┘    │
    │              │     │  ┌──────────────────────────────────┐    │
    │              │     │  │ SupplierEngagementProjector      │    │
    │              │     │  │ → rm_supplier_engagement          │    │
    │              │     │  └──────────────────────────────────┘    │
    │              │     │  ┌──────────────────────────────────┐    │
    │              │     │  │ DisclosureCompletionProjector    │    │
    │              │     │  │ → rm_disclosure_completion        │    │
    │              │     │  └──────────────────────────────────┘    │
    │              │     │  ┌──────────────────────────────────┐    │
    │              │     │  │ AuditTrailProjector              │    │
    │              │     │  │ → rm_audit_trail                  │    │
    │              │     │  └──────────────────────────────────┘    │
    │              │     │  ┌──────────────────────────────────┐    │
    │              │     │  │ EmissionsTimeSeriesProjector     │    │
    │              │     │  │ → rm_emissions_timeseries         │    │
    │              │     │  └──────────────────────────────────┘    │
    └──────────────┘     └───────────────────────────────────────────┘
```

---

## Key Operations as Event Flows

### Recording Activity Data and Calculating Emissions

```
1. User submits activity data via API
   → Command: RecordActivityData { source_id, period_id, quantity, unit, ... }

2. EmissionSourceAggregate validates:
   - Source exists and is active
   - Period is not locked
   - Quantity and unit are plausible (anomaly check)

3. Event emitted: ActivityDataRecorded { ... }

4. GHG Calculation Engine subscribes to ActivityDataRecorded:
   - Selects appropriate emission factor based on source type, region, date
   - Calculates individual GHG quantities using GHG Protocol methodology
   - Applies GWP values (AR5 or AR6 as configured)

5. Event emitted: EmissionsCalculated { ... }

6. Projectors update:
   - rm_emissions_summary (increment totals)
   - rm_facility_emissions (increment facility totals)
   - rm_emissions_timeseries (update monthly aggregation)
   - rm_audit_trail (log the activity)
```

### Emission Factor Update and Recalculation

```
1. Admin publishes new emission factor library version
   → Command: PublishEmissionFactorLibrary { library_id, factors[] }

2. Events emitted:
   - EmissionFactorLibraryPublished { ... }
   - EmissionFactorCreated { ... } (for each new factor)
   - EmissionFactorDeprecated { ... } (for superseded factors)

3. Recalculation Engine identifies affected emissions:
   - Queries event store for EmissionsCalculated events using deprecated factors
   - For each: recalculates using new factor

4. Events emitted: EmissionsRecalculated { old_co2e, new_co2e, delta, reason }

5. Projectors update summaries with corrected totals
6. Notification service alerts affected users of restatement
```

### Time-Travel Query (Auditor Use Case)

```
Auditor asks: "What were the Scope 1 emissions as reported on 2025-12-15?"

1. Query the event store for all events in the EmissionSource and Emission streams
   WHERE tenant_id = X AND created_at <= '2025-12-15T23:59:59Z'

2. Replay events to reconstruct state as of that timestamp

3. Return point-in-time emissions summary

-- OR use a pre-built temporal projection:
SELECT * FROM rm_emissions_summary_temporal
WHERE tenant_id = X AND as_of_date = '2025-12-15';
```

---

## Aggregate Root Definitions

### OrganisationAggregate

```
Stream type: Organisation
Commands: RegisterOrganisation, UpdateBoundaryApproach, SetBaseYear
Events: OrganisationRegistered, BoundaryApproachChanged, BaseYearUpdated
Invariants:
  - Boundary approach must be one of: operational_control, financial_control, equity_share
  - Base year cannot be in the future
  - Changing boundary approach triggers restatement workflow
```

### EmissionSourceAggregate

```
Stream type: EmissionSource
Commands: DefineEmissionSource, RecordActivityData, ValidateActivityData, ApproveActivityData
Events: EmissionSourceDefined, ActivityDataRecorded, ActivityDataValidated, ActivityDataApproved
Invariants:
  - Cannot record activity data for a locked reporting period
  - Scope must be 1, 2, or 3; scope3_category required if scope = 3
  - Activity data quantity must be non-negative
  - Validation must precede approval (four-eyes)
```

### SupplierAggregate

```
Stream type: Supplier
Commands: OnboardSupplier, RequestSupplierData, RecordSupplierResponse, ValidateSupplierData
Events: SupplierOnboarded, SupplierDataRequested, SupplierDataSubmitted, SupplierDataValidated
Invariants:
  - Cannot request data from inactive supplier
  - Reminder cannot be sent before initial request
  - Validation requires different user than data submitter
```

### ReportingPeriodAggregate

```
Stream type: ReportingPeriod
Commands: OpenReportingPeriod, LockPeriod, UnlockPeriod, DraftDisclosure, ReviewDisclosure, PublishDisclosure
Events: ReportingPeriodOpened, ReportingPeriodLocked, DisclosureResponseDrafted, DisclosureResponsePublished
Invariants:
  - Period dates must not overlap with existing periods for the same tenant
  - Lock requires all mandatory disclosures to be in 'approved' status
  - Unlock requires approver-level authorization and reason
  - Cannot draft disclosures for a period that is not open or unlocked
```

---

## Pros

1. **Audit trail is the architecture, not a bolt-on.** Every state change is an immutable event with a timestamp, actor, and causation chain. Auditors can replay any aggregate to any point in time. This directly addresses the CSRD/ESRS assurance requirement for data lineage from source document to published disclosure, which is the hardest requirement for traditional CRUD systems to satisfy retroactively.

2. **Emission factor restatements are first-class citizens.** When the IPCC publishes updated GWP values or a grid emission factor changes, the system can replay the event stream with new factors and emit `EmissionsRecalculated` events showing the delta. The original calculation is preserved alongside the restatement -- precisely what auditors need for year-over-year comparisons.

3. **Multi-framework reporting via independent projections.** Each reporting framework (GRI, CSRD, ISSB, CDP) gets its own projector reading from the same event stream. Adding a new framework means adding a new projector -- no schema migration, no data duplication. This is a significant advantage for a platform whose core value proposition is single-dataset, multi-framework reporting.

4. **Temporal queries without complexity.** Questions like "What was our Scope 3 total as of Q2 close?" or "Show me the state before the October data correction" are trivially answered by replaying events up to a timestamp. In a traditional CRUD system, these require either a separate temporal table pattern or extensive audit log reconstruction.

5. **Natural domain alignment.** ESG reporting is inherently event-driven: data is recorded, validated, corrected, approved, and published in a well-defined workflow. The event types map directly to business domain concepts that sustainability managers already understand.

6. **Independent read/write scaling.** The write path (event store) and read path (projections) scale independently. During reporting season when dashboards are under heavy load, read replicas can be added for projections without affecting the write path's consistency guarantees.

---

## Cons

1. **Significantly higher implementation complexity.** Event sourcing requires building aggregate root logic, event handlers, projectors, snapshot management, and idempotent event processing. The development team needs experience with event-sourced systems. This is a substantial investment for an early-stage platform that could ship faster with a CRUD approach.

2. **Eventual consistency in read models.** Projections are updated asynchronously after events are stored. There is a lag (typically milliseconds to seconds) between a write and its visibility in dashboards. For ESG reporting this is generally acceptable, but users may be confused when they submit data and the dashboard does not update instantly.

3. **Event schema evolution is hard.** As the domain evolves, event shapes change. An `EmissionsCalculated` event from 2025 may lack fields added in 2027. The system needs upcasting/versioning logic to handle old events alongside new ones. This is manageable but adds ongoing maintenance burden.

4. **Aggregate replay performance for long-lived streams.** An emission source with 5 years of daily activity data entries could have thousands of events. Replaying from scratch is slow; snapshots mitigate this but add another system to maintain. The `event_snapshots` table helps but requires a periodic snapshot creation strategy.

5. **Debugging is harder.** When something goes wrong, developers must reason about event sequences rather than inspecting a single database row. The learning curve is steep, and tooling for event-sourced debugging is less mature than for traditional CRUD systems.

6. **Storage amplification.** The event store grows monotonically (events are never deleted). Read model tables duplicate data from the event store. A single activity data entry generates at minimum `ActivityDataRecorded` + `EmissionsCalculated` events and updates 3-4 projections. Total storage is roughly 3-5x that of an equivalent CRUD system.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event Store | PostgreSQL 16+ (event_store table with partitioning) or EventStoreDB for dedicated event store |
| Event Bus | PostgreSQL LISTEN/NOTIFY for simple setups; Apache Kafka or NATS JetStream for high-throughput |
| Projection Database | PostgreSQL for relational projections; optionally Redis for hot cache projections |
| Framework | Marten (C#/.NET) or Axon Framework (Java/Kotlin) for full ES/CQRS; custom TypeScript with pg for Node.js |
| Serialization | JSON (JSONB in PostgreSQL) for events; Avro or Protobuf if using Kafka |
| Snapshot Strategy | Time-based (every 100 events per stream) or on-demand for aggregates with high event counts |
| CDC (Change Data Capture) | Debezium for PostgreSQL WAL-based event publishing to Kafka |
| Monitoring | OpenTelemetry tracing across command → event → projector pipeline |
| Testing | Event-driven BDD: Given [events] → When [command] → Then [new events] |

---

## Migration and Scaling Considerations

### Initial Data Migration
- Historical emissions data (from spreadsheets or legacy systems) is imported as synthetic events: `ActivityDataRecorded` and `EmissionsCalculated` events with `data_origin: 'historical_import'` in metadata.
- Each imported record generates a pair of events, preserving the full audit trail even for pre-platform data.
- A one-time `HistoricalDataImported` meta-event marks the boundary between imported and live data.

### Scaling Strategy
- **Event store partitioning:** Partition `event_store` by `created_at` (monthly). Archive old partitions to cold storage after retention period (7-10 years for most ESG regulations).
- **Projection parallelism:** Each projector runs independently and tracks its own `last_event_id`. Slow projectors do not block fast ones. Projectors can be restarted from any checkpoint.
- **Projection rebuild:** If a projection needs to change (e.g., adding a new column to `rm_emissions_summary`), truncate the projection table and replay from the event store. This is a powerful capability that traditional schemas cannot match.
- **Multi-tenant isolation:** Events are tagged with `tenant_id` in both the event payload and metadata. Projections are filtered by tenant. For very large deployments, the event store can be partitioned by tenant.
- **Read scaling:** Projection tables can be replicated to read replicas. For high-traffic dashboards, hot projections can be cached in Redis with TTLs aligned to event processing lag.

### Event Store Growth Estimation
For a mid-sized deployment (50 tenants, ~500 facilities, ~5,000 emission sources):
- Activity data events: ~50,000/month across all tenants
- Emission calculation events: ~50,000/month
- Supplier engagement events: ~5,000/month during data collection
- Audit/workflow events: ~10,000/month
- Total: ~115,000 events/month, ~1.4M events/year
- At ~2KB average event size: ~2.8GB/year for the event store
- With projections: ~8-14GB/year total storage
- Comfortable on a single PostgreSQL instance for 5+ years
