# Elasticsearch Ingest Pipelines

This directory contains Elasticsearch ingest pipeline configurations for enriching Medicare provider data with HCPCS, RVU, RBCS, and geographic (CBSA/County) information.

## HCPCS and Geographic Enrichment

The enrichment pipeline adds:
- **RVU (Relative Value Unit)** data - Medicare payment calculations and procedure complexity
- **RBCS (Restructured BETOS Classification System)** data - Clinical categorization of procedures
- **CBSA and County** data - Geographic information based on ZIP codes

### Setup

#### Step 1: Create Lookup Indices

Create three lookup indices from Kibana from file upload.
Optionally each index can be configured with `"mode": "lookup"` to support ES|QL JOIN operations (for future demos).


In the customize index settings UI add this.
```json
{
  "index": {
    "mode": "lookup"
  }
}
```
##### 1. ZIP/CBSA/County Lookup Index

```json
PUT zip_county_cbsa_lookup
{
  "settings": {
    "index": {
      "mode": "lookup"
    }
  },
  "mappings": {
    "properties": {
      "zcta5ce20": { "type": "keyword" },
      "cbsa_name": { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "county_name": { "type": "text", "fields": { "keyword": { "type": "keyword" } } }
    }
  }
}
```

##### 2. HCPCS RVU Lookup Index

```json
PUT hcpcs_rvu_lookup
{
  "settings": {
    "index": {
      "mode": "lookup"
    }
  },
  "mappings": {
    "properties": {
      "hcpcs_cd": { "type": "keyword" },
      "year": { "type": "integer" },
      "description": { "type": "text" },
      "status_code": { "type": "keyword" },
      "is_active": { "type": "boolean" },
      "work_rvu": { "type": "float" },
      "non_facility_pe_rvu": { "type": "float" },
      "facility_pe_rvu": { "type": "float" },
      "malpractice_rvu": { "type": "float" },
      "non_facility_total_rvu": { "type": "float" },
      "facility_total_rvu": { "type": "float" },
      "global_period": { "type": "keyword" },
      "pc_tc_indicator": { "type": "keyword" },
      "conversion_factor": { "type": "float" },
      "non_facility_national_payment": { "type": "float" },
      "facility_national_payment": { "type": "float" }
    }
  }
}
```

##### 3. HCPCS RBCS Lookup Index

```json
PUT hcpcs_rbcs_lookup
{
  "settings": {
    "index": {
      "mode": "lookup"
    }
  },
  "mappings": {
    "properties": {
      "hcpcs_cd": { "type": "keyword" },
      "rbcs_id": { "type": "keyword" },
      "rbcs_cat": { "type": "keyword" },
      "rbcs_cat_desc": { "type": "text" },
      "rbcs_cat_subcat": { "type": "keyword" },
      "rbcs_subcat_desc": { "type": "text" },
      "rbcs_famnumb": { "type": "keyword" },
      "rbcs_family_desc": { "type": "text" },
      "rbcs_major_ind": { "type": "keyword" },
      "hcpcs_cd_add_dt": { "type": "date" },
      "hcpcs_cd_end_dt": { "type": "date" },
      "rbcs_latest_assignment": { "type": "keyword" },
      "first_rbcs_release_year": { "type": "integer" },
      "rbcs_analysis_start_dt": { "type": "date" },
      "rbcs_analysis_end_dt": { "type": "date" },
      "alt_assignment_method": { "type": "keyword" },
      "rbcs_id_ever_reassigned": { "type": "keyword" }
    }
  }
}
```

**Note**: The `"mode": "lookup"` setting enables these indices for ES|QL JOIN operations. This is not currently used in this demo but may be utilized in future demonstrations.

#### Step 2: Load Data into Lookup Indices

After creating the indices, load the data from the CSV files:
- `zip_county_cbsa_lookup` ← Load from ZIP/CBSA/County data source
- `hcpcs_rvu_lookup` ← Load from `hcpcs_rvu_lookup_all.csv`
- `hcpcs_rbcs_lookup` ← Load from `RBCS_RY_2025.csv`

#### Step 3: Create and Execute Enrich Policies

##### 1. ZIP/CBSA Enrichment Policy

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

POST _enrich/policy/zip_cbsa_policy/_execute
```

##### 2. RVU Enrichment Policy

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

POST _enrich/policy/hcpcs_rvu_policy/_execute
```

##### 3. RBCS Enrichment Policy

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

