# Data Model Suggestion 4: Graph Database + Time-Series Hybrid (Neo4j + TimescaleDB)

> Project: Sustainability & ESG Reporting (457)
> Model: Graph database (Neo4j) for supply chain and framework relationships + TimescaleDB for emissions time-series data
> Generated: 2026-05-25

---

## Design Philosophy

ESG reporting has two deeply embedded data problems that relational databases handle poorly:

1. **Supply chain relationship graphs.** Scope 3 emissions -- which represent 75-95% of most organizations' total carbon footprint -- flow through complex, multi-tier supplier networks. A company's Scope 1 emissions are its customer's Scope 3 Category 1. A supplier's supplier's emissions cascade upward through the value chain. These relationships are inherently graph-structured: entities connected by edges that carry attribution weights, temporal validity, and emissions data. Relational models require expensive recursive CTEs or self-joins to traverse these networks; graph databases navigate them natively.

2. **Temporal emissions data at varying granularities.** Emissions data arrives at different cadences: IoT meter readings every 15 minutes, utility bills monthly, supplier surveys annually, fleet GPS data daily. Tracking year-over-year trends, detecting anomalies, computing rolling averages, and supporting base-year recalculation all require time-series operations that relational databases perform adequately but specialized time-series engines optimize by orders of magnitude.

This model uses Neo4j for the organizational, supply chain, and framework relationship graph, and TimescaleDB (PostgreSQL extension) for all time-stamped measurement data. A thin PostgreSQL layer handles authentication, workflow state, and document management.

---

## Graph Model (Neo4j)

### Node Types

```cypher
// === ORGANIZATIONAL NODES ===

// Tenant (top-level reporting entity)
(:Tenant {
    id: UUID,
    name: String,
    legal_name: String,
    industry_code: String,  // NACE Rev 2
    country_code: String,   // ISO 3166-1
    boundary_approach: String,  // operational_control | financial_control | equity_share
    base_year: Integer,
    fiscal_year_start: Integer
})

// Organizational unit (subsidiary, division, JV)
(:OrgUnit {
    id: UUID,
    tenant_id: UUID,
    name: String,
    unit_type: String,
    ownership_pct: Float,
    country_code: String,
    is_active: Boolean
})

// Physical facility
(:Facility {
    id: UUID,
    tenant_id: UUID,
    name: String,
    facility_type: String,
    country_code: String,
    latitude: Float,
    longitude: Float,
    gross_floor_area_m2: Float,
    employee_count: Integer,
    is_active: Boolean
})

// === SUPPLY CHAIN NODES ===

// Supplier entity (external to the tenant's boundary)
(:Supplier {
    id: UUID,
    tenant_id: UUID,
    name: String,
    country_code: String,
    industry_code: String,
    tier: Integer,
    annual_spend: Float,
    spend_currency: String,
    risk_score: Float,
    ecovadis_score: Integer,
    sbti_committed: Boolean,
    certifications: List<String>
})

// Product or service in the value chain
(:Product {
    id: UUID,
    name: String,
    sku: String,
    category: String,
    unit_weight_kg: Float,
    pcf_kg_co2e: Float  // product carbon footprint per unit
})

// === EMISSIONS STRUCTURE NODES ===

// Emission source classification
(:EmissionSource {
    id: UUID,
    tenant_id: UUID,
    scope: Integer,          // 1, 2, 3
    scope3_category: Integer, // 1-15 if scope=3
    source_type: String,
    name: String,
    is_active: Boolean
})

// Emission factor (versioned)
(:EmissionFactor {
    id: UUID,
    name: String,
    category: String,
    region: String,
    gas_type: String,
    factor_value: Float,
    unit_numerator: String,
    unit_denominator: String,
    gwp_source: String,
    valid_from: Date,
    valid_to: Date,
    data_quality_score: Integer
})

// Emission factor library
(:FactorLibrary {
    id: UUID,
    name: String,
    source: String,      // IPCC, IEA, EPA, ecoinvent
    version: String,
    publication_date: Date
})

// === REPORTING FRAMEWORK NODES ===

// Reporting framework
(:Framework {
    id: UUID,
    code: String,        // GRI, CSRD_ESRS, ISSB_S2, CDP, SB253
    name: String,
    version: String,
    governing_body: String
})

// Individual disclosure requirement
(:Disclosure {
    id: UUID,
    framework_code: String,
    disclosure_code: String,  // GRI 305-1, ESRS E1-6, IFRS S2.29
    name: String,
    topic: String,            // E, S, G
    data_type: String,        // quantitative, narrative, boolean
    is_mandatory: Boolean
})

// Materiality topic
(:MaterialityTopic {
    id: UUID,
    name: String,
    esrs_code: String,       // E1, E2, S1, G1, etc.
    description: String
})

// === INDUSTRY CLASSIFICATION ===

(:Industry {
    code: String,
    name: String,
    classification_system: String  // NACE, NAICS, ANZSIC
})
```

