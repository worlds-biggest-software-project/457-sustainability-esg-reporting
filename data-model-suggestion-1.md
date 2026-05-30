# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Sustainability & ESG Reporting (457)
> Model: Fully normalized relational schema (3NF) on PostgreSQL 16+
> Generated: 2026-05-25

---

## Design Philosophy

This model follows a traditional normalized relational approach, storing every ESG data point in dedicated, strongly-typed tables with explicit foreign key relationships. The design is inspired by the GHG Protocol's hierarchical structure (Organisation > Facility > Emission Source > Activity Data > Emission), the Microsoft Cloud for Sustainability Common Data Model entity structure, and the Persefoni "ledger-style" approach that maps carbon accounting to financial-grade auditability patterns.

Normalization to 3NF ensures data integrity, eliminates update anomalies, and supports the audit-trail requirements that are central to CSRD/ESRS assurance readiness. Every calculation is traceable from source document to published disclosure through explicit foreign key chains.

---

## Core Schema

### Organizational Structure

```sql
-- Multi-tenant support: each tenant is an independent reporting organization
CREATE TABLE tenants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    legal_name          VARCHAR(500),
    industry_code       VARCHAR(20),          -- NACE/NAICS/ANZSIC
    industry_standard   VARCHAR(10) DEFAULT 'NACE',
    country_code        CHAR(2) NOT NULL,     -- ISO 3166-1
    fiscal_year_start   SMALLINT DEFAULT 1,   -- month (1-12)
    boundary_approach   VARCHAR(30) NOT NULL CHECK (boundary_approach IN (
                            'operational_control', 'financial_control', 'equity_share'
                        )),
    base_year           SMALLINT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Organizational units (subsidiaries, divisions, business units)
CREATE TABLE organizational_units (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    parent_unit_id      UUID REFERENCES organizational_units(id),
    name                VARCHAR(255) NOT NULL,
    unit_type           VARCHAR(30) NOT NULL CHECK (unit_type IN (
                            'subsidiary', 'division', 'business_unit', 'joint_venture', 'associate'
                        )),
    ownership_pct       NUMERIC(5,2) CHECK (ownership_pct BETWEEN 0 AND 100),
    country_code        CHAR(2),
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_units_tenant ON organizational_units(tenant_id);
CREATE INDEX idx_org_units_parent ON organizational_units(parent_unit_id);

-- Physical facilities, sites, buildings
CREATE TABLE facilities (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    org_unit_id         UUID NOT NULL REFERENCES organizational_units(id),
    name                VARCHAR(255) NOT NULL,
    facility_type       VARCHAR(50) NOT NULL,  -- office, warehouse, factory, data_center, etc.
    address_line1       VARCHAR(255),
    address_line2       VARCHAR(255),
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    country_code        CHAR(2) NOT NULL,
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    gross_floor_area_m2 NUMERIC(12,2),
    employee_count      INTEGER,
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_facilities_tenant ON facilities(tenant_id);
CREATE INDEX idx_facilities_org_unit ON facilities(org_unit_id);
```

### Reporting Periods and Frameworks