POST /_enrich/policy/hcpcs_rbcs_policy/_execute
```

#### Step 4: Create the Ingest Pipeline

```json
PUT _ingest/pipeline/hcpcs_enrichment
{
  "description": "Enrich HCPCS codes with RVU and RBCS data",
  "processors": [
    {
      "enrich": {
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

#### Step 5: Test with Sample Data

Test the enrichment pipeline with different procedure types and geographic locations:

**E&M Office Visit (99215)** - Level 4 established patient office visit, typically 30-39 minutes. Provider located in Corona, Queens, NY (New York-Newark-Jersey City metro area).

```json
POST /your-index/_doc?pipeline=hcpcs_enrichment
{
  "hcpcs_cd": "99215",
  "rndrng_prvdr_zip5": "11368"
}
```

**Chest X-ray (71046)** - Two-view chest radiograph (frontal and lateral). Provider located in downtown Chicago, IL (Chicago-Naperville-Elgin metro area).

```json
POST /your-index/_doc?pipeline=hcpcs_enrichment
{
  "hcpcs_cd": "71046",
  "rndrng_prvdr_zip5": "60601"
}
```

**CABG Surgery (33533)** - Coronary artery bypass graft using arterial conduit (single graft). Provider located in Beverly Hills, CA (Los Angeles-Long Beach-Anaheim metro area).

```json
POST /your-index/_doc?pipeline=hcpcs_enrichment
{
  "hcpcs_cd": "33533",
  "rndrng_prvdr_zip5": "90210"
}
```

**View enriched documents:**

```json
GET your-index/_search
```

**Compare RVU complexity across procedures** - Query only the HCPCS code and facility total RVU to see the relative complexity/reimbursement weight:

```json
GET your-index/_search
{"_source": ["hcpcs_cd","rvu_data.facility_total_rvu"]}
```

Example output showing the dramatic difference in RVU values:

| HCPCS Code | Procedure | Facility Total RVU |
|------------|-----------|-------------------|
| 99215 | E&M Office Visit | 4.23 |
| 71046 | Chest X-ray | 1.01 |
| 33533 | CABG Surgery | 54.90 |

The RVU values reflect procedure complexity and resource intensity. A CABG surgery (54.90 RVUs) requires significantly more physician work, time, and skill compared to reading a chest X-ray (1.01 RVUs).

For additional details, see [HCPCS_CBSA_ENRICHMENT_DEMO.md](../../HCPCS_CBSA_ENRICHMENT_DEMO.md)


---

## Usage

### Apply Pipeline Per Request

```json
POST /medicare-provider-services-2023/_doc?pipeline=hcpcs_enrichment
{
  "hcpcs_code": "99213",
  "rndrng_prvdr_zip5": "98101",
  "npi": "1234567890"
}
```

### Set as Default Pipeline for an Index

```json
PUT /medicare-provider-services-2023/_settings
{
  "index.default_pipeline": "hcpcs_enrichment"
}
```

### Configure in Logstash

In your Logstash output configuration:

```ruby
output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "medicare-provider-services-%{year}"
    pipeline => "hcpcs_enrichment"
  }
}
```

---

## Enriched Data Structure

After enrichment, documents will have three additional nested fields:

```json
{
  "hcpcs_code": "99214",
  "rndrng_prvdr_zip5": "98101",
  "npi": "1234567890",
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

## Updating Enrich Policies

When lookup data is updated, re-execute the enrich policies:

```json
POST /_enrich/policy/hcpcs_rvu_policy/_execute
POST /_enrich/policy/hcpcs_rbcs_policy/_execute
POST /_enrich/policy/zip_cbsa_policy/_execute
```

---

## Troubleshooting

### Check Enrich Policy Status

```json
GET /_enrich/policy/hcpcs_rvu_policy
GET /_enrich/policy/hcpcs_rbcs_policy
GET /_enrich/policy/zip_cbsa_policy
```

### Test the Pipeline

```json
POST /_ingest/pipeline/hcpcs_enrichment/_simulate
{
  "docs": [
    {
      "_source": {
        "hcpcs_code": "99214",
        "rndrng_prvdr_zip5": "98101"
      }
    }
  ]
}
```

### View Pipeline Configuration

```json
GET /_ingest/pipeline/hcpcs_enrichment
```

---

## Additional Resources

For a complete step-by-step demo with sample data and expected results, see:
- [HCPCS_CBSA_ENRICHMENT_DEMO.md](../../HCPCS_CBSA_ENRICHMENT_DEMO.md)