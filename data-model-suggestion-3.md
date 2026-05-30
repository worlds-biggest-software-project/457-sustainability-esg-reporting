# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL + JSONB)

> Project: Sustainability & ESG Reporting (457)
> Model: Normalized relational tables for stable domain entities + JSONB columns for variable/framework-specific data
> Generated: 2026-05-25

---

## Design Philosophy

ESG reporting sits at the intersection of two opposing data modeling forces:

1. **Stable, well-defined structures** -- GHG Protocol scopes (1, 2, 3), the 15 Scope 3 categories, organizational hierarchies, emission factors, and approval workflows are standardized and unlikely to change shape. These benefit from normalized relational modeling with strong typing and foreign keys.

2. **Highly variable, framework-dependent structures** -- Each reporting framework (GRI, CSRD/ESRS, ISSB, CDP, SB 253, SEC) defines its own set of data points with different schemas. ESRS E1 has 220 data points; GRI 305 has different metrics; CDP has its own questionnaire structure. Social and governance metrics are even more diverse. New frameworks emerge, existing ones evolve, and the data shapes are inherently polymorphic.

The hybrid approach stores stable entities (organisations, facilities, emission sources, emissions calculations) in normalized relational tables with explicit types and constraints, while using PostgreSQL JSONB columns for the variable parts: framework-specific disclosure data, configurable metrics, custom metadata, and integration payloads that vary by source.

This is the pattern used by Sweep ("flexible data model that adapts to any organisational structure") and is well-aligned with PostgreSQL's strengths as a hybrid relational-document database.

---

## Schema: Normalized Core

### Organization and Structure

```sql
-- Core tenant table: fully normalized, stable fields
CREATE TABLE tenants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    legal_name          VARCHAR(500),
    industry_code       VARCHAR(20),
    industry_standard   VARCHAR(10) DEFAULT 'NACE',
    country_code        CHAR(2) NOT NULL,
    fiscal_year_start   SMALLINT DEFAULT 1,
    boundary_approach   VARCHAR(30) NOT NULL CHECK (boundary_approach IN (
                            'operational_control', 'financial_control', 'equity_share'
                        )),
    base_year           SMALLINT,
    -- JSONB for tenant-level settings that vary by deployment
    settings            JSONB NOT NULL DEFAULT '{}',
    -- Example settings: {
    --   "gwp_source": "AR6",
    --   "default_currency": "EUR",
    --   "scope2_preference": "market_based",
    --   "enabled_frameworks": ["CSRD_ESRS", "GRI", "CDP"],
    --   "custom_fields_schema": { ... }
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Organizational hierarchy: relational (well-defined structure)
CREATE TABLE organizational_units (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    parent_unit_id      UUID REFERENCES organizational_units(id),
    name                VARCHAR(255) NOT NULL,
    unit_type           VARCHAR(30) NOT NULL,
    ownership_pct       NUMERIC(5,2),
    country_code        CHAR(2),
    is_active           BOOLEAN DEFAULT TRUE,
    -- JSONB for unit-specific custom attributes
    attributes          JSONB NOT NULL DEFAULT '{}',
    -- Example: { "legal_entity_id": "DE-HRB-12345", "consolidation_method": "full" }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_units_tenant ON organizational_units(tenant_id);
CREATE INDEX idx_org_units_parent ON organizational_units(parent_unit_id);

-- Facilities: stable columns for common fields, JSONB for facility-type-specific data
CREATE TABLE facilities (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    org_unit_id         UUID NOT NULL REFERENCES organizational_units(id),
    name                VARCHAR(255) NOT NULL,
    facility_type       VARCHAR(50) NOT NULL,
    country_code        CHAR(2) NOT NULL,
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    is_active           BOOLEAN DEFAULT TRUE,
    -- Stable address fields
    address_line1       VARCHAR(255),
    city                VARCHAR(100),
    state_province      VARCHAR(100),
    postal_code         VARCHAR(20),
    -- JSONB for facility-type-specific attributes
    attributes          JSONB NOT NULL DEFAULT '{}',
    -- Examples:
    -- For office: { "gross_floor_area_m2": 5000, "employee_count": 200, "energy_rating": "B" }
    -- For factory: { "production_capacity_tonnes": 50000, "iso14001_certified": true, "stack_count": 3 }
    -- For data_center: { "it_load_kw": 2000, "pue": 1.3, "cooling_type": "free_air" }
    -- For warehouse: { "storage_capacity_m3": 10000, "refrigerated": true, "loading_docks": 12 }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_facilities_tenant ON facilities(tenant_id);
CREATE INDEX idx_facilities_type ON facilities(facility_type);
CREATE INDEX idx_facilities_attrs ON facilities USING gin (attributes);
```