```sql
-- Reporting periods (fiscal years, quarters for interim reporting)
CREATE TABLE reporting_periods (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    name                VARCHAR(100) NOT NULL,  -- e.g., "FY2025", "Q3 2025"
    period_type         VARCHAR(20) NOT NULL CHECK (period_type IN (
                            'annual', 'semi_annual', 'quarterly', 'custom'
                        )),
    start_date          DATE NOT NULL,
    end_date            DATE NOT NULL,
    status              VARCHAR(20) DEFAULT 'draft' CHECK (status IN (
                            'draft', 'data_collection', 'under_review',
                            'approved', 'assured', 'published'
                        )),
    locked_at           TIMESTAMPTZ,
    locked_by           UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_period_dates CHECK (end_date > start_date),
    UNIQUE(tenant_id, name)
);

-- Reporting frameworks the tenant must comply with
CREATE TABLE reporting_frameworks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code                VARCHAR(30) NOT NULL UNIQUE,  -- GRI, CSRD_ESRS, ISSB_S1, ISSB_S2, CDP, SB253, SEC_CLIMATE
    name                VARCHAR(255) NOT NULL,
    version             VARCHAR(30) NOT NULL,
    governing_body      VARCHAR(100),
    effective_date      DATE,
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Which frameworks apply to which tenant for which period
CREATE TABLE tenant_framework_applicability (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    framework_id        UUID NOT NULL REFERENCES reporting_frameworks(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    is_mandatory        BOOLEAN DEFAULT FALSE,
    is_voluntary        BOOLEAN DEFAULT FALSE,
    notes               TEXT,
    UNIQUE(tenant_id, framework_id, reporting_period_id)
);

-- Framework disclosure requirements (the individual data points each framework demands)
CREATE TABLE framework_disclosures (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id        UUID NOT NULL REFERENCES reporting_frameworks(id),
    disclosure_code     VARCHAR(50) NOT NULL,  -- e.g., "ESRS E1-6", "GRI 305-1", "IFRS S2.29"
    disclosure_name     VARCHAR(500) NOT NULL,
    topic_area          VARCHAR(50),           -- E, S, G or sub-category
    is_mandatory        BOOLEAN DEFAULT TRUE,
    data_type           VARCHAR(30),           -- quantitative, narrative, boolean
    unit_of_measure     VARCHAR(50),
    description         TEXT,
    UNIQUE(framework_id, disclosure_code)
);

-- Crosswalk mapping between equivalent disclosures across frameworks
CREATE TABLE framework_crosswalks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_disclosure_id UUID NOT NULL REFERENCES framework_disclosures(id),
    target_disclosure_id UUID NOT NULL REFERENCES framework_disclosures(id),
    mapping_strength     VARCHAR(20) CHECK (mapping_strength IN (
                            'exact', 'partial', 'related', 'supplementary'
                        )),
    notes               TEXT,
    UNIQUE(source_disclosure_id, target_disclosure_id)
);
```

### Emission Factors

```sql
-- Emission factor libraries (IPCC, IEA, EPA, ecoinvent, custom)
CREATE TABLE emission_factor_libraries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    source              VARCHAR(100) NOT NULL,  -- IPCC, IEA, US_EPA, ecoinvent, custom
    version             VARCHAR(50) NOT NULL,
    publication_date    DATE,
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    licence_type        VARCHAR(50),
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(source, version)
);

-- Individual emission factors
CREATE TABLE emission_factors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id          UUID NOT NULL REFERENCES emission_factor_libraries(id),
    factor_code         VARCHAR(100),
    name                VARCHAR(500) NOT NULL,
    category            VARCHAR(100),           -- energy, transport, materials, waste, etc.
    subcategory         VARCHAR(100),
    region              VARCHAR(100),            -- country/grid region this factor applies to
    gas_type            VARCHAR(20) NOT NULL DEFAULT 'CO2e' CHECK (gas_type IN (
                            'CO2', 'CH4', 'N2O', 'HFCs', 'PFCs', 'SF6', 'NF3', 'CO2e'
                        )),
    factor_value        NUMERIC(20,10) NOT NULL, -- kg CO2e per unit
    unit_numerator      VARCHAR(30) NOT NULL,    -- kg CO2e
    unit_denominator    VARCHAR(50) NOT NULL,    -- kWh, litre, km, kg, USD, etc.
    gwp_source          VARCHAR(50),             -- AR5, AR6
    uncertainty_pct     NUMERIC(5,2),
    data_quality_score  SMALLINT CHECK (data_quality_score BETWEEN 1 AND 5),
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    source_url          TEXT,
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ef_library ON emission_factors(library_id);
CREATE INDEX idx_ef_category_region ON emission_factors(category, region);
CREATE INDEX idx_ef_name_trgm ON emission_factors USING gin (name gin_trgm_ops);
```

### Activity Data and Emissions

