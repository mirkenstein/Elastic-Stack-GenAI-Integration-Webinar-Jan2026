# Sample Kibana Queries for Medicare Provider Services

## Useful ES|QL Queries

### 1. Top providers by total Medicare payments
```esql
FROM medicare-provider-services-2021
| STATS total_payment = SUM(average_medicare_payment_amt * line_srvc_cnt),
        total_services = SUM(line_srvc_cnt)
  BY npi, nppes_provider_last_org_name, nppes_provider_state
| SORT total_payment DESC
| LIMIT 100
```

### 2. Service distribution by provider type
```esql
FROM medicare-provider-services-2021
| STATS 
    provider_count = COUNT_DISTINCT(npi),
    total_services = SUM(line_srvc_cnt),
    avg_payment = AVG(average_medicare_payment_amt)
  BY provider_type
| SORT total_services DESC
```

### 3. Geographic analysis - payments by state
```esql
FROM medicare-provider-services-2021
| STATS 
    providers = COUNT_DISTINCT(npi),
    total_beneficiaries = SUM(bene_unique_cnt),
    total_payment = SUM(average_medicare_payment_amt * line_srvc_cnt),
    avg_payment_per_service = AVG(average_medicare_payment_amt)
  BY nppes_provider_state
| EVAL payment_per_provider = total_payment / providers
| SORT total_payment DESC
```

### 4. Top HCPCS codes by volume
```esql
FROM medicare-provider-services-2021
| STATS 
    service_count = SUM(line_srvc_cnt),
    provider_count = COUNT_DISTINCT(npi),
    avg_payment = AVG(average_medicare_payment_amt)
  BY hcpcs_code, hcpcs_description
| SORT service_count DESC
| LIMIT 50
```

### 5. Payment variation analysis
```esql
FROM medicare-provider-services-2021
| WHERE hcpcs_code == "99213"  // Office visit code as example
| STATS 
    min_payment = MIN(average_medicare_payment_amt),
    max_payment = MAX(average_medicare_payment_amt),
    avg_payment = AVG(average_medicare_payment_amt),
    std_dev = SQRT(AVG(POW(average_medicare_payment_amt - AVG(average_medicare_payment_amt), 2))),
    provider_count = COUNT_DISTINCT(npi)
  BY nppes_provider_state
| EVAL coefficient_of_variation = std_dev / avg_payment * 100
| SORT coefficient_of_variation DESC
```

### 6. Find high-volume low-payment services (potential efficiency indicators)
```esql
FROM medicare-provider-services-2021
| WHERE line_srvc_cnt > 1000
| STATS 
    total_services = SUM(line_srvc_cnt),
    avg_payment = AVG(average_medicare_payment_amt),
    provider_count = COUNT_DISTINCT(npi)
  BY hcpcs_code, hcpcs_description
| WHERE avg_payment < 50
| SORT total_services DESC
| LIMIT 25
```

### 7. Provider specialization analysis
```esql
FROM medicare-provider-services-2021
| STATS 
    unique_services = COUNT_DISTINCT(hcpcs_code),
    total_services = SUM(line_srvc_cnt)
  BY npi, nppes_provider_last_org_name, provider_type
| EVAL services_per_code = total_services / unique_services
| WHERE unique_services > 5
| SORT services_per_code DESC
| LIMIT 100
```

### 8. Medicare vs. Submitted charge analysis
```esql
FROM medicare-provider-services-2021
| WHERE average_submitted_chrg_amt > 0
| EVAL 
    payment_ratio = average_medicare_payment_amt / average_submitted_chrg_amt * 100,
    discount_amount = average_submitted_chrg_amt - average_medicare_payment_amt
| STATS 
    avg_payment_ratio = AVG(payment_ratio),
    total_discount = SUM(discount_amount * line_srvc_cnt),
    service_count = SUM(line_srvc_cnt)
  BY provider_type
| SORT avg_payment_ratio ASC
```