### Emission Factors (Fully Normalized)

```sql
-- Emission factor libraries: stable, well-defined structure
CREATE TABLE emission_factor_libraries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    source              VARCHAR(100) NOT NULL,
    version             VARCHAR(50) NOT NULL,
    publication_date    DATE,
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    licence_type        VARCHAR(50),
    is_active           BOOLEAN DEFAULT TRUE,
    metadata            JSONB NOT NULL DEFAULT '{}',
    -- Example: { "gwp_basis": "AR6", "regions_covered": ["EU", "US", "APAC"], "factor_count": 45000 }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(source, version)
);

-- Individual emission factors: normalized for query performance
-- These are high-volume lookup tables queried millions of times
CREATE TABLE emission_factors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id          UUID NOT NULL REFERENCES emission_factor_libraries(id),
    factor_code         VARCHAR(100),
    name                VARCHAR(500) NOT NULL,
    category            VARCHAR(100) NOT NULL,
    subcategory         VARCHAR(100),
    region              VARCHAR(100),
    gas_type            VARCHAR(20) NOT NULL DEFAULT 'CO2e',
    factor_value        NUMERIC(20,10) NOT NULL,
    unit_numerator      VARCHAR(30) NOT NULL,
    unit_denominator    VARCHAR(50) NOT NULL,
    gwp_source          VARCHAR(50),
    valid_from          DATE NOT NULL,
    valid_to            DATE,
    data_quality_score  SMALLINT,
    is_active           BOOLEAN DEFAULT TRUE,
    -- JSONB for factor-specific technical details that vary by category
    technical_details   JSONB NOT NULL DEFAULT '{}',
    -- Example for electricity: { "grid_region": "ERCOT", "grid_mix": {"coal": 0.2, "gas": 0.45, "wind": 0.25, "solar": 0.1} }
    -- Example for fuel: { "fuel_density_kg_l": 0.832, "net_calorific_value_mj_kg": 43.0 }
    -- Example for transport: { "vehicle_class": "HGV_rigid_7.5_12t", "load_factor": 0.6 }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ef_library ON emission_factors(library_id);
CREATE INDEX idx_ef_category_region ON emission_factors(category, region);
CREATE INDEX idx_ef_name_trgm ON emission_factors USING gin (name gin_trgm_ops);
CREATE INDEX idx_ef_technical ON emission_factors USING gin (technical_details);
```

### Activity Data and Emissions (Hybrid)