### Relationship Types

```cypher
// === ORGANIZATIONAL RELATIONSHIPS ===

// Org hierarchy
(:Tenant)-[:OWNS {ownership_pct: Float, consolidation_method: String}]->(:OrgUnit)
(:OrgUnit)-[:OWNS {ownership_pct: Float}]->(:OrgUnit)
(:OrgUnit)-[:OPERATES]->(:Facility)

// === SUPPLY CHAIN RELATIONSHIPS ===
// This is where the graph model excels: multi-tier supplier networks
// with emissions attribution flowing through edges

(:Tenant)-[:PROCURES_FROM {
    annual_spend: Float,
    spend_currency: String,
    scope3_category: Integer,  // typically 1 (purchased goods) or 2 (capital goods)
    contract_start: Date,
    contract_end: Date,
    procurement_category: String
}]->(:Supplier)

(:Supplier)-[:SUPPLIES_TO {
    product_category: String,
    volume: Float,
    unit: String
}]->(:Tenant)

(:Supplier)-[:SUB_SUPPLIES_FROM {
    tier: Integer,
    annual_spend: Float,
    relationship_type: String  // direct, broker, consortium
}]->(:Supplier)

// Product flows through supply chain
(:Supplier)-[:PRODUCES]->(:Product)
(:Product)-[:USED_BY]->(:Facility)
(:Product)-[:COMPONENT_OF]->(:Product)

// Emission attribution through supply chain
(:Supplier)-[:EMITS {
    scope: Integer,
    co2e_tonnes: Float,
    reporting_year: Integer,
    methodology: String,
    data_quality: Integer,
    verified: Boolean
}]->(:EmissionSource)

// === EMISSION STRUCTURE RELATIONSHIPS ===

(:Facility)-[:HAS_SOURCE]->(:EmissionSource)
(:EmissionSource)-[:USES_FACTOR {valid_from: Date, valid_to: Date}]->(:EmissionFactor)
(:FactorLibrary)-[:CONTAINS]->(:EmissionFactor)
(:EmissionFactor)-[:SUPERSEDES]->(:EmissionFactor)

// === FRAMEWORK RELATIONSHIPS ===
// Cross-framework equivalence mapping as graph edges

(:Framework)-[:DEFINES]->(:Disclosure)
(:Disclosure)-[:EQUIVALENT_TO {
    strength: String,  // exact, partial, related
    notes: String
}]->(:Disclosure)

(:Disclosure)-[:REQUIRES_DATA_FROM]->(:EmissionSource)
(:Disclosure)-[:MAPS_TO_METRIC {field: String, aggregation: String}]->(:MaterialityTopic)

(:Tenant)-[:MUST_REPORT_UNDER {
    is_mandatory: Boolean,
    effective_year: Integer
}]->(:Framework)

// Materiality relationships
(:Tenant)-[:MATERIAL_TOPIC {
    financial_materiality: Float,    // 0-1 scale
    impact_materiality: Float,       // 0-1 scale
    is_material: Boolean,
    assessment_date: Date
}]->(:MaterialityTopic)

// Industry classification
(:Tenant)-[:CLASSIFIED_AS]->(:Industry)
(:Industry)-[:SASB_STANDARD]->(:Framework)
```

### Key Graph Queries

