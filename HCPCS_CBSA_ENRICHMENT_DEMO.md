# HCPCS and Geographic Enrichment Pipeline Demo

## Overview

This demo demonstrates how to enrich [Medicare provider service data](https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service) with HCPCS procedure information (RVU and RBCS classifications) and geographic data (CBSA and county names) using Elasticsearch enrich policies and ingest pipelines.

> **⚠️ Demo Purpose**: This demonstration uses **publicly available, de-identified data sources** from CMS and U.S. Census Bureau for educational purposes only.

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

## Demo Steps for Kibana Dev Console

### Step 1: Verify Lookup Indices

First, verify that the three lookup indices exist and contain data.

```json
GET zip_county_cbsa_lookup/_search
{
  "size": 1
}
```

```json
GET hcpcs_rvu_lookup/_search
{
  "size": 1
}
```

```json
GET hcpcs_rbcs_lookup/_search
{
  "size": 1
}
```

**Expected Result**: Each query should return at least one document from the respective lookup index.

---

### Step 2: Create ZIP/CBSA Enrich Policy

Create an enrich policy to add CBSA and county information based on ZIP code.

```json
PUT /_enrich/policy/zip_cbsa_policy
{
  "match": {
    "indices": "zip_county_cbsa_lookup",
    "match_field": "zcta5ce20",
    "enrich_fields": [
      "cbsa_name",
      "county_name"
    ]
  }
}
```

**Expected Result**: `{"acknowledged": true}`

---

### Step 3: Execute ZIP/CBSA Enrich Policy

Execute the policy to build the enrich index.

```json
POST /_enrich/policy/zip_cbsa_policy/_execute
```

**Expected Result**: Execution status showing the policy was executed successfully.

---

### Step 4: Create HCPCS RVU Enrich Policy

Create an enrich policy to add Relative Value Unit data based on HCPCS code.

```json
PUT /_enrich/policy/hcpcs_rvu_policy
{
  "match": {
    "indices": "hcpcs_rvu_lookup",
    "match_field": "hcpcs_cd",
    "enrich_fields": [
      "year",
      "description",
      "status_code",
      "is_active",
      "work_rvu",
      "non_facility_pe_rvu",
      "facility_pe_rvu",
      "malpractice_rvu",
      "non_facility_total_rvu",
      "facility_total_rvu",
      "global_period",
      "pc_tc_indicator",
      "conversion_factor",
      "non_facility_national_payment",
      "facility_national_payment"
    ]
  }
}
```

**Expected Result**: `{"acknowledged": true}`

---

### Step 5: Execute HCPCS RVU Enrich Policy

Execute the RVU policy to build the enrich index.

```json
POST /_enrich/policy/hcpcs_rvu_policy/_execute
```

**Expected Result**: Execution status showing the policy was executed successfully.

---

### Step 6: Create HCPCS RBCS Enrich Policy

Create an enrich policy to add RBCS classification data based on HCPCS code.

```json
PUT /_enrich/policy/hcpcs_rbcs_policy
{
  "match": {
    "indices": "hcpcs_rbcs_lookup",
    "match_field": "hcpcs_cd",
    "enrich_fields": [
      "rbcs_id",
      "rbcs_cat",
      "rbcs_cat_desc",
      "rbcs_cat_subcat",
      "rbcs_subcat_desc",
      "rbcs_famnumb",
      "rbcs_family_desc",
      "rbcs_major_ind",
      "hcpcs_cd_add_dt",
      "hcpcs_cd_end_dt",
      "rbcs_latest_assignment",
      "first_rbcs_release_year",
      "rbcs_analysis_start_dt",
      "rbcs_analysis_end_dt",
      "alt_assignment_method",
      "rbcs_id_ever_reassigned"
    ]
  }
}
```

**Expected Result**: `{"acknowledged": true}`

---

### Step 7: Execute HCPCS RBCS Enrich Policy

Execute the RBCS policy to build the enrich index.

```json
POST /_enrich/policy/hcpcs_rbcs_policy/_execute
```

**Expected Result**: Execution status showing the policy was executed successfully.

---

### Step 8: Create Combined Enrichment Ingest Pipeline

Create the ingest pipeline that applies all three enrichments to incoming documents.

```json
PUT _ingest/pipeline/hcpcs_enrichment
{
  "description": "Enrich HCPCS codes with RVU and RBCS data, and add geographic CBSA/county information",
  "processors": [
    {
      "enrich": {
        "description": "Add RVU data for HCPCS code",
        "policy_name": "hcpcs_rvu_policy",
        "field": "hcpcs_code",
        "target_field": "rvu_data",
        "max_matches": "1",
        "ignore_missing": true,
        "ignore_failure": true
      }
    },
    {
      "enrich": {
        "description": "Add RBCS classification for HCPCS code",
        "policy_name": "hcpcs_rbcs_policy",
        "field": "hcpcs_code",
        "target_field": "rbcs_data",
        "max_matches": "1",
        "ignore_missing": true,
        "ignore_failure": true
      }
    },
    {
      "enrich": {
        "description": "Add CBSA and county data for ZIP code",
        "policy_name": "zip_cbsa_policy",
        "field": "rndrng_prvdr_zip5",
        "target_field": "cbsa_data",
        "max_matches": "1",
        "ignore_missing": true,
        "ignore_failure": true
      }
    }
  ]
}
```

**Expected Result**: `{"acknowledged": true}`

---