```sql
-- Emission sources: relational core with JSONB for source-type-specific configuration
CREATE TABLE emission_sources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    facility_id         UUID REFERENCES facilities(id),
    org_unit_id         UUID REFERENCES organizational_units(id),
    scope               SMALLINT NOT NULL CHECK (scope IN (1, 2, 3)),
    scope3_category     SMALLINT CHECK (scope3_category BETWEEN 1 AND 15),
    source_name         VARCHAR(255) NOT NULL,
    source_type         VARCHAR(50) NOT NULL,
    is_active           BOOLEAN DEFAULT TRUE,
    -- JSONB for source-type-specific configuration
    configuration       JSONB NOT NULL DEFAULT '{}',
    -- Example for stationary_combustion: { "fuel_types": ["natural_gas", "diesel"], "equipment": ["boiler_1", "generator_backup"] }
    -- Example for purchased_electricity: { "grid_region": "ERCOT", "renewable_pct": 30, "contractual_instruments": ["REC_bundle"] }
    -- Example for business_travel: { "travel_policy": "economy_only", "distance_calculation": "great_circle" }
    -- Example for purchased_goods: { "procurement_categories": ["raw_materials", "packaging"], "spend_threshold_usd": 10000 }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_es_tenant ON emission_sources(tenant_id);
CREATE INDEX idx_es_scope ON emission_sources(scope, scope3_category);
CREATE INDEX idx_es_config ON emission_sources USING gin (configuration);

-- Activity data: relational core + JSONB for variable activity details
CREATE TABLE activity_data (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    emission_source_id  UUID NOT NULL REFERENCES emission_sources(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    activity_date       DATE NOT NULL,
    -- Core quantitative fields (always present, always queried)
    quantity            NUMERIC(20,6) NOT NULL,
    unit                VARCHAR(30) NOT NULL,
    data_origin         VARCHAR(30) NOT NULL,
    data_quality_score  SMALLINT CHECK (data_quality_score BETWEEN 1 AND 5),
    status              VARCHAR(20) DEFAULT 'pending',
    -- JSONB for activity-type-specific details
    details             JSONB NOT NULL DEFAULT '{}',
    -- Examples by source type:
    --
    -- Stationary combustion:
    -- { "fuel_type": "natural_gas", "equipment_id": "boiler_1",
    --   "meter_reading_start": 45230, "meter_reading_end": 46100 }
    --
    -- Mobile combustion:
    -- { "vehicle_id": "TRUCK-042", "vehicle_type": "HGV_rigid",
    --   "distance_km": 450, "fuel_litres": 120, "payload_tonnes": 8 }
    --
    -- Business travel (air):
    -- { "departure": "LHR", "arrival": "JFK", "class": "economy",
    --   "trip_type": "round_trip", "passengers": 1, "radiative_forcing": true }
    --
    -- Purchased electricity:
    -- { "tariff": "green_100", "grid_region": "GB",
    --   "renewable_pct": 100, "supplier": "Octopus Energy",
    --   "contractual_instrument": "REGO" }
    --
    -- Purchased goods (spend-based):
    -- { "supplier_name": "Acme Corp", "procurement_category": "raw_materials",
    --   "spend_amount": 125000, "spend_currency": "EUR",
    --   "product_description": "Aluminium ingots Grade A" }
    --
    source_document_id  UUID,
    notes               TEXT,
    validated_by        UUID,
    validated_at        TIMESTAMPTZ,
    created_by          UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ad_source ON activity_data(emission_source_id);
CREATE INDEX idx_ad_period ON activity_data(reporting_period_id);
CREATE INDEX idx_ad_date ON activity_data(activity_date);
CREATE INDEX idx_ad_details ON activity_data USING gin (details);

-- Calculated emissions: fully normalized (high-value, frequently aggregated)
CREATE TABLE emissions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    activity_data_id    UUID NOT NULL REFERENCES activity_data(id),
    emission_factor_id  UUID NOT NULL REFERENCES emission_factors(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    scope               SMALLINT NOT NULL CHECK (scope IN (1, 2, 3)),
    scope3_category     SMALLINT,
    calculation_method  VARCHAR(50) NOT NULL,
    co2_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    ch4_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    n2o_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    hfc_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    pfc_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    sf6_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    nf3_tonnes          NUMERIC(16,6) NOT NULL DEFAULT 0,
    co2e_tonnes         NUMERIC(16,6) NOT NULL,
    scope2_method       VARCHAR(20),
    biogenic_co2_tonnes NUMERIC(16,6) DEFAULT 0,
    uncertainty_pct     NUMERIC(5,2),
    -- JSONB for calculation traceability metadata
    calculation_details JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "engine_version": "2.3.1",
    --   "gwp_values_used": { "CH4": 28, "N2O": 265 },
    --   "intermediate_steps": [
    --     { "step": "unit_conversion", "input": "1200 litres", "output": "999.6 kg" },
    --     { "step": "factor_application", "factor": 2.68, "result": "2678.93 kg CO2e" }
    --   ],
    --   "data_quality_assessment": { "score": 3, "method": "PCAF", "factors": ["estimated_ef", "proxy_data"] }
    -- }
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_em_tenant_period ON emissions(tenant_id, reporting_period_id);
CREATE INDEX idx_em_scope ON emissions(scope, scope3_category);
CREATE INDEX idx_em_activity ON emissions(activity_data_id);
```

### Reporting Periods and Frameworks

```sql
CREATE TABLE reporting_periods (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    name                VARCHAR(100) NOT NULL,
    period_type         VARCHAR(20) NOT NULL,
    start_date          DATE NOT NULL,
    end_date            DATE NOT NULL,
    status              VARCHAR(20) DEFAULT 'draft',
    locked_at           TIMESTAMPTZ,
    locked_by           UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

-- Framework definitions with JSONB for framework-specific schema definitions
CREATE TABLE reporting_frameworks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code                VARCHAR(30) NOT NULL UNIQUE,
    name                VARCHAR(255) NOT NULL,
    version             VARCHAR(30) NOT NULL,
    governing_body      VARCHAR(100),
    effective_date      DATE,
    is_active           BOOLEAN DEFAULT TRUE,
    -- JSONB stores the framework's disclosure taxonomy/schema
    disclosure_schema   JSONB NOT NULL DEFAULT '{}',
    -- This is the key innovation: the framework's own data point definitions
    -- are stored as a JSONB document rather than normalized into rows.
    -- Example for GRI:
    -- {
    --   "standards": {
    --     "GRI 305": {
    --       "disclosures": {
    --         "305-1": { "name": "Direct GHG emissions (Scope 1)", "type": "quantitative",
    --                    "unit": "tonnes CO2e", "fields": ["gross_scope1", "gases_included", "biogenic_co2",
    --                    "base_year", "source_of_ef", "consolidation_approach"] },
    --         "305-2": { "name": "Energy indirect GHG emissions (Scope 2)", "type": "quantitative",
    --                    "fields": ["gross_location_based", "gross_market_based", "gases_included"] },
    --         ...
    --       }
    --     }
    --   }
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rf_schema ON reporting_frameworks USING gin (disclosure_schema);
```