```cypher
// 1. Trace full Scope 3 supply chain emissions (multi-tier)
// "What are my total upstream emissions including Tier 2 and Tier 3 suppliers?"
MATCH path = (t:Tenant {id: $tenantId})-[:PROCURES_FROM*1..3]->(s:Supplier)
MATCH (s)-[e:EMITS]->(es:EmissionSource)
WHERE e.reporting_year = $year
RETURN s.name, s.tier, 
       length(path) AS chain_depth,
       SUM(e.co2e_tonnes) AS supplier_emissions,
       e.data_quality AS quality
ORDER BY supplier_emissions DESC

// 2. Identify emission hotspots in the supply chain
// "Which supplier relationships contribute most to my Scope 3?"
MATCH (t:Tenant {id: $tenantId})-[p:PROCURES_FROM]->(s:Supplier)-[e:EMITS]->(es:EmissionSource)
WHERE e.reporting_year = $year AND es.scope = 3
WITH s, p, SUM(e.co2e_tonnes) AS total_emissions
RETURN s.name, s.country_code, p.annual_spend, 
       total_emissions,
       total_emissions / p.annual_spend AS emission_intensity_per_spend
ORDER BY total_emissions DESC
LIMIT 20

// 3. Framework crosswalk: find all disclosures equivalent to GRI 305-1
// "If I've prepared GRI 305-1, which other framework disclosures can I auto-fill?"
MATCH (d:Disclosure {disclosure_code: 'GRI 305-1'})-[eq:EQUIVALENT_TO*1..2]-(related:Disclosure)
RETURN related.framework_code, related.disclosure_code, related.name,
       eq[0].strength AS mapping_strength

// 4. Double materiality assessment visualization
// "Show me the materiality matrix for this tenant"
MATCH (t:Tenant {id: $tenantId})-[m:MATERIAL_TOPIC]->(mt:MaterialityTopic)
WHERE m.assessment_date >= date($assessmentDate)
RETURN mt.name, mt.esrs_code,
       m.financial_materiality, m.impact_materiality,
       m.is_material
ORDER BY m.financial_materiality * m.impact_materiality DESC

// 5. Supply chain risk propagation
// "If Supplier X has a compliance issue, which of my products are affected?"
MATCH (s:Supplier {id: $supplierId})-[:PRODUCES]->(p:Product)-[:COMPONENT_OF*0..3]->(final:Product)
MATCH (final)-[:USED_BY]->(f:Facility)<-[:OPERATES]-(ou:OrgUnit)<-[:OWNS]-(t:Tenant)
RETURN DISTINCT final.name, f.name, ou.name, t.name
```

---

## Time-Series Model (TimescaleDB)

TimescaleDB extends PostgreSQL with hypertable partitioning, continuous aggregates, and compression optimized for time-stamped data. All temporal emissions, energy, and activity measurements go here.

### Hypertables