### Step 9: Test the Pipeline with Sample Data

Create a test index and ingest sample documents with the enrichment pipeline.

#### Test Document 1: Infliximab Injection in Chicago

```json
POST /demo-enriched-services/_doc?pipeline=hcpcs_enrichment
{
  "hcpcs_code": "J1745",
  "rndrng_prvdr_zip5": "60601"
}
```

**HCPCS J1745**: Infliximab injection (biological medication)
**ZIP 60601**: Chicago, IL

---

#### Test Document 2: Dental Procedure in New York

```json
POST /demo-enriched-services/_doc?pipeline=hcpcs_enrichment
{
  "hcpcs_code": "D5987",
  "rndrng_prvdr_zip5": "11368"
}
```

**HCPCS D5987**: Commissure splint (dental procedure)
**ZIP 11368**: Queens, New York

---

#### Test Document 3: Annual Wellness Visit in Beverly Hills

```json
POST /demo-enriched-services/_doc?pipeline=hcpcs_enrichment
{
  "hcpcs_code": "G0438",
  "rndrng_prvdr_zip5": "90210"
}
```

**HCPCS G0438**: Annual wellness visit (preventive care)
**ZIP 90210**: Beverly Hills, CA

---

#### Test Document 4: Office Visit in Seattle

```json
POST /demo-enriched-services/_doc?pipeline=hcpcs_enrichment
{
  "hcpcs_code": "99214",
  "rndrng_prvdr_zip5": "98101"
}
```

**HCPCS 99214**: Office visit, established patient, moderate complexity
**ZIP 98101**: Seattle, WA

---

### Step 10: Query Enriched Documents

Retrieve and examine the enriched documents.

```json
GET /demo-enriched-services/_search
{
  "query": {
    "match_all": {}
  }
}
```

**Expected Result**: Documents containing the original fields plus three nested enrichment fields:
- `rvu_data`: RVU values, payment amounts, and procedure description
- `rbcs_data`: RBCS category, subcategory, and family classifications
- `cbsa_data`: CBSA name and county name for the ZIP code

---

## Example Enriched Document Structure

After enrichment, documents will have this structure:

```json
{
  "hcpcs_code": "99214",
  "rndrng_prvdr_zip5": "98101",
  "rvu_data": {
    "year": 2023,
    "description": "Office outpatient visit 20-29 minutes",
    "work_rvu": 1.50,
    "non_facility_pe_rvu": 1.95,
    "facility_pe_rvu": 0.90,
    "malpractice_rvu": 0.10,
    "non_facility_total_rvu": 3.55,
    "facility_total_rvu": 2.50,
    "non_facility_national_payment": 120.30,
    "facility_national_payment": 84.72
  },
  "rbcs_data": {
    "rbcs_id": "EM001N",
    "rbcs_cat": "E",
    "rbcs_cat_desc": "Evaluation and Management",
    "rbcs_subcat_desc": "Office Visits",
    "rbcs_family_desc": "Established Patient Office Visit"
  },
  "cbsa_data": {
    "cbsa_name": "Seattle-Tacoma-Bellevue, WA",
    "county_name": "King County"
  }
}
```

---

## Use Cases

### Clinical Analysis
Query procedures by RBCS category to analyze service mix:
```json
GET /demo-enriched-services/_search
{
  "query": {
    "term": {
      "rbcs_data.rbcs_cat": "E"
    }
  }
}
```

### Financial Analysis
Calculate average RVU values by metropolitan area:
```json
GET /demo-enriched-services/_search
{
  "size": 0,
  "aggs": {
    "by_cbsa": {
      "terms": {
        "field": "cbsa_data.cbsa_name.keyword"
      },
      "aggs": {
        "avg_work_rvu": {
          "avg": {
            "field": "rvu_data.work_rvu"
          }
        }
      }
    }
  }
}
```

### Geographic Analysis
Find services provided in specific counties:
```json
GET /demo-enriched-services/_search
{
  "query": {
    "match": {
      "cbsa_data.county_name": "King County"
    }
  }
}
```

---

## Production Deployment

### Set as Default Pipeline

Apply the enrichment pipeline automatically to all new documents:

```json
PUT /medicare-provider-services-*/_settings
{
  "index.default_pipeline": "hcpcs_enrichment"
}
```

### Refresh Enrich Policies

When lookup data is updated, re-execute the enrich policies:

```json
POST /_enrich/policy/hcpcs_rvu_policy/_execute
POST /_enrich/policy/hcpcs_rbcs_policy/_execute
POST /_enrich/policy/zip_cbsa_policy/_execute
```

---

## Cleanup (Optional)

To clean up the demo resources:

```json
DELETE /demo-enriched-services
DELETE /_ingest/pipeline/hcpcs_enrichment
DELETE /_enrich/policy/hcpcs_rvu_policy
DELETE /_enrich/policy/hcpcs_rbcs_policy
DELETE /_enrich/policy/zip_cbsa_policy
```

---

## Notes

- **ZIP Codes**: The test examples use ZIP codes from major U.S. metropolitan areas (Chicago, New York, Beverly Hills, Seattle)
- **HCPCS Codes**: Cover diverse procedure types (injections, dental, preventive care, office visits)
- **Fault Tolerance**: All enrichments use `ignore_missing` and `ignore_failure` to ensure documents are indexed even if enrichment data is unavailable
- **Performance**: Enrich policies create in-memory indices for fast lookup during ingestion