### Framework-Agnostic Disclosure Responses (JSONB-Heavy)

```sql
-- Disclosure responses use JSONB for framework-specific response data
-- This is where the hybrid model pays off most: each framework has a different
-- data shape for its disclosures, and new frameworks can be added without schema migration.
CREATE TABLE disclosure_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    framework_code      VARCHAR(30) NOT NULL,
    disclosure_code     VARCHAR(50) NOT NULL,
    -- JSONB for the response data, structured according to the framework's disclosure_schema
    response_data       JSONB NOT NULL,
    -- Examples:
    --
    -- GRI 305-1 (Scope 1):
    -- {
    --   "gross_scope1_tonnes_co2e": 12500.45,
    --   "gases_included": ["CO2", "CH4", "N2O"],
    --   "biogenic_co2_tonnes": 340.2,
    --   "base_year": 2020,
    --   "base_year_emissions_tonnes": 15200.0,
    --   "source_of_emission_factors": "IPCC 2023",
    --   "consolidation_approach": "operational_control",
    --   "methodology": "GHG Protocol Corporate Standard"
    -- }
    --
    -- ESRS E1-6 (Scope 1, 2, 3 GHG Emissions):
    -- {
    --   "gross_scope1": { "total_co2e": 12500.45, "by_country": {"DE": 8000, "FR": 4500} },
    --   "gross_scope2_location": { "total_co2e": 6200.3 },
    --   "gross_scope2_market": { "total_co2e": 4100.1 },
    --   "gross_scope3": {
    --     "total_co2e": 85000.0,
    --     "by_category": {
    --       "1_purchased_goods": 45000, "4_upstream_transport": 12000,
    --       "6_business_travel": 3000, "11_use_of_sold_products": 25000
    --     },
    --     "significant_categories_disclosed": [1, 4, 6, 11]
    --   },
    --   "total_ghg_emissions": 103800.85,
    --   "ghg_intensity_per_revenue": { "value": 45.2, "unit": "tonnes CO2e / EUR million" }
    -- }
    --
    -- CDP Climate C6.1 (Scope 1 by country):
    -- {
    --   "countries": [
    --     { "country": "Germany", "scope1_tonnes": 8000, "gases": ["CO2", "CH4"] },
    --     { "country": "France", "scope1_tonnes": 4500, "gases": ["CO2", "CH4", "N2O"] }
    --   ]
    -- }
    --
    -- Social metric (ESRS S1 - Own Workforce):
    -- {
    --   "total_employees": 5200,
    --   "by_gender": { "male": 2800, "female": 2300, "non_binary": 50, "undisclosed": 50 },
    --   "by_contract": { "permanent": 4800, "temporary": 400 },
    --   "by_region": { "EU": 3500, "APAC": 1200, "Americas": 500 },
    --   "turnover_rate_pct": 12.5,
    --   "gender_pay_gap_pct": 8.2
    -- }
    
    -- Standard relational fields for workflow and audit
    status              VARCHAR(20) DEFAULT 'draft' CHECK (status IN (
                            'draft', 'pending_review', 'approved', 'published'
                        )),
    data_sources        UUID[],
    reviewed_by         UUID,
    reviewed_at         TIMESTAMPTZ,
    approved_by         UUID,
    approved_at         TIMESTAMPTZ,
    created_by          UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, reporting_period_id, framework_code, disclosure_code)
);
CREATE INDEX idx_dr_tenant_period ON disclosure_responses(tenant_id, reporting_period_id);
CREATE INDEX idx_dr_framework ON disclosure_responses(framework_code, disclosure_code);
CREATE INDEX idx_dr_response ON disclosure_responses USING gin (response_data);
CREATE INDEX idx_dr_status ON disclosure_responses(status);
```

### ESG Metrics Store (Schema-on-Read via JSONB)

