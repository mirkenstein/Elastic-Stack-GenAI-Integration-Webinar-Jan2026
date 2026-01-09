# Secure Scalable Integration of GenAI with Elastic Stack

**Healthcare Data Pipeline Architecture with Privacy-First LLM Integration**

---

## Overview

This repository contains demonstration materials showcasing secure, scalable integration patterns for combining Large Language Models (LLMs) with the Elastic Stack in healthcare environments. The focus is on **privacy-preserving architectures** that enable AI-powered data enrichment while maintaining compliance with healthcare regulations (HIPAA, GDPR).

> **⚠️ DEMO PURPOSE DISCLAIMER**
>
> These demonstrations use **publicly available data sources** for educational and reference purposes only:
> - [CMS Medicare Provider Services Dataset](https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service) (de-identified provider data)
> - [CMS RVU and RBCS Classification Files](https://www.cms.gov/medicare/medicare-fee-service-payment/physicianfeesched/pfs-relative-value-files) (public reference data)
> - [MTS-Dialog Medical Conversations](https://github.com/abachaa/MTS-Dialog) (research dataset)
>
> **No actual Protected Health Information (PHI) is used in these demos.** The architectural patterns shown are designed for secure production deployment with real PHI, but all examples utilize publicly accessible, de-identified datasets.

### Key Principles

- **Security First**: All architectures support on-premise, air-gapped deployment
- **Privacy by Design**: PHI never leaves your infrastructure
- **Scalability**: Horizontal scaling patterns for multi-million record datasets
- **Flexibility**: Support for cloud APIs, self-hosted models, and embedded ML
- **Production Ready**: Real-world patterns tested with [CMS Medicare Provider Services](https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service) datasets (9M+ records)

---

## Architecture Journey

This presentation follows a progressive architecture, building from traditional ETL to AI-powered enrichment:

```
1. Distributed Data Pipeline (Logstash)
   ↓
2. Enrichment at Ingest Time (Elasticsearch Enrich Policies)
   ↓
3. LLM Inference at Ingest Time (Elasticsearch Inference API)
   ↓
4. Secure Local LLM Integration (Same-Pod Architecture)
```

---

## Part 1: Distributed Data Pipeline with Logstash

**[📄 PARALLEL_PROCESSING_OVERVIEW.md](PARALLEL_PROCESSING_OVERVIEW.md)**

### Challenge
Processing 9+ million [Medicare provider service records](https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service) efficiently and reliably.

### Solution
Horizontal scaling with **Primary Key (PK) range partitioning** across multiple Logstash pods.

### Architecture Highlights
- **Multiple Logstash pods** process distinct PK ranges in parallel
- **Independent progress tracking** per pod enables fault tolerance
- **Linear scalability**: 4 pods = 4x throughput (~200K docs/sec)
- **Kubernetes-native**: Uses Elastic Cloud on Kubernetes (ECK) operator

### Key Features
- ✅ Resumable ingestion from last processed PK
- ✅ ConfigMap-driven range distribution
- ✅ Composite document IDs prevent duplicates
- ✅ Per-pod monitoring and observability

**Performance**: 9M records ingested in ~45 minutes with 4 parallel pods

---

## Part 2: Data Enrichment at Ingest Time

**[📄 HCPCS_CBSA_ENRICHMENT_DEMO.md](HCPCS_CBSA_ENRICHMENT_DEMO.md)**

### Concept
Enrich healthcare service records with authoritative lookup data **during ingestion** using Elasticsearch enrich policies.

### Data Sources (CMS & Census)
1. **RVU (Relative Value Units)** - Medicare payment calculations
2. **RBCS (Restructured BETOS)** - Clinical procedure classifications
3. **CBSA (Core Based Statistical Areas)** - Geographic metadata

### Benefits
- **Centralized enrichment logic** in Elasticsearch ingest pipeline
- **Consistent data quality** across all documents
- **Real-time enrichment** without application complexity
- **In-memory lookups** for fast processing

### Example Use Cases
```json
// Input: HCPCS code + ZIP code
{"hcpcs_code": "99214", "rndrng_prvdr_zip5": "98101"}

// Output: Enriched with clinical, financial, and geographic data
{
  "rvu_data": {"work_rvu": 1.50, "non_facility_payment": 120.30},
  "rbcs_data": {"rbcs_cat_desc": "Evaluation and Management"},
  "cbsa_data": {"cbsa_name": "Seattle-Tacoma-Bellevue, WA"}
}
```

---

## Part 3: LLM Inference at Ingest Time

**[📄 DEMO_INFERENCE.md](DEMO_INFERENCE.md)**

### Dataset
**MTS-Dialog (Medical Task-Oriented Dialog)** - Doctor-patient conversations with 20 clinical section categories.

### Architecture
Elasticsearch **Inference API** with ingest pipeline processor for automated LLM enrichment.

### Privacy Options

#### Option 1: Self-Hosted Models (Maximum Privacy)
```json
PUT _inference/completion/local-llm
{
  "service": "openai",
  "service_settings": {
    "url": "https://internal-llm.yourcompany.com/v1/chat/completions",
    "model_id": "medllama-finetuned"
  }
}
```
- **Private LLM endpoint** within your infrastructure
- **Fine-tuned medical models** (e.g., MedLLaMA, BioGPT)
- **Zero data egress** to external providers

#### Option 2: Elasticsearch ML Nodes (Embedded Models)
```json
PUT _ml/trained_models/clinical-bert-classifier
{
  "model_type": "pytorch",
  "inference_config": {
    "text_classification": {...}
  }
}
```
- **BERT models** running directly on Elasticsearch nodes
- **No external API calls** - all inference in-cluster
- **Fine-tuned domain models** for your specific usecase

#### Option 3: Cloud API with VPN/PrivateLink (Hybrid Security)
```json
PUT _inference/completion/claude-opus
{
  "service": "anthropic",
  "service_settings": {
    "api_key": "${ANTHROPIC_API_KEY}"
  }
}
```
- **Private connectivity** via AWS PrivateLink or Azure Private Endpoint
- **End-to-end encryption** in transit
- **Access via VPN tunnel** - no public internet exposure

### Use Case: Medical Conversation Classification
- **Categorizes** transcripts into 20 clinical section types
- **Generates** diagnostic summaries
- **Outputs** structured JSON for downstream analytics

### Advantages of Inference at Ingestion
1. **Centralized processing** - no preprocessing needed
2. **Consistent enrichment** - same model/prompt for all docs
3. **Real-time analysis** - insights available immediately
4. **Reduced complexity** - applications just index raw text
5. **Version control** - pipeline definitions auditable in Elasticsearch

---

## Part 4: Secure Local LLM Integration with Logstash

**[📄 SLIDES-ETL-ARCHITECTURE.md](SLIDES-ETL-ARCHITECTURE.md)**

### Architecture: Same-Pod LLM Service

The ultimate privacy-preserving pattern: **LLM and ETL engine in the same Kubernetes pod**.

```
┌─────────────────────────────────────┐
│  Kubernetes Job Pod                 │
│  ┌────────────────────────────────┐ │
│  │ Init: Model Loader (GPU)       │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌──────────────┐  localhost:11434  │
│  │ Local LLM    │◄───────────────┐  │
│  │ (Ollama/MAX) │                │  │
│  │ GPU: 1x A10  │                │  │
│  └──────────────┘                │  │
│                                   │  │
│  ┌────────────────────────────────┼┐ │
│  │ Logstash ETL Engine           ││ │
│  │ - Reads data                  ││ │
│  │ - Calls localhost:11434 ──────┘│ │
│  │ - Enriches with LLM            │ │
│  │ - Outputs results              │ │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
```

### Security Benefits

**Zero Network Exposure:**
- All communication via `localhost` within single pod
- LLM never exposed to cluster network
- **PHI never traverses network** - even internal

**Complete Air-Gap Capability:**
- No external API dependencies
- Models pre-loaded in init container
- **100% on-premise execution**

**Ephemeral Processing:**
- Job runs and auto-terminates
- No persistent LLM service
- **Data cleaned up after processing** (TTL-based)

### Compliance Advantages
- ✅ **HIPAA Compliant**: No PHI transmission to third parties
- ✅ **GDPR Ready**: Data processed entirely on-premise
- ✅ **SOC2 Friendly**: No external service dependencies
- ✅ **Auditable**: Complete execution logs within cluster

### Performance
- **GPU-accelerated**: ~200ms per record with NVIDIA A10/T4
- **Batch efficiency**: 100 records in ~30 seconds
- **Cost-effective**: Spot GPU instances (~$0.50/hour)

### Supported LLM Backends
- **Ollama**: Popular open-source models (Llama, Mistral, Gemma)
- **Modular MAX**: High-performance inference engine
- **vLLM**: Optimized for production serving
- **TensorRT-LLM**: NVIDIA-optimized inference

---

## End-to-End Example: Secure Medical Record Processing

### Scenario
Process 9M [Medicare provider service records](https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service) with:
1. Parallel ingestion from PostgreSQL
2. Enrichment with HCPCS/RVU/CBSA lookup data
3. LLM-based medical note summarization

### Architecture
```
PostgreSQL (9M records)
    ↓ (4 parallel Logstash pods)
Elasticsearch Cluster
    ↓ (Enrich pipeline: RVU + RBCS + CBSA)
    ↓ (Inference pipeline: Self-hosted MedLLaMA)
Enriched Index with AI Summaries
```

### Security Profile
- **Data sources**: Internal PostgreSQL (no cloud sync)
- **Enrichment**: Local Elasticsearch enrich indices
- **LLM inference**: Self-hosted model endpoint via VPN
- **Network**: All communication within private subnet
- **PHI handling**: Never leaves internal infrastructure

### Result
Fully enriched, AI-summarized medical records with **zero external data exposure**.

---

## Getting Started

### Prerequisites
- Kubernetes cluster with GPU nodes (for local LLM option)
- Elasticsearch cluster (version 8.11+)
- Logstash (version 8.11+)
- PostgreSQL or other JDBC-compatible data source

### Quick Start

1. **Deploy Parallel Logstash Pipeline**
```bash
./scripts/calculate-pk-ranges.sh 4  # For 4 pods
kubectl apply -k k8s/provider-services/
```

2. **Create Enrichment Policies**
```bash
# Execute commands from HCPCS_CBSA_ENRICHMENT_DEMO.md
```

3. **Setup LLM Inference**
```bash
# Option A: Self-hosted endpoint (recommended for production)
# Configure endpoint URL in inference pipeline

# Option B: Same-pod architecture
kubectl apply -f k8s/llm-etl-job.yaml
```

4. **Configure Ingest Pipelines**
```bash
# Execute commands from DEMO_INFERENCE.md
```

---

## Repository Structure

```
├── DEMO_INFERENCE.md              # LLM inference pipeline demo
├── HCPCS_CBSA_ENRICHMENT_DEMO.md  # Data enrichment demo
├── PARALLEL_PROCESSING_OVERVIEW.md # Parallel Logstash architecture
├── SLIDES-ETL-ARCHITECTURE.md     # Secure LLM-ETL integration
├── scripts/
│   └── calculate-pk-ranges.sh     # PK range calculator
└── k8s/
    ├── provider-services/         # Logstash deployment manifests
    └── llm-etl-job.yaml          # Same-pod LLM job
```

---

## Security Best Practices

### 1. Data in Transit
- Use TLS 1.3 for all connections
- Implement mutual TLS (mTLS) between services
- Consider AWS PrivateLink or Azure Private Endpoint for cloud APIs

### 2. Data at Rest
- Enable Elasticsearch encryption at rest
- Use encrypted PersistentVolumes for model storage
- Implement key rotation policies

### 3. Access Control
- Use Elasticsearch RBAC for fine-grained permissions
- Implement Kubernetes RBAC for pod access
- Rotate API keys regularly (if using external LLMs)

### 4. Network Isolation
- Deploy services in private subnets
- Use Network Policies to restrict pod-to-pod communication
- Implement egress filtering to prevent data exfiltration

### 5. Model Security
- Verify model checksums before loading
- Use signed container images for LLM services
- Implement model versioning and audit trails

### 6. PHI Handling
- Anonymize/de-identify data when possible
- Implement field-level access control
- Audit all access to sensitive fields
- Consider differential privacy techniques for aggregations

---

## Performance Tuning

### Logstash Parallel Processing
- **Pod count**: Scale based on table size (1 pod per 2-3M records)
- **Batch size**: 500-1000 rows per fetch
- **Worker threads**: Match CPU cores
- **Memory**: 2-4GB per pod

### Elasticsearch Enrichment
- **Enrich indices**: Keep lookup data updated
- **Cache size**: Monitor enrich cache hit rates
- **Bulk requests**: 500-1000 documents per bulk

### LLM Inference
- **GPU selection**: A10/T4 for balanced cost/performance
- **Batch size**: 8-16 for optimal GPU utilization
- **Model quantization**: Consider 4-bit/8-bit quantization
- **Context length**: Minimize prompt size for speed

---

## Use Cases

### Clinical Documentation
- Automated section classification (CC, HPI, Assessment, Plan)
- SOAP note generation from transcripts
- ICD-10/CPT code suggestion

### Healthcare Analytics
- Service utilization patterns by geography
- RVU-based cost analysis
- Procedure category trends

### Regulatory Compliance
- Automated quality reporting (MIPS, HEDIS)
- Risk adjustment coding validation
- Clinical documentation improvement (CDI)

### Research & Population Health
- Cohort identification from unstructured notes
- Outcome analysis with geographic correlation
- Social determinants of health (SDOH) extraction

---

## Contributing

This repository contains demonstration materials for educational and reference purposes. The patterns shown are production-tested but should be adapted to your specific security and compliance requirements.

### Feedback & Questions
- Open an issue for questions or clarifications
- Share your implementation experiences
- Suggest additional security patterns

---

## Resources

### Official Documentation
- [Elasticsearch Inference API](https://www.elastic.co/guide/en/elasticsearch/reference/current/inference-apis.html)
- [Elasticsearch Enrich Processor](https://www.elastic.co/guide/en/elasticsearch/reference/current/enrich-processor.html)
- [Logstash JDBC Input Plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html)
- [Elastic Cloud on Kubernetes (ECK)](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)

### Data Sources (Public Datasets for Demo)
- [CMS Medicare Provider Services](https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider-and-service) - De-identified provider service records
- [CMS RVU Files](https://www.cms.gov/medicare/medicare-fee-service-payment/physicianfeesched/pfs-relative-value-files) - Relative Value Units for payment calculations
- [CMS RBCS Classification](https://data.cms.gov/provider-summary-by-type-of-service/provider-service-classifications/restructured-betos-classification-system) - Procedure classification system
- [MTS-Dialog Dataset](https://github.com/abachaa/MTS-Dialog) - Medical conversation research dataset
- [Census TIGER/Line Shapefiles](https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html) - Geographic reference data

### Related Technologies
- [Ollama](https://ollama.ai/) - Local LLM serving
- [vLLM](https://github.com/vllm-project/vllm) - High-throughput inference
- [Hugging Face Transformers](https://huggingface.co/docs/transformers) - Model library
- [Kubernetes](https://kubernetes.io/) - Container orchestration

---

## License

These materials are provided for educational and demonstration purposes. Please ensure compliance with all applicable healthcare data regulations (HIPAA, GDPR, etc.) when implementing in production environments.

---

## Summary

This repository demonstrates a **comprehensive, security-first approach** to integrating Large Language Models with healthcare data pipelines using the Elastic Stack. By combining:

- **Scalable distributed processing** (parallel Logstash)
- **Enrichment at ingestion** (Elasticsearch enrich policies)
- **AI-powered analysis** (inference processors)
- **Privacy-preserving architecture** (self-hosted models, same-pod execution)

Organizations can build powerful AI-driven healthcare analytics systems while maintaining complete control over Protected Health Information (PHI) and meeting stringent regulatory requirements.

**Key Takeaway**: You can leverage cutting-edge LLM capabilities without sacrificing data privacy or security by choosing the right architectural patterns and deployment models.