```sql
-- Emission sources at each facility (GHG Protocol classification)
CREATE TABLE emission_sources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    facility_id         UUID REFERENCES facilities(id),
    org_unit_id         UUID REFERENCES organizational_units(id),
    scope               SMALLINT NOT NULL CHECK (scope IN (1, 2, 3)),
    scope3_category     SMALLINT CHECK (scope3_category BETWEEN 1 AND 15),
    source_name         VARCHAR(255) NOT NULL,
    source_type         VARCHAR(50) NOT NULL,   -- stationary_combustion, mobile_combustion,
                                                -- fugitive, process, purchased_electricity,
                                                -- purchased_goods, business_travel, etc.
    description         TEXT,
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_emission_sources_tenant ON emission_sources(tenant_id);
CREATE INDEX idx_emission_sources_scope ON emission_sources(scope, scope3_category);

-- Raw activity data entries (the source measurements)
CREATE TABLE activity_data (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    emission_source_id  UUID NOT NULL REFERENCES emission_sources(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    activity_date       DATE NOT NULL,
    quantity            NUMERIC(20,6) NOT NULL,
    unit                VARCHAR(30) NOT NULL,    -- kWh, litres, km, kg, USD, etc.
    data_origin         VARCHAR(30) NOT NULL CHECK (data_origin IN (
                            'manual', 'csv_import', 'api', 'ocr', 'supplier_portal',
                            'iot_meter', 'erp_connector', 'estimated'
                        )),
    data_quality_score  SMALLINT CHECK (data_quality_score BETWEEN 1 AND 5),
    source_document_id  UUID REFERENCES source_documents(id),
    notes               TEXT,
    status              VARCHAR(20) DEFAULT 'pending' CHECK (status IN (
                            'pending', 'validated', 'flagged', 'rejected', 'approved'
                        )),
    validated_by        UUID,
    validated_at        TIMESTAMPTZ,
    created_by          UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_activity_data_source ON activity_data(emission_source_id);
CREATE INDEX idx_activity_data_period ON activity_data(reporting_period_id);
CREATE INDEX idx_activity_data_date ON activity_data(activity_date);

-- Calculated emissions (the output of GHG calculations)
CREATE TABLE emissions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    activity_data_id    UUID NOT NULL REFERENCES activity_data(id),
    emission_factor_id  UUID NOT NULL REFERENCES emission_factors(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    scope               SMALLINT NOT NULL CHECK (scope IN (1, 2, 3)),
    scope3_category     SMALLINT CHECK (scope3_category BETWEEN 1 AND 15),
    calculation_method  VARCHAR(50) NOT NULL CHECK (calculation_method IN (
                            'spend_based', 'activity_based', 'supplier_specific',
                            'hybrid', 'direct_measurement', 'average_data'
                        )),
    -- Individual GHG breakdowns (tonnes)
    co2_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    ch4_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    n2o_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    hfc_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    pfc_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    sf6_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    nf3_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    -- Total CO2-equivalent
    co2e_tonnes         NUMERIC(16,6) NOT NULL,
    -- Market vs location based (Scope 2)
    scope2_method       VARCHAR(20) CHECK (scope2_method IN ('location_based', 'market_based')),
    -- Biogenic CO2 (reported separately per GHG Protocol)
    biogenic_co2_tonnes NUMERIC(16,6) DEFAULT 0,
    -- Uncertainty
    uncertainty_pct     NUMERIC(5,2),
    -- Audit fields
    calculation_engine_version VARCHAR(20),
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_emissions_tenant_period ON emissions(tenant_id, reporting_period_id);
CREATE INDEX idx_emissions_scope ON emissions(scope, scope3_category);
CREATE INDEX idx_emissions_activity ON emissions(activity_data_id);
```

### Source Documents and Data Lineage