```sql
-- Universal ESG metrics store for Social and Governance data
-- Uses JSONB for the actual metric value because S and G metrics are enormously diverse
CREATE TABLE esg_metrics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    org_unit_id         UUID REFERENCES organizational_units(id),
    facility_id         UUID REFERENCES facilities(id),
    -- Categorization (relational for filtering/aggregation)
    pillar              VARCHAR(1) NOT NULL CHECK (pillar IN ('E', 'S', 'G')),
    topic               VARCHAR(100) NOT NULL,
    subtopic            VARCHAR(100),
    metric_code         VARCHAR(50) NOT NULL,
    metric_name         VARCHAR(255) NOT NULL,
    -- JSONB for the actual metric value (flexible shape)
    metric_value        JSONB NOT NULL,
    -- Examples:
    -- Numeric: { "value": 12.5, "unit": "percent" }
    -- Boolean: { "value": true }
    -- Narrative: { "value": "The company has implemented a whistleblower policy..." }
    -- Structured: { "male": 2800, "female": 2300, "total": 5100 }
    -- List: { "policies": ["anti-corruption", "human-rights", "environmental"] }
    -- Table: { "rows": [{ "region": "EU", "incidents": 2 }, { "region": "APAC", "incidents": 0 }] }
    
    data_origin         VARCHAR(30),
    data_quality_score  SMALLINT,
    source_document_id  UUID,
    evidence_notes      TEXT,
    status              VARCHAR(20) DEFAULT 'draft',
    created_by          UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_esg_tenant_period ON esg_metrics(tenant_id, reporting_period_id);
CREATE INDEX idx_esg_pillar_topic ON esg_metrics(pillar, topic, subtopic);
CREATE INDEX idx_esg_metric_code ON esg_metrics(metric_code);
CREATE INDEX idx_esg_value ON esg_metrics USING gin (metric_value);
```

### Supplier Engagement

```sql
-- Suppliers: relational core + JSONB for variable supplier attributes
CREATE TABLE suppliers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    name                VARCHAR(255) NOT NULL,
    country_code        CHAR(2),
    tier                SMALLINT DEFAULT 1,
    annual_spend        NUMERIC(16,2),
    spend_currency      CHAR(3) DEFAULT 'USD',
    is_active           BOOLEAN DEFAULT TRUE,
    -- JSONB for variable supplier metadata
    profile             JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "contact_name": "Jane Doe",
    --   "contact_email": "jane@supplier.com",
    --   "industry_code": "C25.1",
    --   "certifications": ["ISO 14001", "ISO 50001"],
    --   "ecovadis_score": 65,
    --   "sbti_committed": true,
    --   "last_audit_date": "2025-06-15",
    --   "risk_factors": ["high_water_stress_region", "energy_intensive_process"]
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_suppliers_tenant ON suppliers(tenant_id);
CREATE INDEX idx_suppliers_profile ON suppliers USING gin (profile);

-- Supplier data requests
CREATE TABLE supplier_data_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    supplier_id         UUID NOT NULL REFERENCES suppliers(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    status              VARCHAR(20) DEFAULT 'draft',
    sent_at             TIMESTAMPTZ,
    due_date            DATE,
    reminder_count      INTEGER DEFAULT 0,
    last_reminder_at    TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    -- JSONB for the request template (varies by request type)
    request_template    JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "request_type": "ghg_and_energy",
    --   "requested_scopes": [1, 2],
    --   "requested_metrics": ["total_energy_mwh", "renewable_pct", "water_withdrawal_m3"],
    --   "questionnaire_version": "2026-Q1"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Supplier responses: JSONB for variable response data
CREATE TABLE supplier_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id          UUID NOT NULL REFERENCES supplier_data_requests(id),
    -- JSONB for the actual response data (matches the request template structure)
    response_data       JSONB NOT NULL,
    -- Example: {
    --   "scope1_co2e_tonnes": 450.3,
    --   "scope2_co2e_tonnes": 220.1,
    --   "total_energy_mwh": 3500,
    --   "renewable_pct": 45,
    --   "water_withdrawal_m3": 12000,
    --   "methodology": "GHG Protocol",
    --   "boundary": "operational_control",
    --   "reporting_year": 2025,
    --   "third_party_verified": false,
    --   "comments": "Scope 2 uses location-based method"
    -- }
    data_quality_score  SMALLINT,
    supporting_doc_id   UUID,
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    validated_by        UUID,
    validated_at        TIMESTAMPTZ,
    status              VARCHAR(20) DEFAULT 'submitted',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sr_request ON supplier_responses(request_id);
CREATE INDEX idx_sr_response ON supplier_responses USING gin (response_data);
```

### Audit Trail and Document Management

