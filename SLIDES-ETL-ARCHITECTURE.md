# LLM-Powered ETL Pipeline - Architecture Overview

> **⚠️ Demo Purpose**: This architecture demonstration showcases secure patterns for healthcare data processing. Examples use publicly available datasets for educational purposes only.

## Slide 1: Secure Same-Pod Architecture

```
┌─────────────────────────────────────────────┐
│  Kubernetes Job Pod (Production GPU)        │
│  ┌────────────────────────────────────────┐ │
│  │ Init Container: Model Loader           │ │
│  │ - Pre-loads model (e.g., llama3)       │ │
│  │ - GPU: 1x NVIDIA (A10/T4)              │ │
│  │ - Exits after model ready              │ │
│  └────────────────────────────────────────┘ │
│                                              │
│  ┌──────────────────┐  localhost:11434      │
│  │ Local LLM        │◄──────────────────┐   │
│  │ Service          │                   │   │
│  │                  │                   │   │
│  │ GPU: 1x NVIDIA   │                   │   │
│  │ RAM: 12-16Gi     │                   │   │
│  │                  │                   │   │
│  │ (Ollama/Modular  │                   │   │
│  │  MAX GPU)        │                   │   │
│  └──────────────────┘                   │   │
│                                          │   │
│  ┌───────────────────────────────────────┼─┐│
│  │ Logstash ETL Engine                  │ │││
│  │                                       │ │││
│  │ CPU: 2-4 cores                        │ │││
│  │ RAM: 2-4Gi                            │ │││
│  │                                       │ │││
│  │ - Reads input data                    │ │││
│  │ - Calls localhost:11434 ──────────────┘ ││
│  │ - Enriches data with LLM              │  │
│  │ - Outputs results                     │  │
│  │ - Auto-terminates on completion       │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  Total: 1 GPU + 6 CPU + 16Gi RAM            │
└─────────────────────────────────────────────┘
```

**Key Components:**
- **Init Container**: Pre-loads ML model before ETL starts
- **LLM Service**: Local inference engine (no external API calls)
- **ETL Engine**: Processes data with LLM enrichment
- **Communication**: Internal localhost only

---

## Slide 2: Security & Performance Benefits

### 🔒 Security Advantages

**Zero Network Exposure:**
- All communication via `localhost:<port>` within single pod
- No service mesh, no network policies needed
- LLM never exposed to cluster network
- Data never leaves the pod

**No External Dependencies:**
- No API keys or cloud LLM services
- Data processed entirely on-premise
- Complete air-gap capability
- Compliance-friendly (GDPR, HIPAA, SOC2)

**Ephemeral Execution:**
- Job runs and auto-terminates
- No persistent LLM service
- Data cleaned up after TTL expiry

### ⚡ Performance

**Production (GPU):**
- Inference: ~200ms per record
- 100 records: ~30 seconds
- Total job time: ~2 minutes

**Cost Efficiency:**
- Spot GPU instances: ~$0.50/hour
- Only runs when needed (batch jobs)
- No always-on LLM service costs

---

## Slide 3: Execution Flow

### Simple 4-Step Process

```
1. Deploy → 2. Process → 3. Extract → 4. Cleanup
```

**1. Deploy Job**
```bash
kubectl apply -f etl-job.yaml
```

**2. Automatic Processing**
- Init container downloads model (~2 min)
- LLM service starts on localhost
- Logstash processes all records
- Job terminates on completion

**3. Extract Results**
```bash
kubectl logs job/llm-etl -c logstash > results.json
```

**4. Auto-Cleanup**
- Job auto-deletes after 1 hour (configurable TTL)
- All resources released

### Use Cases

- **Sentiment Analysis**: Customer feedback classification
- **Data Enrichment**: Adding semantic tags/categories
- **Content Moderation**: Automated content screening
- **Entity Extraction**: NLP processing at scale
- **Data Transformation**: AI-powered ETL pipelines

---

## Key Takeaways

✅ **Secure by Design**: Localhost-only, no network exposure
✅ **Production Ready**: GPU-accelerated for real workloads
✅ **Flexible**: Supports multiple LLM backends (Ollama, Modular MAX)
✅ **Cost Effective**: Batch jobs on spot instances
✅ **Compliant**: On-premise, no data egress