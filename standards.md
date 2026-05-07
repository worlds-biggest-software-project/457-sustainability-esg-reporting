# Standards & API Reference

> Project: Sustainability & ESG Reporting · Generated: 2026-05-07

---

## Industry Standards & Specifications

### GHG Accounting Standards

**GHG Protocol Corporate Accounting and Reporting Standard**
- URL: https://ghgprotocol.org/corporate-standard
- The foundational methodology for corporate Scope 1, 2, and 3 greenhouse gas inventories. Used by more than 90% of Fortune 500 companies reporting to CDP. The GHG Protocol defines Scope boundaries, allocation rules, and calculation approaches that all major ESG platforms implement. Licensed under Creative Commons — freely available for implementation.

**ISO 14064-1:2018 — Greenhouse Gases: Quantification and Reporting at Organisation Level**
- URL: https://www.iso.org/standard/66453.html
- The ISO standard for corporate-level GHG inventory design, reporting, and third-party verification. Closely aligned with the GHG Protocol Corporate Standard but adds certification and accredited verification pathways. ISO and the GHG Protocol announced a strategic partnership in 2024 to harmonise their standards into a unified global framework. Relevant for platforms seeking assurance-grade audit trails.

**ISO 14067:2018 — Carbon Footprint of Products**
- URL: https://www.iso.org/standard/71206.html
- Specifies requirements for quantifying and communicating the carbon footprint of products (PCF) across their full lifecycle (cradle-to-gate or cradle-to-grave). Builds on ISO 14040/14044 LCA principles but narrows scope to carbon metrics. As of 2026, transitioning from voluntary standard to market-access requirement in the EU under the proposed Green Claims Directive. Essential for any product carbon footprinting (PCF) module.

**ISO 14040/14044 — Life Cycle Assessment (LCA)**
- URL: https://www.iso.org/standard/37456.html
- Defines principles and methodology for full environmental life cycle assessment, underpinning ISO 14067 and ecoinvent-based emissions factor databases. Relevant for Scope 3 Category 1 (purchased goods and services) and product-level carbon calculations.

### Sustainability Disclosure Frameworks

**IFRS S1 — General Requirements for Disclosure of Sustainability-related Financial Information**
- URL: https://www.ifrs.org/issued-standards/ifrs-sustainability-standards-navigator/ifrs-s1-general-requirements/
- The ISSB's general framework for investor-focused sustainability disclosure. Establishes materiality, governance, risk, and opportunity disclosure requirements applicable across all sustainability topics. Jurisdictions adopting IFRS S1 as of 2026 include Australia, UK, Canada, Japan, Singapore, and others. Supersedes the TCFD recommendations, which were disbanded in October 2023 and absorbed into ISSB.

**IFRS S2 — Climate-related Disclosures**
- URL: https://www.ifrs.org/issued-standards/ifrs-sustainability-standards-navigator/ifrs-s2-climate-related-disclosures/
- The ISSB's climate-specific disclosure standard, fully integrating the retired TCFD framework. Requires disclosure of climate-related risks, opportunities, governance, strategy, scenario analysis, and GHG emissions (Scopes 1, 2, and 3). Central standard for any investor-facing ESG reporting platform in 2026.

**GRI Universal Standards (GRI 1, GRI 2, GRI 3) and Topic Standards**
- URL: https://www.globalreporting.org/standards/
- The dominant multi-stakeholder sustainability reporting framework globally. GRI Universal Standards set general disclosure requirements; Topic Standards cover specific environmental, social, and governance areas. GRI launched an XBRL-based digital taxonomy in June 2025 enabling machine-readable GRI filings and API submission. GRI remains the most commonly used framework for non-financial ESG disclosure.

**ESRS — European Sustainability Reporting Standards (CSRD)**
- URL: https://www.efrag.org/en/sustainability-reporting/esrs-workstreams
- The 12 mandatory disclosure standards issued under the EU's Corporate Sustainability Reporting Directive (CSRD), covering climate (E1), pollution, water, biodiversity, circular economy, own workforce, value chain workers, affected communities, consumers, and business conduct. The 2025 EU Omnibus Simplification reduced the ESRS data point count by approximately 50%. As of 2026, large EU companies file ESRS disclosures in XBRL-tagged Inline XBRL format under the European Single Electronic Format (ESEF).

