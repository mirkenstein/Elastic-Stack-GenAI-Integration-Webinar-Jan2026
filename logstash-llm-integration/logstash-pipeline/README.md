# Modular Logstash Pipeline for MEDIQA Classification

This directory contains a modular Logstash pipeline split into separate config files.

## Files

| File | Description |
|------|-------------|
| `01-input.conf` | File input with multiline CSV codec |
| `02-filter-csv.conf` | CSV parsing (handles quoted/unquoted fields) |
| `03-filter-llm.conf` | Ollama LLM call and response parsing |
| `04-output.conf` | Console and file output |

## How It Works

Logstash loads all `.conf` files from a directory in **alphabetical order** and merges them into a single pipeline:

```
01-input.conf    →  input { ... }
02-filter-csv.conf  →  filter { ... }  ←─┐
03-filter-llm.conf  →  filter { ... }  ←─┤ Filters execute in order
04-output.conf   →  output { ... }
```

## Usage

### Run with directory
```bash
logstash -f /path/to/logstash-pipeline/
```

### Run with Docker
```bash
docker run --rm -it \
  -v $(pwd)/logstash-pipeline:/usr/share/logstash/pipeline:ro \
  -v /tmp:/tmp \
  docker.elastic.co/logstash/logstash:8.11.0
```

### Run specific files only (skip LLM)
```bash
# Parse CSV only, no LLM calls
logstash -f 01-input.conf -f 02-filter-csv.conf -f 04-output.conf
```

## Customization

### Change the model
Edit `03-filter-llm.conf` line 18:
```
"model": "llama3",
```

### Change input file
Edit `01-input.conf` line 3:
```
path => "/your/path/to/file.csv"
```

### Disable LLM (CSV parsing only)
Rename or remove `03-filter-llm.conf`:
```bash
mv 03-filter-llm.conf 03-filter-llm.conf.disabled
```

## File Naming Convention

Files are processed alphabetically:
- `01-*` to `09-*` - Input configs
- `10-*` to `89-*` - Filter configs
- `90-*` to `99-*` - Output configs

This allows inserting new filters in between:
```
02-filter-csv.conf
025-filter-validation.conf   # New filter
03-filter-llm.conf
```