```sql
-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Activity measurements: the raw time-stamped data from all sources
CREATE TABLE activity_measurements (
    time                TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    facility_id         UUID NOT NULL,
    emission_source_id  UUID NOT NULL,
    scope               SMALLINT NOT NULL,
    scope3_category     SMALLINT,
    source_type         VARCHAR(50) NOT NULL,
    -- Measurement values
    quantity            NUMERIC(20,6) NOT NULL,
    unit                VARCHAR(30) NOT NULL,
    -- Calculated emissions (populated by GHG engine after factor application)
    co2e_tonnes         NUMERIC(16,6),
    co2_tonnes          NUMERIC(16,6),
    ch4_tonnes          NUMERIC(16,6),
    n2o_tonnes          NUMERIC(16,6),
    -- Metadata
    data_origin         VARCHAR(30) NOT NULL,
    data_quality_score  SMALLINT,
    emission_factor_id  UUID,
    calculation_method  VARCHAR(50),
    source_document_id  UUID,
    created_by          UUID,
    -- Scope 2 specifics
    scope2_method       VARCHAR(20),
    -- Tags for flexible filtering
    tags                JSONB DEFAULT '{}'
);

-- Convert to hypertable (partitioned by time, chunked weekly)
SELECT create_hypertable('activity_measurements', by_range('time', INTERVAL '1 week'));

-- Add compression policy (compress chunks older than 3 months)
ALTER TABLE activity_measurements SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, facility_id, scope',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('activity_measurements', INTERVAL '3 months');

-- Retention policy: move data older than 10 years to cold storage
-- SELECT add_retention_policy('activity_measurements', INTERVAL '10 years');

-- Indexes for common query patterns
CREATE INDEX idx_am_tenant_time ON activity_measurements(tenant_id, time DESC);
CREATE INDEX idx_am_facility_time ON activity_measurements(facility_id, time DESC);
CREATE INDEX idx_am_scope_time ON activity_measurements(scope, scope3_category, time DESC);
CREATE INDEX idx_am_source ON activity_measurements(emission_source_id, time DESC);

-- Energy consumption time series (sub-hourly IoT/smart meter data)
CREATE TABLE energy_readings (
    time                TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    facility_id         UUID NOT NULL,
    meter_id            VARCHAR(100) NOT NULL,
    energy_type         VARCHAR(30) NOT NULL,  -- electricity, natural_gas, steam, chilled_water
    reading_kwh         NUMERIC(16,6) NOT NULL,
    demand_kw           NUMERIC(12,4),
    power_factor        NUMERIC(4,3),
    voltage             NUMERIC(8,2),
    -- Grid emission factor at time of reading (for real-time carbon intensity)
    grid_carbon_intensity_gco2_kwh NUMERIC(10,4),
    co2e_kg             NUMERIC(16,6),         -- real-time emissions calculation
    is_renewable        BOOLEAN DEFAULT FALSE,
    tags                JSONB DEFAULT '{}'
);

SELECT create_hypertable('energy_readings', by_range('time', INTERVAL '1 day'));

ALTER TABLE energy_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, facility_id, meter_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('energy_readings', INTERVAL '1 month');

CREATE INDEX idx_er_tenant_time ON energy_readings(tenant_id, time DESC);
CREATE INDEX idx_er_facility_meter ON energy_readings(facility_id, meter_id, time DESC);

-- Supplier emissions time series
CREATE TABLE supplier_emissions_ts (
    time                TIMESTAMPTZ NOT NULL,     -- reporting period end date
    tenant_id           UUID NOT NULL,
    supplier_id         UUID NOT NULL,
    scope1_co2e_tonnes  NUMERIC(16,6),
    scope2_co2e_tonnes  NUMERIC(16,6),
    scope3_co2e_tonnes  NUMERIC(16,6),
    total_co2e_tonnes   NUMERIC(16,6),
    data_quality_score  SMALLINT,
    methodology         VARCHAR(100),
    is_verified         BOOLEAN DEFAULT FALSE,
    response_id         UUID,
    tags                JSONB DEFAULT '{}'
);

SELECT create_hypertable('supplier_emissions_ts', by_range('time', INTERVAL '1 year'));

CREATE INDEX idx_set_tenant_time ON supplier_emissions_ts(tenant_id, time DESC);
CREATE INDEX idx_set_supplier ON supplier_emissions_ts(supplier_id, time DESC);

-- Target tracking time series
CREATE TABLE target_progress (
    time                TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    target_id           UUID NOT NULL,
    scope_coverage      VARCHAR(30) NOT NULL,
    actual_co2e_tonnes  NUMERIC(16,6) NOT NULL,
    target_co2e_tonnes  NUMERIC(16,6) NOT NULL,
    baseline_co2e_tonnes NUMERIC(16,6) NOT NULL,
    reduction_pct       NUMERIC(7,3),
    on_track            BOOLEAN,
    tags                JSONB DEFAULT '{}'
);

SELECT create_hypertable('target_progress', by_range('time', INTERVAL '1 quarter'));

CREATE INDEX idx_tp_tenant ON target_progress(tenant_id, time DESC);
CREATE INDEX idx_tp_target ON target_progress(target_id, time DESC);
```

### Continuous Aggregates (Materialised Views with Automatic Refresh)