**SASB Standards (now under ISSB/IFRS Foundation)**
- URL: https://sasb.ifrs.org/standards/
- 77 industry-specific standards defining the financially material ESG topics and metrics for each industry sector, classified across 11 sectors. The 2024 SASB XBRL Taxonomy contains approximately 4,620 reporting concepts. Available for download via the IFRS Foundation; taxonomy files are freely browsable. Directly referenced in IFRS S1 as industry-specific guidance.

**TCFD — Task Force on Climate-related Financial Disclosures (archived)**
- URL: https://www.fsb-tcfd.org/
- Disbanded October 2023; its four-pillar framework (Governance, Strategy, Risk Management, Metrics and Targets) is now legally embodied in IFRS S2. Still referenced in California SB 261 (companies with $500M+ revenue must prepare TCFD-aligned reports) and in many contractual and bond covenant contexts. Platforms must maintain backward compatibility with TCFD output.

**TNFD — Taskforce on Nature-related Financial Disclosures**
- URL: https://tnfd.global/
- Published final recommendations in September 2023 using the same four-pillar structure as TCFD but focused on biodiversity, water, and land use. TNFD disclosures are becoming mandatory in several jurisdictions from 2026 (e.g., Australia). An emerging requirement for any platform seeking to support nature-related disclosure alongside climate.

### Digital Reporting and Data Exchange Standards

**ESRS XBRL Taxonomy (EFRAG / ESMA)**
- URL: https://www.efrag.org/en/sustainability-reporting/esrs-workstreams/digital-reporting-with-xbrl
- The machine-readable digital taxonomy enabling XBRL tagging of CSRD/ESRS sustainability statements. Covers 1,232 ESRS data points across 12 standards. EFRAG committed to delivering an updated taxonomy by December 2026. Mandatory for EU companies filing under CSRD from the 2025 financial year onwards (filings due 2026). Required for any platform targeting CSRD compliance in the European market.

**GRI Sustainability Taxonomy (XBRL)**
- URL: https://www.globalreporting.org/standards/standards-development/gri-sustainability-taxonomy/
- GRI's XBRL-based digital taxonomy launched June 2025, covering all GRI Universal, Sector, and Topic Standards. Enables machine-readable digital GRI filings via a desktop app and API. Co-developed with TCS. Provides the basis for direct filing to GRI and ERP integration via secure APIs.

**SASB Standards XBRL Taxonomy**
- URL: https://sasb.ifrs.org/sasb-standards-taxonomy/
- XBRL taxonomy for SASB's 77 industry standards, updated in October 2024. Contains approximately 4,620 reporting concepts. Available as downloadable taxonomy files from the IFRS Foundation for integration into reporting tools. Supports tagging of sustainability-related financial information for investors.

**PACT (Partnership for Carbon Transparency) — Pathfinder Data Exchange Protocol**
- URL: https://www.carbon-transparency.org/
- WBCSD-led standard for peer-to-peer exchange of product carbon footprint (PCF) data across supply chains. Originally called the Pathfinder Framework (2021); now the PACT Methodology. Defines a standardised API and data schema for companies to share cradle-to-gate PCF data with supply chain partners. Supported by the Open Footprint Forum. Critical for any Scope 3 Category 1 and product-level carbon data exchange module.

**Open Footprint Forum Data Model and API Standard**
- URL: https://www.opengroup.org/openfootprint-forum
- The Open Group initiative defining open data models and API standards for GHG footprint data. Supports GHG Protocol Scope 1/2/3, WBCSD/PACT, GRI, ISSB, PCAF, and other audit and reporting standards. Presented at The Open Group Summit Oslo, April 2026. Relevant as an interoperability layer for any platform seeking to exchange ESG data between systems without proprietary lock-in.

**CDP Disclosure API**
- URL: https://www.cdp.net/en/insights/cdp-disclosure-api-2026-streamline-your-disclosure
- CDP's official API enabling direct data transfer from sustainability software platforms into the CDP Portal, eliminating manual re-entry for the annual CDP questionnaire. Available for full corporate, SME, Cities, and States and Regions questionnaires. 2026 Disclosure API programme participating providers include Salesforce, Workiva, Novisto, and others. Accredited Solutions Provider (ASP) programme governs API access.

### Security and Authentication Standards

**OAuth 2.0 / OpenID Connect**
- URL: https://oauth.net/2/ / https://openid.net/connect/
- Standard authentication and authorisation protocols required for secure third-party data integrations (ERP connectors, utility API connections, supplier portals). All major ESG platforms use OAuth 2.0 for API authentication. OpenID Connect provides identity layer on top of OAuth 2.0 for user authentication in multi-tenant SaaS environments.