```sql
-- Source documents (utility bills, invoices, supplier responses, meter readings)
CREATE TABLE source_documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    document_type       VARCHAR(30) NOT NULL CHECK (document_type IN (
                            'utility_bill', 'invoice', 'meter_reading', 'supplier_response',
                            'travel_receipt', 'fuel_receipt', 'certificate', 'audit_report', 'other'
                        )),
    file_name           VARCHAR(500) NOT NULL,
    file_path           TEXT NOT NULL,
    file_size_bytes     BIGINT,
    mime_type           VARCHAR(100),
    checksum_sha256     CHAR(64),
    ocr_processed       BOOLEAN DEFAULT FALSE,
    ocr_confidence      NUMERIC(5,2),
    extracted_data_json TEXT,                   -- structured data extracted by OCR/LLM
    uploaded_by         UUID NOT NULL,
    uploaded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_source_docs_tenant ON source_documents(tenant_id);

-- Complete audit trail for all data changes
CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    table_name          VARCHAR(100) NOT NULL,
    record_id           UUID NOT NULL,
    action              VARCHAR(10) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    changed_fields      JSONB,
    old_values          JSONB,
    new_values          JSONB,
    changed_by          UUID NOT NULL,
    changed_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address          INET,
    user_agent          TEXT
);
CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id);
CREATE INDEX idx_audit_log_table_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_log_timestamp ON audit_log(changed_at);
-- Partition audit_log by month for performance
-- ALTER TABLE audit_log PARTITION BY RANGE (changed_at);
```

### Supplier Engagement (Scope 3)

```sql
-- Suppliers in the value chain
CREATE TABLE suppliers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    name                VARCHAR(255) NOT NULL,
    legal_name          VARCHAR(500),
    industry_code       VARCHAR(20),
    country_code        CHAR(2),
    contact_email       VARCHAR(255),
    contact_name        VARCHAR(255),
    tier                SMALLINT DEFAULT 1,     -- 1 = direct, 2 = second-tier, etc.
    annual_spend        NUMERIC(16,2),
    spend_currency      CHAR(3) DEFAULT 'USD',
    risk_score          NUMERIC(5,2),
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_suppliers_tenant ON suppliers(tenant_id);

-- Data requests sent to suppliers
CREATE TABLE supplier_data_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    supplier_id         UUID NOT NULL REFERENCES suppliers(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    request_type        VARCHAR(50) NOT NULL,   -- ghg_emissions, energy_data, water_data, etc.
    sent_at             TIMESTAMPTZ,
    due_date            DATE,
    status              VARCHAR(20) DEFAULT 'draft' CHECK (status IN (
                            'draft', 'sent', 'viewed', 'in_progress',
                            'submitted', 'validated', 'rejected', 'overdue'
                        )),
    reminder_count      INTEGER DEFAULT 0,
    last_reminder_at    TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sdr_tenant_period ON supplier_data_requests(tenant_id, reporting_period_id);
CREATE INDEX idx_sdr_supplier ON supplier_data_requests(supplier_id);
CREATE INDEX idx_sdr_status ON supplier_data_requests(status);

-- Supplier-submitted emissions data
CREATE TABLE supplier_emissions_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id          UUID NOT NULL REFERENCES supplier_data_requests(id),
    scope1_co2e_tonnes  NUMERIC(16,6),
    scope2_co2e_tonnes  NUMERIC(16,6),
    scope3_co2e_tonnes  NUMERIC(16,6),
    total_co2e_tonnes   NUMERIC(16,6),
    methodology         VARCHAR(100),
    reporting_boundary  VARCHAR(50),
    data_quality_score  SMALLINT CHECK (data_quality_score BETWEEN 1 AND 5),
    supporting_doc_id   UUID REFERENCES source_documents(id),
    notes               TEXT,
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    validated_by        UUID,
    validated_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Targets, Reduction Plans, and Metrics

```sql
-- Emissions reduction targets (SBTi, net-zero, custom)
CREATE TABLE reduction_targets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    target_name         VARCHAR(255) NOT NULL,
    target_type         VARCHAR(30) NOT NULL CHECK (target_type IN (
                            'absolute', 'intensity', 'net_zero', 'carbon_neutral'
                        )),
    scope_coverage      VARCHAR(30) NOT NULL,   -- scope_1, scope_1_2, scope_1_2_3
    baseline_year       SMALLINT NOT NULL,
    baseline_co2e       NUMERIC(16,6) NOT NULL,
    target_year         SMALLINT NOT NULL,
    target_co2e         NUMERIC(16,6),
    target_reduction_pct NUMERIC(5,2),
    sbti_validated      BOOLEAN DEFAULT FALSE,
    framework_alignment VARCHAR(50),            -- SBTi, Paris, custom
    status              VARCHAR(20) DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Social and Governance metrics (non-GHG ESG data points)