```sql
-- Monthly emissions aggregate (auto-refreshed by TimescaleDB)
CREATE MATERIALIZED VIEW cagg_monthly_emissions
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', time) AS month,
    tenant_id,
    facility_id,
    scope,
    scope3_category,
    SUM(co2e_tonnes) AS total_co2e_tonnes,
    SUM(co2_tonnes) AS total_co2_tonnes,
    SUM(ch4_tonnes) AS total_ch4_tonnes,
    SUM(n2o_tonnes) AS total_n2o_tonnes,
    COUNT(*) AS measurement_count,
    AVG(data_quality_score) AS avg_data_quality
FROM activity_measurements
GROUP BY month, tenant_id, facility_id, scope, scope3_category;

-- Refresh policy: update every hour, looking back 2 days for late-arriving data
SELECT add_continuous_aggregate_policy('cagg_monthly_emissions',
    start_offset => INTERVAL '2 days',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Annual emissions aggregate (for year-over-year comparison)
CREATE MATERIALIZED VIEW cagg_annual_emissions
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 year', time) AS year,
    tenant_id,
    scope,
    scope3_category,
    SUM(co2e_tonnes) AS total_co2e_tonnes,
    COUNT(*) AS measurement_count,
    AVG(data_quality_score) AS avg_data_quality
FROM activity_measurements
GROUP BY year, tenant_id, scope, scope3_category;

SELECT add_continuous_aggregate_policy('cagg_annual_emissions',
    start_offset => INTERVAL '30 days',
    end_offset   => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- Daily energy consumption with real-time carbon intensity
CREATE MATERIALIZED VIEW cagg_daily_energy
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS day,
    tenant_id,
    facility_id,
    meter_id,
    energy_type,
    SUM(reading_kwh) AS total_kwh,
    MAX(demand_kw) AS peak_demand_kw,
    AVG(grid_carbon_intensity_gco2_kwh) AS avg_carbon_intensity,
    SUM(co2e_kg) AS total_co2e_kg,
    SUM(CASE WHEN is_renewable THEN reading_kwh ELSE 0 END) AS renewable_kwh
FROM energy_readings
GROUP BY day, tenant_id, facility_id, meter_id, energy_type;

SELECT add_continuous_aggregate_policy('cagg_daily_energy',
    start_offset => INTERVAL '3 hours',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Supplier emissions year-over-year comparison
CREATE MATERIALIZED VIEW cagg_supplier_yoy
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 year', time) AS year,
    tenant_id,
    supplier_id,
    SUM(total_co2e_tonnes) AS annual_co2e_tonnes,
    AVG(data_quality_score) AS avg_quality,
    bool_and(is_verified) AS all_verified
FROM supplier_emissions_ts
GROUP BY year, tenant_id, supplier_id;

SELECT add_continuous_aggregate_policy('cagg_supplier_yoy',
    start_offset => INTERVAL '90 days',
    end_offset   => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);
```

---

## PostgreSQL Coordination Layer

A standard PostgreSQL database handles non-time-series, non-graph data: user authentication, workflow state, document metadata, and system configuration.

```sql
-- Users, authentication, and RBAC
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    email               VARCHAR(255) NOT NULL UNIQUE,
    full_name           VARCHAR(255) NOT NULL,
    role                VARCHAR(30) NOT NULL,
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Reporting periods
CREATE TABLE reporting_periods (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    name                VARCHAR(100) NOT NULL,
    period_type         VARCHAR(20) NOT NULL,
    start_date          DATE NOT NULL,
    end_date            DATE NOT NULL,
    status              VARCHAR(20) DEFAULT 'draft',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

-- Approval workflows
CREATE TABLE approval_workflows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    entity_type         VARCHAR(50) NOT NULL,
    entity_id           UUID NOT NULL,
    status              VARCHAR(20) DEFAULT 'pending',
    submitted_by        UUID NOT NULL REFERENCES users(id),
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at        TIMESTAMPTZ
);

CREATE TABLE approval_steps (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id         UUID NOT NULL REFERENCES approval_workflows(id),
    step_number         SMALLINT NOT NULL,
    reviewer_id         UUID NOT NULL REFERENCES users(id),
    decision            VARCHAR(20),
    comments            TEXT,
    decided_at          TIMESTAMPTZ
);

-- Disclosure responses (final submissions for each framework)
CREATE TABLE disclosure_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    framework_code      VARCHAR(30) NOT NULL,
    disclosure_code     VARCHAR(50) NOT NULL,
    response_data       JSONB NOT NULL,
    status              VARCHAR(20) DEFAULT 'draft',
    approved_by         UUID,
    approved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, reporting_period_id, framework_code, disclosure_code)
);

-- Source documents
CREATE TABLE source_documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    document_type       VARCHAR(30) NOT NULL,
    file_name           VARCHAR(500) NOT NULL,
    file_path           TEXT NOT NULL,
    checksum_sha256     CHAR(64),
    extracted_data      JSONB,
    uploaded_by         UUID NOT NULL,
    uploaded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Supplier data request workflow state
CREATE TABLE supplier_data_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    supplier_id         UUID NOT NULL,  -- references Neo4j Supplier node
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    status              VARCHAR(20) DEFAULT 'draft',
    sent_at             TIMESTAMPTZ,
    due_date            DATE,
    reminder_count      INTEGER DEFAULT 0,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Audit log
CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    table_name          VARCHAR(100) NOT NULL,
    record_id           UUID NOT NULL,
    action              VARCHAR(10) NOT NULL,
    changes             JSONB NOT NULL,
    changed_by          UUID NOT NULL,
    changed_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Application Layer                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐ │
│  │  REST API     │  │ Supplier     │  │ Report       │  │ AI/ML         │ │
│  │  Gateway      │  │ Portal       │  │ Generator    │  │ Pipeline      │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘ │
└─────────┼──────────────────┼──────────────────┼──────────────────┼─────────┘
          │                  │                  │                  │
    ┌─────▼──────────────────▼──────────────────▼──────────────────▼─────┐
    │                     Service Layer                                   │
    │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
    │  │ Graph       │  │ Time-Series │  │ Workflow    │                │
    │  │ Service     │  │ Service     │  │ Service     │                │
    │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                │
    └─────────┼────────────────┼────────────────┼───────────────────────┘
              │                │                │
    ┌─────────▼─────┐  ┌──────▼──────┐  ┌──────▼──────┐
    │    Neo4j      │  │ TimescaleDB │  │ PostgreSQL  │
    │               │  │             │  │             │
    │ • Org graph   │  │ • Activity  │  │ • Users     │
    │ • Supply      │  │   measure-  │  │ • Workflows │
    │   chain       │  │   ments     │  │ • Docs      │
    │ • Framework   │  │ • Energy    │  │ • Disclosure│
    │   crosswalks  │  │   readings  │  │   responses │
    │ • Materiality │  │ • Supplier  │  │ • Audit log │
    │ • EF graph    │  │   emissions │  │             │
    │               │  │ • Target    │  │             │
    │               │  │   progress  │  │             │
    └───────────────┘  └─────────────┘  └─────────────┘
```

