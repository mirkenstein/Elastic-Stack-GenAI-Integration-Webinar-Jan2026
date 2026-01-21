# HCPCS and Geographic Enrichment Pipeline Demo

## Overview

This demo demonstrates how to enrich [Medicare provider service data](https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service) with HCPCS procedure information (RVU and RBCS classifications) and geographic data (CBSA and county names) using Elasticsearch enrich policies and ingest pipelines.

> **Demo Purpose**: This demonstration uses **publicly available, de-identified data sources** from CMS and U.S. Census Bureau for educational purposes only.

---

## Data Sources

This enrichment pipeline integrates three authoritative CMS and Census data sources:

### 1. RVU (Relative Value Unit) Data
**Relative Value Units** are the basis for Medicare physician fee schedule payment amounts, representing the relative resources and complexity of medical procedures.

**Source**: [CMS Physician Fee Schedule - RVU Files (2023)](https://www.cms.gov/medicare/medicare-fee-service-payment/physicianfeesched/pfs-relative-value-files/rvu23a)

**Fields**: Year, description, work RVU, practice expense RVU (facility/non-facility), malpractice RVU, total RVU, global period, conversion factor, and national payment amounts.

### 2. RBCS (Restructured BETOS Classification System) Data
**RBCS** is a classification system that groups healthcare procedure codes into clinically meaningful categories for analysis and reporting.

**Source**: [CMS Restructured BETOS Classification System](https://data.cms.gov/provider-summary-by-type-of-service/provider-service-classifications/restructured-betos-classification-system)

**Fields**: RBCS ID, category, subcategory, family descriptions, major indicator, date ranges, and assignment history.

### 3. CBSA (Core Based Statistical Area) and County Data
**CBSA data** links ZIP codes to metropolitan and micropolitan statistical areas and their corresponding counties for geographic analysis.

**Source**: Derived from [U.S. Census Bureau TIGER/Line Shapefiles](https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html)

**Fields**: ZIP code (ZCTA), CBSA name, county name.

---

## Benefits of Combined Enrichment

Enriching provider service data with all three sources enables:
- **Clinical Analysis**: Understand procedure complexity and clinical categories
- **Payment Analysis**: Calculate expected reimbursements based on RVUs
- **Geographic Analysis**: Analyze service patterns by metropolitan area and county
- **Comprehensive Reporting**: Combine clinical, financial, and geographic dimensions

---

## Implementation

For complete setup instructions including:
- Creating lookup indices
- Creating and executing enrich policies
- Building the ingest pipeline
- Testing with sample data

See the detailed guide: **[es_ingest_pipelines/enrich/README.md](es_ingest_pipelines/enrich/README.md)**

---

## Use Cases

### Clinical Analysis
Query procedures by RBCS category to analyze service mix across different clinical classifications (E&M, procedures, imaging, tests, etc.).

### Financial Analysis
Calculate average RVU values by metropolitan area to understand geographic variations in procedure complexity and reimbursement.

### Geographic Analysis
Find services provided in specific counties or CBSAs to analyze regional healthcare utilization patterns.

---

## Example Enriched Document

After enrichment, documents contain three nested enrichment fields:

| Field | Description |
|-------|-------------|
| `rvu_data` | RVU values, payment amounts, and procedure description |
| `rbcs_data` | RBCS category, subcategory, and family classifications |
| `cbsa_data` | CBSA metropolitan area name and county name |

---

## Notes

- **ZIP Codes**: Test examples use ZIP codes from major U.S. metropolitan areas
- **HCPCS Codes**: Cover diverse procedure types (E&M visits, imaging, surgery)
- **Fault Tolerance**: All enrichments use `ignore_missing` and `ignore_failure` to ensure documents are indexed even if enrichment data is unavailable
- **Performance**: Enrich policies create in-memory indices for fast lookup during ingestion