CREATE TABLE esg_metrics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    metric_category     VARCHAR(30) NOT NULL CHECK (metric_category IN (
                            'environmental', 'social', 'governance'
                        )),
    metric_code         VARCHAR(50) NOT NULL,    -- maps to framework disclosure codes
    metric_name         VARCHAR(255) NOT NULL,
    value_numeric       NUMERIC(20,6),
    value_text          TEXT,
    value_boolean       BOOLEAN,
    unit                VARCHAR(50),
    data_origin         VARCHAR(30),
    source_document_id  UUID REFERENCES source_documents(id),
    status              VARCHAR(20) DEFAULT 'draft',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_esg_metrics_tenant_period ON esg_metrics(tenant_id, reporting_period_id);
CREATE INDEX idx_esg_metrics_category ON esg_metrics(metric_category, metric_code);

-- Disclosure responses (the actual text/data submitted for each framework disclosure)
CREATE TABLE disclosure_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    framework_disclosure_id UUID NOT NULL REFERENCES framework_disclosures(id),
    response_text       TEXT,
    response_numeric    NUMERIC(20,6),
    response_boolean    BOOLEAN,
    unit                VARCHAR(50),
    data_sources        UUID[],                  -- array of activity_data / esg_metric IDs
    status              VARCHAR(20) DEFAULT 'draft' CHECK (status IN (
                            'draft', 'pending_review', 'approved', 'published'
                        )),
    reviewed_by         UUID,
    reviewed_at         TIMESTAMPTZ,
    approved_by         UUID,
    approved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, reporting_period_id, framework_disclosure_id)
);
```

### Users, Roles, and Approval Workflows

```sql
-- Users (references external identity provider)
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    email               VARCHAR(255) NOT NULL,
    full_name           VARCHAR(255) NOT NULL,
    role                VARCHAR(30) NOT NULL CHECK (role IN (
                            'admin', 'sustainability_manager', 'data_entry',
                            'reviewer', 'approver', 'auditor', 'viewer'
                        )),
    department          VARCHAR(100),
    is_active           BOOLEAN DEFAULT TRUE,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

