# Sustainability & ESG Reporting

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source platform that centralises carbon accounting, multi-framework ESG disclosure, and Scope 3 supplier data collection so organisations can meet CSRD, ISSB, GRI, and SEC climate requirements from a single dataset.

Sustainability reporting has moved from voluntary initiative to regulatory obligation. Organisations now face overlapping mandates -- the EU's CSRD, SEC climate rules, California SB 253/261, and global frameworks like GRI, SASB, and ISSB -- each demanding similar but non-identical data. This project provides an AI-native, open-source platform for GHG accounting across Scope 1, 2, and 3, automated framework mapping, supplier engagement, and audit-ready disclosure generation, replacing the spreadsheets and six-figure SaaS contracts that dominate the market today.

---

## Why Sustainability & ESG Reporting?

- **Enterprise pricing locks out the mid-market.** Incumbent platforms (Sweep, Watershed, Persefoni, IBM Envizi) price at $50k--$200k/yr, yet the organisations that generate most Scope 3 emissions -- mid-market suppliers -- cannot afford them.
- **Proprietary emissions factor libraries create lock-in.** Platforms like Watershed (500,000+ factors) and IBM Envizi (40,000+ factors) use proprietary, non-portable datasets. Cross-platform auditing is impractical, and switching costs are high.
- **Scope 3 data collection remains manual and annual.** Most platforms rely on annual supplier surveys rather than automated, transaction-level calculation from procurement data. Real-time Scope 3 tracking is largely unsolved.
- **No platform covers the full ESG data model well.** Carbon-centric tools (Persefoni, Normative, Greenly) lack social and governance data collection; finance-centric tools (Workiva) lack GHG calculation depth. An integrated E, S, and G data model remains underserved.
- **AI-assisted materiality assessment is missing.** CSRD requires double materiality assessments, yet no incumbent automates this process meaningfully.

---

## Key Features

### GHG Calculation Engine

- Scope 1 and 2 GHG calculation with configurable emission factors aligned to GHG Protocol methodology
- Scope 3 calculation using spend-based and activity-based methods across all fifteen categories
- Emission factor library with versioning, update notifications, and transparent sourcing (IPCC, IEA, GHG Protocol)
- Organisational boundary configuration supporting operational control and equity share approaches

### Multi-Framework Reporting

- Single-dataset mapping to GRI, ISSB/TCFD, CSRD (ESRS), CDP, SB 253, and SEC climate rules
- Framework crosswalk tools that reduce duplication when satisfying multiple standards simultaneously
- Automated ESRS double materiality assessment templates
- Report export in audit-ready formats with embedded data lineage

### Supplier Engagement & Scope 3

- Self-service supplier portal for Scope 3 Category 1 and Category 11 data collection
- Automated reminder workflows, data quality scoring, and gap analysis
- Supplier response tracking with status dashboards for procurement teams
- Future: real-time API-based Scope 3 calculation from procurement transaction data

### AI-Powered Analytics

- Document ingestion via OCR and LLM classification for utility bills, invoices, and supplier documents
- NLP-based emissions factor matching mapping activity descriptions to correct GHG Protocol categories
- Anomaly and error detection flagging outliers in emissions data before submission
- AI Copilot for natural language data queries and methodology guidance

### Audit Trail & Governance

- Full data lineage from source documents to published disclosures
- Role-based access control with four-eyes approval workflows
- Version control and change tracking across reporting periods
- Assurance-ready data exports supporting limited and reasonable assurance engagements

---

## AI-Native Advantage

AI transforms ESG reporting from a compliance burden into a data intelligence capability. LLM-powered document ingestion eliminates the manual data entry bottleneck -- classifying utility bills, invoices, and supplier submissions automatically. NLP-based emissions factor matching reduces methodology errors by mapping free-text activity descriptions to the correct GHG Protocol categories and emission factors. Anomaly detection surfaces data quality issues before they reach auditors, and AI-generated narrative drafting produces CSRD/GRI disclosure responses grounded in structured data rather than starting from blank pages.

---

## Tech Stack & Deployment

Deployment targets include self-hosted (Docker/Kubernetes), cloud-hosted, and hybrid configurations. The GHG calculation engine builds on publicly available methodologies (GHG Protocol, IPCC emission factors) and open emissions factor databases where licences permit. Data ingestion supports CSV import, ERP connectors (SAP, NetSuite), and REST APIs for custom integrations. The supplier portal is web-based with no software required on the supplier side.

---

## Market Context

The ESG software market is consolidating around regulatory mandates, with ISSB absorbing TCFD and SASB into IFRS S1/S2 as a global reporting baseline. Enterprise incumbents -- Sweep, Watershed, Persefoni, Workiva, IBM Envizi -- command premium pricing ($50k--$200k/yr), leaving a large mid-market segment underserved. Primary buyers are sustainability teams, finance and compliance departments, and procurement organisations responsible for Scope 3 supplier engagement.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