### Data Flow for Key Operations

```
1. Record Activity Data:
   API → Workflow Service (validate, PostgreSQL) 
       → Time-Series Service (store in TimescaleDB)
       → Graph Service (update EmissionSource node relationships)

2. Generate Multi-Framework Report:
   API → Graph Service (traverse Framework→Disclosure→EmissionSource graph)
       → Time-Series Service (aggregate emissions for period)
       → Report Generator (combine graph structure with time-series data)
       → Workflow Service (store disclosure_responses in PostgreSQL)

3. Scope 3 Supply Chain Analysis:
   API → Graph Service (traverse multi-tier supplier graph in Neo4j)
       → Time-Series Service (pull supplier_emissions_ts for trend)
       → AI Pipeline (identify hotspots, suggest reductions)

4. Supplier Data Collection:
   Supplier Portal → Workflow Service (PostgreSQL: request status)
                   → Time-Series Service (TimescaleDB: store emissions data)
                   → Graph Service (Neo4j: update Supplier node, add EMITS relationship)
```

---

## Anomaly Detection with Time-Series Functions

```sql
-- Detect anomalous emissions readings using TimescaleDB statistical functions
-- Flag any monthly reading that deviates more than 2 standard deviations from the trailing 12-month average

WITH monthly_stats AS (
    SELECT
        month,
        tenant_id,
        facility_id,
        scope,
        total_co2e_tonnes,
        AVG(total_co2e_tonnes) OVER (
            PARTITION BY tenant_id, facility_id, scope
            ORDER BY month
            ROWS BETWEEN 12 PRECEDING AND 1 PRECEDING
        ) AS trailing_avg,
        STDDEV(total_co2e_tonnes) OVER (
            PARTITION BY tenant_id, facility_id, scope
            ORDER BY month
            ROWS BETWEEN 12 PRECEDING AND 1 PRECEDING
        ) AS trailing_stddev
    FROM cagg_monthly_emissions
    WHERE tenant_id = $1
)
SELECT month, facility_id, scope, total_co2e_tonnes,
       trailing_avg,
       (total_co2e_tonnes - trailing_avg) / NULLIF(trailing_stddev, 0) AS z_score,
       CASE
           WHEN ABS((total_co2e_tonnes - trailing_avg) / NULLIF(trailing_stddev, 0)) > 2 THEN 'ANOMALY'
           WHEN ABS((total_co2e_tonnes - trailing_avg) / NULLIF(trailing_stddev, 0)) > 1.5 THEN 'WARNING'
           ELSE 'NORMAL'
       END AS status
FROM monthly_stats
WHERE trailing_avg IS NOT NULL
ORDER BY month DESC;

-- Year-over-year emissions comparison with trend detection
SELECT
    year,
    scope,
    total_co2e_tonnes,
    LAG(total_co2e_tonnes) OVER (PARTITION BY scope ORDER BY year) AS prev_year,
    ROUND(
        (total_co2e_tonnes - LAG(total_co2e_tonnes) OVER (PARTITION BY scope ORDER BY year))
        / NULLIF(LAG(total_co2e_tonnes) OVER (PARTITION BY scope ORDER BY year), 0) * 100,
        2
    ) AS yoy_change_pct
FROM cagg_annual_emissions
WHERE tenant_id = $1
ORDER BY year, scope;

-- Real-time carbon intensity tracking: identify peak vs off-peak emissions
SELECT
    time_bucket('1 hour', time) AS hour,
    facility_id,
    SUM(reading_kwh) AS total_kwh,
    AVG(grid_carbon_intensity_gco2_kwh) AS avg_intensity,
    SUM(co2e_kg) AS total_co2e_kg,
    CASE
        WHEN AVG(grid_carbon_intensity_gco2_kwh) > 400 THEN 'high_carbon'
        WHEN AVG(grid_carbon_intensity_gco2_kwh) > 200 THEN 'medium_carbon'
        ELSE 'low_carbon'
    END AS carbon_band
FROM energy_readings
WHERE tenant_id = $1 AND time >= NOW() - INTERVAL '7 days'
GROUP BY hour, facility_id
ORDER BY hour;
```