```sql
-- Source documents: relational metadata + JSONB for extracted data
CREATE TABLE source_documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    document_type       VARCHAR(30) NOT NULL,
    file_name           VARCHAR(500) NOT NULL,
    file_path           TEXT NOT NULL,
    file_size_bytes     BIGINT,
    mime_type           VARCHAR(100),
    checksum_sha256     CHAR(64),
    uploaded_by         UUID NOT NULL,
    uploaded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- JSONB for OCR/LLM-extracted structured data
    extracted_data      JSONB,
    -- Example for utility bill: {
    --   "ocr_confidence": 0.94,
    --   "processing_engine": "tesseract+gpt4o",
    --   "extracted_fields": {
    --     "supplier": "British Gas",
    --     "account_number": "12345678",
    --     "billing_period": { "from": "2025-01-01", "to": "2025-01-31" },
    --     "total_kwh": 12500,
    --     "total_cost": { "amount": 3750.00, "currency": "GBP" },
    --     "meter_readings": { "opening": 45230, "closing": 57730 }
    --   },
    --   "classification": { "type": "electricity_bill", "confidence": 0.97 }
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sd_tenant ON source_documents(tenant_id);
CREATE INDEX idx_sd_extracted ON source_documents USING gin (extracted_data);

-- Audit log: relational structure + JSONB for change details
CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    table_name          VARCHAR(100) NOT NULL,
    record_id           UUID NOT NULL,
    action              VARCHAR(10) NOT NULL,
    changed_by          UUID NOT NULL,
    changed_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- JSONB for the actual changes (flexible: works for both relational and JSONB fields)
    changes             JSONB NOT NULL,
    -- Example: {
    --   "old": { "quantity": 1200, "unit": "kWh", "status": "pending" },
    --   "new": { "quantity": 1250, "unit": "kWh", "status": "validated" },
    --   "changed_fields": ["quantity", "status"]
    -- }
    ip_address          INET,
    correlation_id      UUID
);
CREATE INDEX idx_al_tenant ON audit_log(tenant_id, changed_at DESC);
CREATE INDEX idx_al_record ON audit_log(table_name, record_id);
CREATE INDEX idx_al_changes ON audit_log USING gin (changes);
```

### Framework Crosswalks (Hybrid)

```sql
-- Framework crosswalk definitions stored as JSONB documents
-- This avoids the explosion of many-to-many join tables when mapping 6+ frameworks
CREATE TABLE framework_crosswalks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    version             VARCHAR(30) NOT NULL,
    -- JSONB mapping document: defines equivalences across all frameworks
    mappings            JSONB NOT NULL,
    -- Example:
    -- {
    --   "scope1_emissions": {
    --     "concept": "Total Scope 1 GHG Emissions",
    --     "frameworks": {
    --       "GRI": { "disclosure": "305-1", "field": "gross_scope1_tonnes_co2e" },
    --       "CSRD_ESRS": { "disclosure": "E1-6", "field": "gross_scope1.total_co2e" },
    --       "ISSB_S2": { "disclosure": "S2.29(a)", "field": "scope1_ghg_emissions" },
    --       "CDP": { "disclosure": "C6.1", "field": "total_scope1" },
    --       "SB253": { "disclosure": "scope1_reporting", "field": "total_scope1_mt" }
    --     },
    --     "unit": "tonnes CO2e",
    --     "aggregation": "sum",
    --     "source_table": "emissions",
    --     "source_filter": { "scope": 1 }
    --   },
    --   "scope2_market_based": {
    --     "concept": "Scope 2 Market-Based GHG Emissions",
    --     "frameworks": {
    --       "GRI": { "disclosure": "305-2", "field": "gross_market_based" },
    --       "CSRD_ESRS": { "disclosure": "E1-6", "field": "gross_scope2_market.total_co2e" },
    --       ...
    --     }
    --   }
    -- }
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fc_mappings ON framework_crosswalks USING gin (mappings);
```

### Users and Workflows (Fully Relational)

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    email               VARCHAR(255) NOT NULL,
    full_name           VARCHAR(255) NOT NULL,
    role                VARCHAR(30) NOT NULL,
    department          VARCHAR(100),
    is_active           BOOLEAN DEFAULT TRUE,
    preferences         JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE approval_workflows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    reporting_period_id UUID NOT NULL REFERENCES reporting_periods(id),
    entity_type         VARCHAR(50) NOT NULL,
    entity_id           UUID NOT NULL,
    status              VARCHAR(20) DEFAULT 'pending',
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
    decision            VARCHAR(20),
    comments            TEXT,
    decided_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## JSONB Validation Strategy