**GDPR and EU Data Protection Regulation**
- URL: https://gdpr-info.eu/
- Sustainability reporting platforms handling EU company and supply chain data must comply with GDPR. Relevant for supplier portal data collection, data residency, and retention policies. Platforms serving CSRD-reporting companies will process personal data from employees (workforce disclosures) and supply chain contacts, triggering GDPR obligations.

**SOC 2 Type II**
- Not an ISO/IETF standard but a de-facto assurance requirement. All major ESG platforms (Watershed, Persefoni, Workiva) publish SOC 2 Type II reports demonstrating security, availability, and confidentiality controls. Required by enterprise procurement and assurance teams for any platform handling audit-ready ESG data.

---

## Similar Products — Developer Documentation & APIs

### Climatiq

- **Description:** Carbon emissions calculation API providing programmatic access to 944,000+ verified emissions factors across energy, travel, freight, procurement, and product carbon footprinting use cases. Designed as the emissions calculation layer for developers building carbon accounting features into applications.
- **API Documentation:** https://www.climatiq.io/docs
- **API Reference:** https://www.climatiq.io/docs/api-reference
- **Quickstart Guide:** https://www.climatiq.io/docs/guides/tutorials/quickstart
- **SDKs / Libraries:** Python snippets provided; REST API usable from any language
- **Standards Compliance:** REST/JSON, ISO 14067-aligned PCF calculations, GHG Protocol-aligned, ecoinvent emission factors available via API
- **Authentication:** API key (HTTPS required for all requests)
- **Notable:** Includes CBAM (EU Carbon Border Adjustment Mechanism) endpoint; Autopilot feature for automated factor matching; 944k+ factors including ecoinvent LCA data

### Watershed (Climate Platform)

- **Description:** Enterprise sustainability AI platform for emissions measurement, decarbonisation, and ESG reporting, targeting large enterprises. Includes OCR-based document ingestion, AI agents, and 500,000+ emissions factor library.
- **API Documentation:** https://apitracker.io/a/watershedclimate (APITracker listing; direct developer portal requires enterprise agreement)
- **Developer Guide:** https://watershed.com/platform
- **SDKs / Libraries:** REST API; SDK details not publicly disclosed (enterprise access)
- **Standards Compliance:** REST/JSON, OpenAPI; GHG Protocol, CSRD, ISSB, GRI, CDP, TCFD aligned outputs
- **Authentication:** OAuth 2.0 / API key (enterprise)

### Persefoni

- **Description:** Carbon accounting and sustainability management platform with ledger-style data model, tiered pricing for organisations of all sizes, and PersefoniAI Copilot for guided carbon accounting.
- **API Documentation:** https://apitracker.io/a/persefoni (APITracker listing; full docs require account)
- **Integration Hub:** https://www.persefoni.com/business/carbon-footprint-measurement-analytics
- **SDKs / Libraries:** OpenAPI-based REST API; pre-built connectors for Coupa, Expensify, NetSuite, Stripe, SAP Concur, Google Sheets, Egencia, Navan
- **Workiva Integration:** https://support.workiva.com/hc/en-us/articles/7431656106132
- **Standards Compliance:** REST/JSON, OpenAPI; GHG Protocol, CSRD, ISSB, TCFD, GRI, SBTi, CDP aligned
- **Authentication:** OAuth 2.0

### Workiva

- **Description:** Unified SaaS platform connecting ESG, GRC, and financial reporting with deep audit trail and assurance-readiness, used by large enterprises and their assurance providers.
- **API Documentation:** https://developers.workiva.com/
- **API Overview:** https://developers.workiva.com/2022-01-01/overview.html
- **Sustainability Support:** https://support.workiva.com/hc/en-us/articles/4404020836244-ESG-Reporting-in-Workiva
- **SDKs / Libraries:** REST API; versioned by date (X-Version header, YYYY-MM-DD format); Go REST library open-sourced at https://github.com/Workiva/go-rest
- **Standards Compliance:** REST/JSON; CSRD/ESRS, GRI, IFRS S1/S2, TCFD, SEC climate rules; ESRS XBRL tagging supported
- **Authentication:** OAuth 2.0; enterprise SSO

### Salesforce Net Zero Cloud