---

## Pros

1. **Supply chain traversal is native and fast.** Neo4j traverses multi-tier supplier relationships in milliseconds, where relational databases would need recursive CTEs or multiple self-joins. For Scope 3 analysis -- the dominant emissions category and the hardest data problem -- this is transformative. Queries like "show me the full upstream emissions cascade for Product X through 3 tiers of suppliers" are natural graph traversals, not SQL gymnastics.

2. **Framework crosswalk as a connected graph.** The relationships between GRI 305-1, ESRS E1-6, IFRS S2.29, and CDP C6.1 form a natural equivalence graph. Adding a new framework means adding new Disclosure nodes and EQUIVALENT_TO edges -- no schema migration, no new join tables. The crosswalk graph can be versioned and queried to auto-populate disclosure responses.

3. **Time-series operations for emissions trends.** TimescaleDB's continuous aggregates, compression, and retention policies are purpose-built for the temporal patterns in ESG data. Monthly/quarterly/annual emissions aggregation, year-over-year comparison, and anomaly detection run orders of magnitude faster than equivalent PostgreSQL materialized views, especially as data volumes grow beyond 5 years.

4. **Real-time carbon intensity tracking.** The `energy_readings` hypertable supports sub-hourly IoT/smart meter data with automatic compression and aggregation. Organizations can track real-time carbon intensity of their electricity consumption and identify opportunities to shift load to low-carbon periods. This is a differentiating feature that most ESG platforms lack.

5. **Materiality assessment as a graph problem.** Double materiality (required by CSRD) maps naturally to a graph: topics connected to industries, connected to tenants, with financial_materiality and impact_materiality scores on edges. Graph algorithms (centrality, community detection) can assist with automated materiality assessment, which is identified as an underserved area in the features survey.

6. **Natural data model for emissions factor lineage.** Emission factors form a versioned graph: Factor A supersedes Factor B, both belong to Library X. When a new library version is published, graph traversal identifies all affected calculations instantly via the SUPERSEDES and USES_FACTOR relationships.

---

## Cons

1. **Operational complexity of three database systems.** Running Neo4j, TimescaleDB, and PostgreSQL in production requires three sets of backup procedures, monitoring dashboards, failover configurations, and operational runbooks. This is the most significant drawback and makes this model unsuitable for small teams or early-stage deployments.

2. **Cross-database consistency is hard.** When a supplier's emissions data must be stored in TimescaleDB (the measurement), Neo4j (the EMITS relationship), and PostgreSQL (the workflow status), maintaining consistency across three systems requires distributed transaction coordination or eventual consistency with careful compensation logic.

