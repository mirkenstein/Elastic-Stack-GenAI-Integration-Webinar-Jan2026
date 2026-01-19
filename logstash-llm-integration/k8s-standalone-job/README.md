# Logstash + Ollama ETL Jobs (Standalone)

Run LLM-powered ETL pipelines on Kubernetes using Logstash and Ollama. No complex tooling required - just `kubectl apply`.

## What's Included

| File | Description |
|------|-------------|
| `00-namespace.yaml` | Creates the `ollama` namespace |
| `01-storage.yaml` | Persistent storage for Ollama models (download once, reuse) |
| `demo.yaml` | Quick demo: classifies 5 sample sentences |
| `mediqa.yaml` | Medical dialogue classification using phi4-mini model |

## Prerequisites

- Kubernetes cluster (Minikube, Docker Desktop, or cloud)
- `kubectl` configured to access your cluster
- For MEDIQA: CSV input file (see below)

## Quick Start

### 1. Setup (one-time)

```shell
# Create namespace and storage
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-storage.yaml

# Verify
kubectl get namespace ollama
kubectl get pvc -n ollama
```

### 2. Run Demo Job

```shell
kubectl apply -f demo.yaml

# Watch logs
kubectl logs -n ollama -f -l app=ollama-etl-demo -c logstash

# Check output
kubectl exec -n ollama -it $(kubectl get pod -n ollama -l app=ollama-etl-demo -o jsonpath='{.items[0].metadata.name}') -c logstash -- cat /output/classifications.json
```

### 3. Run MEDIQA Job

The job auto-downloads the CSV from GitHub if not present:

```shell
# Just run the job - CSV downloads automatically
kubectl apply -f mediqa.yaml

# Watch logs
kubectl logs -n ollama -f -l app=ollama-etl-mediqa -c logstash
```

**Alternative**: Pre-copy the CSV file to the cluster (useful for air-gapped environments):

```shell
# For Minikube
minikube ssh -- sudo mkdir -p /mnt/ollama-etl-output/input
minikube cp ./MTS-Dialog-TestSet-1-MEDIQA-Chat-2023.csv /mnt/ollama-etl-output/input/

# Then run the job
kubectl apply -f mediqa.yaml
```

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│  Kubernetes Job Pod                                             │
│                                                                 │
│  ┌───────────────┐                                              │
│  │ Init Container│  Downloads LLM model (cached for reuse)      │
│  └───────┬───────┘                                              │
│          ▼                                                      │
│  ┌───────────────┐    ┌───────────────┐                        │
│  │    Ollama     │◄──►│   Logstash    │                        │
│  │   (LLM API)   │    │   (ETL)       │                        │
│  │               │    │               │                        │
│  │  • Serve model│    │  • Read input │                        │
│  │  • Process    │    │  • Call LLM   │                        │
│  │    requests   │    │  • Parse JSON │                        │
│  │               │    │  • Write out  │                        │
│  └───────────────┘    └───────────────┘                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Output

### Demo Job
- **Location**: `/output/classifications.json` (in pod)
- **Format**: JSON lines with original text and classification (positive/negative/neutral/question/statement)

### MEDIQA Job
- **Location**: `/mnt/ollama-etl-output/mediqa-classifications.json` (on node)
- **Format**: JSON lines with row_id, expected vs generated classification, clinical summary

To copy results locally:
```bash
# For Minikube
minikube ssh -- cat /mnt/ollama-etl-output/mediqa-classifications.json > results.json

# Or from pod (while running)
kubectl cp ollama/<pod-name>:/output/mediqa-classifications.json ./results.json -c logstash
```

## Cleanup

```bash
# Delete jobs (keeps model cache)
kubectl delete job -n ollama --all

# Full cleanup (removes everything)
kubectl delete -f mediqa.yaml
kubectl delete -f demo.yaml
kubectl delete -f 01-storage.yaml
kubectl delete -f 00-namespace.yaml
```

## Resource Requirements

| Component | Memory | CPU | Notes |
|-----------|--------|-----|-------|
| Ollama (phi4-mini) | 6-8 GB | 4-6 cores | Runs the LLM |
| Logstash | 1-2 GB | 1-2 cores | ETL processing |
| Model download | ~2.5 GB disk | - | Cached in PV |

## Troubleshooting

### Pod stuck in Pending
```bash
kubectl describe pod -n ollama <pod-name>
```
Usually means insufficient resources. Try reducing limits in the YAML.

### Model download fails
```bash
kubectl logs -n ollama <pod-name> -c model-loader
```
Check network connectivity. The init container downloads from ollama.com.

### Logstash errors
```bash
kubectl logs -n ollama <pod-name> -c logstash
```
Look for CSV parsing errors or Ollama connection issues.

### Check all events
```bash
kubectl get events -n ollama --sort-by='.lastTimestamp'
```

## Customization

### Change the model
Edit the YAML files and replace `phi4-mini` with another model (e.g., `llama3`, `phi3:mini`):
1. Init container: `ollama pull <model>`
2. Model check: `grep -q <model>`
3. ConfigMap: `"model": "<model>"`

### Adjust resources
Modify the `resources` section in each container:
```yaml
resources:
  limits:
    memory: "8Gi"
    cpu: "4"
```

## License

MIT License - feel free to use and modify.