-- Approval workflows for four-eyes principle
CREATE TABLE approval_workflows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    workflow_type       VARCHAR(30) NOT NULL,    -- data_approval, report_approval, disclosure_sign_off
    entity_type         VARCHAR(50) NOT NULL,    -- activity_data, emissions, disclosure_response
    entity_id           UUID NOT NULL,
    current_step        SMALLINT DEFAULT 1,
    total_steps         SMALLINT DEFAULT 2,
    status              VARCHAR(20) DEFAULT 'pending' CHECK (status IN (
                            'pending', 'in_review', 'approved', 'rejected', 'escalated'
                        )),
    submitted_by        UUID NOT NULL REFERENCES users(id),
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_steps (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id         UUID NOT NULL REFERENCES approval_workflows(id),
    step_number         SMALLINT NOT NULL,
    reviewer_id         UUID NOT NULL REFERENCES users(id),
    decision            VARCHAR(20) CHECK (decision IN ('approved', 'rejected', 'returned')),
    comments            TEXT,
    decided_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Key Relationships (ERD Summary)

```
tenants
  |-- organizational_units (self-referencing hierarchy)
  |     |-- facilities
  |           |-- emission_sources
  |                 |-- activity_data
  |                       |-- emissions
  |                             |-- emission_factors (lookup)
  |-- reporting_periods
  |     |-- tenant_framework_applicability
  |     |-- disclosure_responses
  |     |-- esg_metrics
  |-- suppliers
  |     |-- supplier_data_requests
  |           |-- supplier_emissions_responses
  |-- reduction_targets
  |-- users
        |-- approval_workflows
              |-- approval_steps

reporting_frameworks
  |-- framework_disclosures
        |-- framework_crosswalks (self-join)
        |-- disclosure_responses

emission_factor_libraries
  |-- emission_factors

source_documents (linked from activity_data, supplier_emissions_responses, esg_metrics)
audit_log (universal, all tables)
```

---

## Materialized Views for Reporting

```sql
-- Aggregated emissions by scope and period (refreshed periodically)
CREATE MATERIALIZED VIEW mv_emissions_summary AS
SELECT
    e.tenant_id,
    e.reporting_period_id,
    rp.name AS period_name,
    e.scope,
    e.scope3_category,
    e.scope2_method,
    SUM(e.co2e_tonnes) AS total_co2e_tonnes,
    SUM(e.co2_tonnes) AS total_co2_tonnes,
    SUM(e.ch4_tonnes) AS total_ch4_tonnes,
    SUM(e.n2o_tonnes) AS total_n2o_tonnes,
    SUM(e.biogenic_co2_tonnes) AS total_biogenic_co2_tonnes,
    COUNT(*) AS emission_count
FROM emissions e
JOIN reporting_periods rp ON rp.id = e.reporting_period_id
GROUP BY e.tenant_id, e.reporting_period_id, rp.name, e.scope, e.scope3_category, e.scope2_method;

CREATE UNIQUE INDEX idx_mv_emissions_summary
    ON mv_emissions_summary(tenant_id, reporting_period_id, scope, scope3_category, scope2_method);

-- Supplier engagement status dashboard
CREATE MATERIALIZED VIEW mv_supplier_engagement AS
SELECT
    sdr.tenant_id,
    sdr.reporting_period_id,
    s.name AS supplier_name,
    s.tier,
    s.annual_spend,
    sdr.status,
    sdr.sent_at,
    sdr.due_date,
    sdr.reminder_count,
    ser.total_co2e_tonnes AS reported_emissions,
    ser.data_quality_score
FROM supplier_data_requests sdr
JOIN suppliers s ON s.id = sdr.supplier_id
LEFT JOIN supplier_emissions_responses ser ON ser.request_id = sdr.id;

-- Framework disclosure completion tracking
CREATE MATERIALIZED VIEW mv_disclosure_completion AS
SELECT
    tfa.tenant_id,
    tfa.reporting_period_id,
    rf.code AS framework_code,
    rf.name AS framework_name,
    COUNT(fd.id) AS total_disclosures,
    COUNT(dr.id) AS completed_disclosures,
    COUNT(CASE WHEN dr.status = 'approved' THEN 1 END) AS approved_disclosures,
    ROUND(COUNT(dr.id)::NUMERIC / NULLIF(COUNT(fd.id), 0) * 100, 1) AS completion_pct
FROM tenant_framework_applicability tfa
JOIN reporting_frameworks rf ON rf.id = tfa.framework_id
JOIN framework_disclosures fd ON fd.framework_id = rf.id
LEFT JOIN disclosure_responses dr ON dr.framework_disclosure_id = fd.id
    AND dr.tenant_id = tfa.tenant_id
    AND dr.reporting_period_id = tfa.reporting_period_id
GROUP BY tfa.tenant_id, tfa.reporting_period_id, rf.code, rf.name;
```

---

## Pros

1. **Audit-grade data integrity.** Fully normalized tables with explicit foreign keys ensure that every emissions figure traces back to an activity data entry, an emission factor, and a source document. This satisfies CSRD's assurance-readiness requirements and supports both limited and reasonable assurance engagements.

2. **Framework crosswalk support.** The `framework_disclosures` and `framework_crosswalks` tables model the many-to-many mapping between overlapping standards (GRI 305-1, ESRS E1-6, IFRS S2.29 all require the same Scope 1 data). This directly supports the platform's core value proposition of single-dataset, multi-framework reporting.

3. **GHG Protocol fidelity.** The schema mirrors the GHG Protocol hierarchy: Organisation > Facility > Emission Source > Activity Data > Emission. Individual greenhouse gas breakdowns (CO2, CH4, N2O, HFCs, PFCs, SF6, NF3) and both Scope 2 methods (location-based and market-based) are first-class fields, not buried in JSON blobs.

4. **Strong typing catches errors early.** CHECK constraints enforce valid scopes (1, 2, 3), valid Scope 3 categories (1-15), valid boundary approaches, and valid calculation methods. This prevents the data quality issues that plague spreadsheet-based processes.

5. **Mature tooling ecosystem.** PostgreSQL 16 with pg_trgm (for emissions factor name matching), materialized views (for dashboard aggregation), and table partitioning (for audit log at scale) leverages a well-understood, widely-deployed stack with excellent ORM support across all popular frameworks.

6. **Row-level security for multi-tenancy.** PostgreSQL's row-level security (RLS) policies can enforce tenant isolation at the database level, providing defense-in-depth beyond application-layer access control.

---

## Cons

1. **Schema rigidity for evolving frameworks.** When EFRAG adds new ESRS data points or ISSB publishes updates, the `framework_disclosures` reference data must be updated, but adding genuinely new metric types (e.g., biodiversity metrics with different structures) requires schema migration. This is painful with 3NF because each new metric shape may demand new tables.

2. **Query complexity for cross-framework reports.** Generating a multi-framework report requires joining through `disclosure_responses`, `framework_crosswalks`, `emissions`, `activity_data`, and `emission_sources` -- seven or more tables. Without careful indexing and materialized views, these queries will be slow.

3. **Scope 3 flexibility limitations.** The 15 Scope 3 categories have very different data structures (business travel needs origin/destination and mode; purchased goods needs SKU and spend data; investments need financial return data). A single `activity_data` table forces all these into the same `quantity` + `unit` pattern, losing domain richness.

4. **Social and governance data is awkward.** The `esg_metrics` table uses a value_numeric/value_text/value_boolean union pattern to accommodate different data types. This is a common anti-pattern in relational models for variable-shape data -- it works but lacks type safety and makes querying less ergonomic.

5. **Emission factor matching at scale.** With 300,000+ factors in a library, the normalized `emission_factors` table requires careful indexing (pg_trgm for text search, composite indexes on category + region + validity dates). NLP-based factor matching will need to operate outside the database via an application layer or vector search extension.

6. **Audit log volume.** A fully normalized audit trail (the `audit_log` table) grows extremely fast in a data-intensive ESG platform. Even with monthly partitioning, this table can reach billions of rows within a few years for large multi-tenant deployments.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ with pg_trgm, pg_partman extensions |
| Connection pooling | PgBouncer or Supavisor |
| ORM | Prisma (TypeScript) or SQLAlchemy (Python) |
| Migrations | Flyway or golang-migrate for versioned schema migrations |
| Full-text search | PostgreSQL tsvector for factor name search; pg_trgm for fuzzy matching |
| Row-level security | PostgreSQL RLS policies for tenant isolation |
| Materialized view refresh | pg_cron for scheduled refresh of reporting aggregations |
| Backup/HA | Patroni + pgBackRest for high availability and point-in-time recovery |
| Monitoring | pg_stat_statements + Prometheus/Grafana |

---

## Migration and Scaling Considerations

### Data Migration Strategy
- **Phase 1:** Core tables (tenants, org_units, facilities, emission_sources, emission_factors) loaded from existing spreadsheet templates via CSV bulk import using PostgreSQL `COPY`.
- **Phase 2:** Historical activity data and emissions backfilled from prior reporting years, with source documents linked retroactively.
- **Phase 3:** Supplier data migrated from existing survey tools (EcoVadis exports, Excel templates).

### Scaling Path
- **Vertical scaling** is sufficient for most deployments up to ~50 tenants with 5 years of data (~10M rows in activity_data, ~50M rows in emissions).
- **Read replicas** handle dashboard and reporting query load separately from write-heavy data collection.
- **Table partitioning** on `audit_log` (by month), `emissions` (by reporting_period), and `activity_data` (by reporting_period) controls table bloat.
- **Materialized view refresh** cadence: every 15 minutes during data collection windows, hourly otherwise.
- **Sharding**: For very large multi-tenant deployments (500+ tenants), Citus extension for PostgreSQL enables horizontal sharding on `tenant_id`, maintaining the relational model while distributing data across nodes.

### Framework Evolution
- New framework versions are handled by inserting new rows into `reporting_frameworks` and `framework_disclosures` -- no schema migration required.
- New metric types that fit the existing quantitative/narrative/boolean pattern are absorbed by `esg_metrics`.
- Genuinely new data structures (e.g., biodiversity assessment matrices) require new dedicated tables and a schema migration.