3. **Neo4j licensing cost.** Neo4j Community Edition is free but lacks features critical for production: online backup, clustering, role-based access control, and monitoring. Neo4j Enterprise requires a commercial licence. Neo4j AuraDB (managed cloud) is an alternative but adds SaaS dependency. Note: Memgraph is an open-source alternative but with a smaller ecosystem.

4. **Developer skill requirements.** The team must be proficient in Cypher (Neo4j's query language), TimescaleDB-specific SQL extensions, and standard PostgreSQL. Finding developers comfortable with all three is harder than finding PostgreSQL-only developers.

5. **Graph model is overkill for simple deployments.** A mid-market company with 50 suppliers and straightforward Scope 1/2 reporting does not need a graph database. The graph model's advantages emerge only with complex multi-tier supply chains (200+ suppliers, Scope 3 across multiple categories). For simpler deployments, the operational overhead is not justified.

6. **TimescaleDB compression trade-offs.** Compressed chunks in TimescaleDB are read-only. If historical emissions data needs correction (restatement), the affected chunk must be decompressed first, which temporarily increases storage and can be slow for very large chunks. ESG restatements, while infrequent, do occur.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Graph Database | Neo4j 5.x (Enterprise or AuraDB for production) or Memgraph as open-source alternative |
| Time-Series Database | TimescaleDB 2.x (PostgreSQL extension) |
| Coordination Database | PostgreSQL 16+ (standard, can share instance with TimescaleDB) |
| Graph ORM | Neo4j JavaScript Driver or neomodel (Python) |
| Time-Series Client | Standard PostgreSQL drivers (psycopg2, node-postgres) |
| Message Bus | Apache Kafka or NATS JetStream for cross-database event coordination |
| API Gateway | GraphQL Federation (Apollo) to unify queries across all three databases |
| Monitoring | Neo4j Ops Manager + Prometheus/Grafana for TimescaleDB + pg_stat_statements |
| Backup | Neo4j Enterprise backup + pgBackRest for PostgreSQL/TimescaleDB |

---

## Migration and Scaling Considerations

### Migration Strategy
- **Phase 1 (MVP):** Start with TimescaleDB + PostgreSQL only (both run in the same PostgreSQL instance). Model supply chain relationships in relational tables with recursive CTEs. This reduces operational complexity while validating the time-series model.
- **Phase 2 (Scale):** When supply chain complexity warrants it (200+ suppliers, multi-tier Scope 3), introduce Neo4j for the graph layer. Migrate supplier relationships and framework crosswalks from relational tables to the graph. Keep TimescaleDB and PostgreSQL unchanged.
- **Phase 3 (Real-time):** Add IoT/smart meter integration feeding into the `energy_readings` hypertable. Enable real-time carbon intensity tracking with continuous aggregates.

### Scaling Path
- **Neo4j:** Neo4j AuraDB auto-scales. Self-hosted Neo4j clusters support read replicas and sharding (fabric) for very large graphs. For ESG, even the largest deployments (10,000+ supplier nodes) fit comfortably in a single Neo4j instance.
- **TimescaleDB:** Horizontal scaling via TimescaleDB multi-node (distributed hypertables). Compression achieves 90-95% storage reduction for older data. Continuous aggregates pre-compute dashboards. For most ESG deployments, a single TimescaleDB instance handles 5+ years of data with compression.
- **PostgreSQL:** Standard vertical scaling with read replicas. The coordination layer has modest data volumes.

### Data Retention and Archival
- **TimescaleDB:** Automated retention policies move data older than the regulatory retention period (typically 7-10 years for ESG) to cold storage (S3 via pg_tier) or delete it per policy.
- **Neo4j:** Graph nodes remain small; archival is rarely needed. Historical supplier relationships can be marked inactive rather than deleted.
- **PostgreSQL:** Standard archive/purge for audit logs older than retention period.

### Cross-Database Synchronization
- Use Kafka Connect with Neo4j Sink Connector and JDBC Source Connector to maintain consistency.
- Alternatively, use the Outbox Pattern: write to PostgreSQL + TimescaleDB first (in a single transaction since they share the same PostgreSQL instance), then publish events to Kafka for Neo4j updates.
- Idempotent consumers ensure at-least-once delivery is safe.