Since JSONB columns lack the compile-time type safety of relational columns, validation must be enforced at the application layer and optionally at the database layer:

### Application-Layer Validation (Primary)

```
Framework disclosure schemas → JSON Schema validation in application code
  - When a disclosure_response is submitted, validate response_data against
    the framework's disclosure_schema from the reporting_frameworks table
  - Fail fast on schema violations before persisting

Activity data details → Type-specific JSON Schema per source_type
  - When activity_data.details is submitted, validate against the schema
    defined for that emission source's source_type
  - Ensures fuel_type is present for combustion, departure/arrival for travel, etc.
```

### Database-Layer Validation (Defense-in-Depth)

```sql
-- Example: ensure response_data always has required top-level keys
ALTER TABLE disclosure_responses ADD CONSTRAINT chk_response_data_valid
    CHECK (
        response_data IS NOT NULL
        AND jsonb_typeof(response_data) = 'object'
        AND response_data != '{}'::jsonb
    );

-- Example: ensure activity details contain required fields for combustion sources
-- (applied via trigger rather than CHECK for complex cross-table logic)
CREATE OR REPLACE FUNCTION validate_activity_details()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT source_type FROM emission_sources WHERE id = NEW.emission_source_id) = 'stationary_combustion' THEN
        IF NOT (NEW.details ? 'fuel_type') THEN
            RAISE EXCEPTION 'Activity details for stationary combustion must include fuel_type';
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_validate_activity_details
    BEFORE INSERT OR UPDATE ON activity_data
    FOR EACH ROW EXECUTE FUNCTION validate_activity_details();
```

---

## Query Patterns

### Cross-Framework Report Generation

```sql
-- Generate a multi-framework emissions disclosure from a single query
SELECT
    dr.framework_code,
    dr.disclosure_code,
    dr.response_data,
    dr.status,
    dr.approved_at
FROM disclosure_responses dr
WHERE dr.tenant_id = $1
  AND dr.reporting_period_id = $2
  AND dr.framework_code = ANY($3)  -- e.g., ARRAY['GRI', 'CSRD_ESRS', 'ISSB_S2']
ORDER BY dr.framework_code, dr.disclosure_code;
```

### Querying JSONB Details

```sql
-- Find all activity data entries for a specific fuel type across all sources
SELECT ad.id, ad.activity_date, ad.quantity, ad.unit,
       ad.details->>'fuel_type' AS fuel_type,
       ad.details->>'equipment_id' AS equipment
FROM activity_data ad
WHERE ad.tenant_id = $1
  AND ad.details @> '{"fuel_type": "natural_gas"}'::jsonb;

-- Aggregate supplier responses by certification status
SELECT
    s.name,
    sr.response_data->>'scope1_co2e_tonnes' AS scope1,
    s.profile->'certifications' AS certs,
    s.profile->>'ecovadis_score' AS ecovadis
FROM suppliers s
JOIN supplier_data_requests sdr ON sdr.supplier_id = s.id
JOIN supplier_responses sr ON sr.request_id = sdr.id
WHERE s.tenant_id = $1
  AND s.profile @> '{"sbti_committed": true}'::jsonb;
```

---

## Pros

1. **Framework evolution without schema migration.** Adding support for a new reporting framework (e.g., TNFD, a new SASB industry standard) requires inserting a new row into `reporting_frameworks` with its `disclosure_schema` in JSONB and adding a crosswalk mapping. No DDL changes, no downtime, no migration scripts. This is the single biggest advantage for a platform whose competitive edge is multi-framework support.

2. **Scope 3 category diversity handled naturally.** Each Scope 3 category has radically different data requirements (business travel needs airports and cabin class; purchased goods needs spend amounts and procurement categories; waste treatment needs disposal method and weight). JSONB `details` columns on `activity_data` accommodate this diversity without requiring 15 separate tables or an awkward union type.

3. **Social and governance data flexibility.** S and G metrics are enormously diverse -- workforce demographics, pay gap ratios, board composition, anti-corruption policies, supply chain due diligence findings. The `esg_metrics` table with JSONB `metric_value` handles all of these without schema changes, while the relational `pillar`, `topic`, and `metric_code` columns enable structured querying and aggregation.

4. **Strong relational integrity where it matters most.** Emissions calculations -- the core GHG accounting engine -- remain fully normalized with explicit foreign keys to activity data and emission factors. The calculation chain (source > activity > emission > factor) has the same audit-grade traceability as a pure relational model.