- **Description:** ESG data management module within the Salesforce CRM platform, using Agentforce AI for automated report generation across CSRD, SASB, GRI, and CDP frameworks.
- **API Documentation:** https://developer.salesforce.com/ (Salesforce platform APIs)
- **Product Page:** https://www.salesforce.com/net-zero/cloud/
- **Integration Docs:** MuleSoft integration layer; REST and SOAP APIs via Salesforce platform
- **SDKs / Libraries:** Salesforce SDKs (JavaScript, Python, Java, Go, Ruby, Apex); MuleSoft connectors
- **Standards Compliance:** REST/JSON, SOAP, GraphQL (Salesforce Connect); CSRD, SASB, GRI, CDP report builders
- **Authentication:** OAuth 2.0 (Salesforce Connected App model)

### IBM Envizi ESG Suite

- **Description:** Modular ESG platform with 40,000+ emissions factors, intelligent factor selection, energy and utility analytics, and multi-framework reporting. Strong facilities/built-environment heritage.
- **API Documentation:** https://www.ibm.com/products/envizi/resources
- **AWS Marketplace Listing:** https://aws.amazon.com/marketplace/pp/prodview-4udtpnwf5g4cq
- **SDKs / Libraries:** IBM Cloud REST APIs; integration with IBM Maximo and IBM OpenScale
- **Standards Compliance:** REST/JSON; GHG Protocol, CSRD, CDP, GRI, SASB, TCFD aligned
- **Authentication:** IBM Cloud IAM (OAuth 2.0 / API key)

### Microsoft Sustainability Manager

- **Description:** Dynamics 365-based ESG platform covering environmental (GHG, water, waste), social, and governance metrics with Copilot AI for natural language queries and multi-framework reporting.
- **API Documentation:** https://learn.microsoft.com/en-us/industry/sustainability/sustainability-manager-overview
- **Developer Docs:** Microsoft Dataverse REST API (Power Platform)
- **SDKs / Libraries:** Dynamics 365 / Power Platform SDKs (JavaScript, .NET, Python); Azure Data Factory connectors; Microsoft Fabric integration
- **Standards Compliance:** REST/JSON (Dataverse OData); CSRD, ASRS, BRSR, GRI, IFRS S1/S2, SASB; XBRL export support
- **Authentication:** Azure Active Directory / OAuth 2.0 / OpenID Connect (Microsoft identity platform)

### CDP Disclosure API

- **Description:** CDP's official disclosure API enabling direct data transfer from sustainability software platforms into the CDP Portal for the annual climate, water, forests, and biodiversity questionnaires.
- **API Programme Page:** https://www.cdp.net/en/insights/cdp-disclosure-api-2026-streamline-your-disclosure
- **Data Access (Research):** https://www.cdp.net/en/data
- **SDKs / Libraries:** REST API; available to Accredited Solutions Providers (ASPs) only — requires CDP partnership
- **Standards Compliance:** REST/JSON; CDP questionnaire schema (proprietary); GHG Protocol aligned
- **Authentication:** API key / OAuth 2.0 (ASP-controlled)

---

## Notes

### Emissions Factor Database Interoperability

A significant gap in the market is the absence of an open, versioned, interoperable emissions factor database standard. Each major platform maintains a proprietary factor library. Initiatives worth monitoring:
- **ecoinvent** (https://ecoinvent.org/) — gold-standard LCA database; commercial licence required for most uses; accessible via Climatiq API
- **IPCC Emission Factor Database (EFDB)** (https://www.ipcc-nggip.iges.or.jp/EFDB/) — free, public; used for national GHG inventories; suitable as a public baseline
- **US EPA Emission Factors for GHG Inventories** (https://www.epa.gov/climateleadership/ghg-emission-factors-hub) — free; US-focused; annually updated
- **IEA Emission Factors** (https://www.iea.org/data-and-statistics/data-product/emissions-factors-2023) — country-level electricity grid factors; annual updates; commercial licence for commercial products

### Emerging Standards to Watch

- **ISSB nature-related disclosures** — ISSB agreed in April 2026 on developing an IFRS Practice Statement for nature-related disclosures under IFRS S1; formal standard forthcoming
- **TNFD mandatory requirements** — TNFD nature-related disclosures becoming mandatory in several jurisdictions from 2026; will require Scope 3 biodiversity and water data collection
- **EU CSDDD (Corporate Sustainability Due Diligence Directive)** — supply chain human rights and environmental due diligence obligations; creating demand for Scope 3 and supplier assessment features beyond GHG metrics
- **California SB 253 / SB 261 implementation** — Scope 1/2 reporting mandatory from 2026 filings; Scope 3 from 2027; TCFD-aligned climate risk reports from 2026 (SB 261, $500M+ revenue companies)
- **Open Footprint Forum API standard** — emerging interoperability standard for GHG data exchange; could become the ESG equivalent of Open Banking APIs if adopted widely