5. **OCR/LLM integration is natural.** Document ingestion produces semi-structured data that varies by document type. The `extracted_data` JSONB column on `source_documents` stores whatever the OCR/LLM pipeline produces without forcing premature schema design for every document type.

6. **GIN indexes make JSONB queryable.** PostgreSQL GIN indexes on JSONB columns enable efficient containment queries (`@>`), existence checks (`?`), and path-based lookups. The performance penalty versus relational columns is real but manageable for the access patterns in ESG reporting (primarily dashboard aggregation and report generation, not OLTP transaction processing).

---

## Cons

1. **Reduced type safety for JSONB columns.** There is no database-level enforcement that a GRI 305-1 response contains `gross_scope1_tonnes_co2e` as a numeric value. Validation must be handled at the application layer via JSON Schema or equivalent. Bugs in validation logic can allow malformed data into the database.

2. **JSONB aggregation is slower than columnar aggregation.** Queries that aggregate across JSONB fields (e.g., "sum all scope1_co2e_tonnes from supplier responses") require extracting values with `->>`and casting, which is significantly slower than summing a native NUMERIC column. Materialized views or application-layer pre-aggregation can mitigate this.

3. **Schema documentation burden.** With relational columns, the schema _is_ the documentation. With JSONB, the expected shapes must be documented separately (in JSON Schema files, developer documentation, or inline comments). This documentation can drift from reality over time.

4. **ORM support is uneven.** While modern ORMs (Prisma, SQLAlchemy, TypeORM) support JSONB, the developer experience for type-safe access to JSONB fields is less polished than for relational columns. TypeScript developers in particular must maintain manual type definitions for JSONB shapes.

5. **Migration complexity for JSONB changes.** If the shape of a JSONB field needs to change retroactively (e.g., renaming a key in `response_data` across all GRI 305-1 responses), the migration requires a JSON path update across all rows rather than a simple `ALTER TABLE RENAME COLUMN`.

6. **Audit trail for JSONB changes.** Tracking which specific field within a JSONB column changed requires JSONB diff logic in the audit trigger. This is more complex than tracking changes to individual relational columns.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ with pg_trgm, btree_gin extensions |
| JSONB validation | AJV (JavaScript) or jsonschema (Python) for application-layer JSON Schema validation |
| ORM | Prisma with custom JSONB type definitions; or Drizzle ORM for better JSON column support |
| JSONB indexing | GIN indexes on all JSONB columns that are queried; jsonb_path_ops for containment-only queries |
| Schema registry | Store JSON Schema definitions for each framework's disclosure_schema in a version-controlled schema registry |
| Migration | Flyway with custom JSONB migration scripts; or application-layer JSONB migrations |
| Search | PostgreSQL full-text search on JSONB text fields; pg_trgm for emission factor fuzzy matching |
| Caching | Redis for frequently-accessed framework schemas and crosswalk mappings |

---

## Migration and Scaling Considerations

### Migration from Pure Relational
- If starting from Data Model 1 (pure relational), the migration to hybrid is straightforward: add JSONB columns to existing tables, then gradually move variable/framework-specific data from normalized tables into JSONB. The relational columns can be retained as a fallback during transition.

### Data Import Strategy
- CSV imports map to the relational core (activity_data with quantity, unit, date).
- ERP connector data goes into activity_data with source-type-specific `details` populated from the connector's field mapping configuration.
- Supplier portal responses land directly in `supplier_responses.response_data` as JSONB, validated against the `request_template` schema.
- OCR/LLM pipeline output goes into `source_documents.extracted_data` as JSONB.

### Scaling Path
- **Vertical scaling** works well up to 100+ tenants and 10+ years of data on PostgreSQL.
- **Read replicas** offload dashboard and report generation queries.
- **Partitioning** on `audit_log` (by month) and `activity_data` (by reporting period).
- **GIN index maintenance:** JSONB GIN indexes need periodic `REINDEX` and monitoring via `pg_stat_user_indexes` to ensure they remain performant as data grows.
- **JSONB column size monitoring:** Set alerts if any individual JSONB column value exceeds 1MB (unusual but possible for very complex disclosure responses). PostgreSQL TOAST compression handles this transparently but performance degrades for very large documents.
- **For very large scale:** Consider extracting the most frequently aggregated JSONB fields into materialized views with typed columns for dashboard performance.

### Framework Version Management
- When a framework publishes a new version (e.g., ESRS Set 2), insert a new `reporting_frameworks` row with the updated `disclosure_schema`.
- Existing disclosure responses for the old version remain valid and linked to the old framework row.
- A migration utility can transform old-version JSONB responses to new-version shape where mappings exist.
