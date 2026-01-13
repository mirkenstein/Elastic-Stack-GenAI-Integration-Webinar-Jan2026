# LLM Inference Pipeline Demo - Medical Conversation Classification

## Overview

This demo demonstrates how to use Elasticsearch's inference pipeline to automatically classify and summarize medical conversations using Large Language Models (LLMs) at ingestion time.

> **⚠️ Demo Purpose**: This demonstration uses the **publicly available MTS-Dialog research dataset** for educational purposes. No actual Protected Health Information (PHI) is used.

### Dataset: MTS-Dialog

The **MTS-Dialog (Medical Task-Oriented Dialog) dataset** is a benchmark dataset containing annotated doctor-patient conversations with 20 standardized clinical section categories used in medical documentation.

**Dataset Source**: [MTS-Dialog GitHub Repository](https://github.com/abachaa/MTS-Dialog/blob/main/Main-Dataset/MTS-Dialog-TestSet-1-MEDIQA-Chat-2023.csv)

### Use Case

This pipeline:
- **Categorizes** transcribed doctor-patient conversations into one of 20 predefined clinical section categories
- **Generates** a short summary with possible diagnosis
- **Outputs** structured JSON with section header and section text

---

### Advantages of Inference at Ingestion Time

Running LLM inference as part of the Elasticsearch ingest pipeline provides several key benefits:

1. **Centralized Processing**: Classification and enrichment happens automatically during indexing, eliminating the need for separate preprocessing steps
2. **Consistent Enrichment**: Every document is processed uniformly with the same model and prompt, ensuring data consistency
3. **Real-time Analysis**: Documents are classified and summarized immediately upon ingestion, making insights available instantly
4. **Reduced Application Complexity**: Applications can index raw transcripts without needing to implement their own LLM integration
5. **Scalability**: Elasticsearch handles the orchestration, batching, and error handling of LLM calls
6. **Version Control**: Pipeline definitions are stored and versioned in Elasticsearch, making them auditable and reproducible
7. **Cost Efficiency**: Failed documents can be routed to separate indices for retry, preventing unnecessary duplicate API calls

---

## Demo Steps for Kibana Dev Console

### Step 1: Create the Inference Endpoint

First, configure the Anthropic Claude model as an inference endpoint. This establishes the connection to the LLM service.

```json
PUT _inference/completion/claude-opus
{
  "service": "anthropic",
  "service_settings": {
    "api_key": "${antropic_api_key}",
    "model_id": "claude-opus-4-5"
  },
  "task_settings": {
    "max_tokens": 2048
  }
}
```

**Expected Result**: `{"inference_id": "claude-opus", "service": "anthropic", ...}`

---

### Step 2: Test the Inference Endpoint

Verify the endpoint is working correctly with a simple test query.

```json
POST _inference/completion/claude-opus
{
  "input": "Tell me a joke about lipids?"
}
```

**Expected Result**: A JSON response containing the generated text from Claude.

---

### Step 3: Create the LLM Classification Ingest Pipeline

Create the ingest pipeline that will process medical conversations. This pipeline:
1. Builds a prompt with the classification instructions and input text
2. Calls the LLM inference endpoint
3. Removes the temporary prompt field
4. Parses the LLM's JSON response into structured fields

```json
PUT _ingest/pipeline/llm-classification
{
  "description": "LLM-based medical conversation classification and summarization pipeline",
  "processors": [
    {
      "script": {
        "description": "Build prompt for LLM with classification instructions",
        "source": "ctx.prompt = 'Please provide short summary and diagnose. Put the diagnose at the beginning. Also categorize the transcribed doctor patient conversation into just one of these 20 categories: (1. fam_sochx [FAMILY HISTORY/SOCIAL HISTORY], 2. genhx [HISTORY of PRESENT ILLNESS], 3. pastmedicalhx [PAST MEDICAL HISTORY], 4. cc [CHIEF COMPLAINT], 5. pastsurgical [PAST SURGICAL HISTORY], 6. allergy, 7. ros [REVIEW OF SYSTEMS], 8. medications, 9. assessment, 10. exam, 11. diagnosis, 12. disposition, 13. plan, 14. edcourse [EMERGENCY DEPARTMENT COURSE], 15. immunizations, 16. imaging, 17. gynhx [GYNECOLOGIC HISTORY], 18. procedures, 19. other_history, 20. labs). Output should be a pure JSON string without markdown format: {\\\"section_header\\\": \\\"one of the 20 categories\\\", \\\"section_text\\\": \\\"short summary\\\"} : ' + ctx.text"
      }
    },
    {
      "inference": {
        "description": "Call Claude Opus for classification and summarization",
        "model_id": "claude-opus",
        "input_output": {
          "input_field": "prompt",
          "output_field": "generated"
        }
      }
    },
    {
      "remove": {
        "description": "Remove temporary prompt field",
        "field": "prompt"
      }
    },
    {
      "json": {
        "description": "Parse LLM JSON response into structured fields",
        "field": "generated",
        "target_field": "json_target"
      }
    }
  ],
  "on_failure": [
    {
      "set": {
        "description": "Route failed documents to separate index",
        "field": "_index",
        "value": "failed-{{{_index}}}"
      }
    },
    {
      "set": {
        "description": "Capture error details for troubleshooting",
        "field": "ingest.failure",
        "value": "{{_ingest.on_failure_message}}"
      }
    }
  ]
}
```

**Expected Result**: `{"acknowledged": true}`

---

### Step 4: Test the Pipeline with Sample Conversations

Test the pipeline using the first 3 sample conversations from the MTS-Dialog test dataset.

**Dataset Reference**: The examples below are taken from the first 3 rows of the [MTS-Dialog TestSet-1 MEDIQA-Chat-2023 CSV](https://github.com/abachaa/MTS-Dialog/blob/main/Main-Dataset/MTS-Dialog-TestSet-1-MEDIQA-Chat-2023.csv).

#### Test Case 1: New Diagnosis Discussion (MRI Results)

```json
POST _ingest/pipeline/llm-classification/_simulate
{
  "docs": [
    {
      "_source": {
        "text": "Doctor: Good afternoon, sir. Did you just have a birthday? I don't have my chart with me right now, the nurse is bringing it. Patient: Good afternoon, sir. Yes, I just turned fifty five. Doctor: You identify as African American, correct? Patient: Yes, that's right. Doctor: When was your last visit, sir? Patient: Um, it was on July twenty ninth two thousand eight. Doctor: Yes, I see. Did we go over your M R I results? Patient: No, I was having those new seizures, remember? Doctor: Yes, I do. Well, the M R I demonstrated right contrast temporal mass. Patient: What exactly does that mean, doctor? Doctor: Well, given this mass, and your new seizures, I am concerned that this could be a high grade glioma, we'll need to do more tests."
      }
    }
  ]
}
```

**Expected Result**: The document enriched with `generated` and `json_target` fields containing the section category (likely "diagnosis" or "assessment") and a summary of the conversation.

---

#### Test Case 2: Family Medical History

```json
POST _ingest/pipeline/llm-classification/_simulate
{
  "docs": [
    {
      "_source": {
        "text": "Doctor: Any medical issues running in your families? Patient: Oh yes, stroke. Doctor: Anything else? Patient: Sleep apnea."
      }
    }
  ]
}
```

**Expected Result**: Section category "fam_sochx" (Family History/Social History) with summary of family stroke and sleep apnea history.

---

#### Test Case 3: Review of Systems (Musculoskeletal)

```json
POST _ingest/pipeline/llm-classification/_simulate
{
  "docs": [
    {
      "_source": {
        "text": "Doctor: Any pain in your muscles? Patient: No, no pain. Doctor: How about joint pain? Patient: Um no, I don't feel any joint pain. Doctor: Okay, good. Doctor: Do you feel any stiffness or weakness in your muscle? Patient: Um, nothing like that. Doctor: Do you have any back pain? Patient: No. Doctor: Okay."
      }
    }
  ]
}
```

**Expected Result**: Section category "ros" (Review of Systems) with summary indicating negative musculoskeletal symptoms.

---

## Alternative: Using Locally Hosted Models

For organizations that prefer to use self-hosted models (e.g., for data privacy or cost reasons), you can configure Ollama or other OpenAI-compatible endpoints:

```json
PUT _inference/completion/ollama-completion-gemma3
{
  "service": "openai",
  "service_settings": {
    "api_key": "${LOCAL_LLM_API_KEY}",
    "model_id": "gemma3",
    "url": "${LOCAL_SERVICE_URL}/v1/chat/completions"
  }
}
```

Then update the pipeline's `model_id` to use `ollama-completion-gemma3` instead of `claude-opus`.

---

## Production Usage

Once validated, documents can be indexed with the pipeline applied automatically:

```json
POST medical-conversations/_doc?pipeline=llm-classification
{
  "text": "Doctor: What brings you in today? Patient: I've been having chest pain...",
  "patient_id": "12345",
  "visit_date": "2024-01-15"
}
```

Or set it as the default pipeline for an index:

```json
PUT medical-conversations/_settings
{
  "index.default_pipeline": "llm-classification"
}
```

---

## Output Structure

After processing, documents will have this structure:

```json
{
  "text": "Doctor: Any medical issues running in your families?...",
  "generated": "{\"section_header\": \"fam_sochx\", \"section_text\": \"Patient reports family history of stroke and sleep apnea\"}",
  "json_target": {
    "section_header": "fam_sochx",
    "section_text": "Patient reports family history of stroke and sleep apnea"
  },
  "patient_id": "12345",
  "visit_date": "2024-01-15"
}
```

---

## 20 Clinical Section Categories

The following categories are used for classification:

1. **fam_sochx** - Family History / Social History
2. **genhx** - History of Present Illness
3. **pastmedicalhx** - Past Medical History
4. **cc** - Chief Complaint
5. **pastsurgical** - Past Surgical History
6. **allergy** - Allergies
7. **ros** - Review of Systems
8. **medications** - Current Medications
9. **assessment** - Clinical Assessment
10. **exam** - Physical Examination
11. **diagnosis** - Diagnosis
12. **disposition** - Patient Disposition
13. **plan** - Treatment Plan
14. **edcourse** - Emergency Department Course
15. **immunizations** - Immunization History
16. **imaging** - Imaging Studies
17. **gynhx** - Gynecologic History
18. **procedures** - Procedures Performed
19. **other_history** - Other Historical Information
20. **labs** - Laboratory Results


---

## What LLMs Can Do Beyond Traditional BERT Models

### 1. All Traditional NLP Tasks (Without Fine-tuning)
- **Named Entity Recognition (NER)**
- **Text Classification**
- **Sentiment Analysis**
- **Question Answering**
- **Token Classification**
- **Similarity/Semantic Search (via embeddings)**

**Key Advantage**: No fine-tuning required—use natural language instructions instead of labeled training datasets.

### 2. Advanced Categorization & Analysis
- **Aspect-Based Sentiment Analysis (ABSA)** with reasoning
- **Multi-label classification** with confidence scores
- **Hierarchical categorization** into nested taxonomies
- **Dynamic category creation** based on data patterns

### 3. Summarization & Generation
- **Abstractive summarization** (not just extractive)
- **Customizable summary length and style** (executive summary, bullet points, medical notes)
- **Multi-document summarization**
- **Conditional generation** (generate text matching specific criteria)

### 4. Zero-Shot & Few-Shot Learning
- **Immediate deployment** without training data collection
- **Rapid prototyping** of new classification tasks
- **Adaptation to new domains** via prompt engineering
- **Task switching** with a single model

### 5. Complex Reasoning & Chain-of-Thought
- **Multi-step logical reasoning**
- **Explanation generation** (why a classification was made)
- **Cause-and-effect analysis**
- **Hypothesis generation** from medical symptoms

### 6. Structured Output Generation
- **JSON/XML/YAML generation** from unstructured text
- **Schema validation** and error correction
- **Database record creation** from narratives
- **API payload generation**

### 7. Long Context Understanding
- **Process entire documents** (100K+ tokens vs BERT's 512 tokens)
- **Cross-document analysis** and comparison
- **Conversation history tracking** across multiple turns
- **Book/report-level comprehension**

### 8. Advanced Entity & Relationship Extraction
- **Contextual entity disambiguation** (Dr. Smith the surgeon vs Dr. Smith the patient's relative)
- **Relationship extraction** (who prescribed what to whom)
- **Temporal reasoning** (sequence of events)
- **Co-reference resolution** with explanations

### 9. Data Quality & Augmentation
- **Synthetic data generation** for training other models
- **Data validation and error detection**
- **Automated data normalization** (addresses, dates, medical codes)
- **Missing data imputation** with reasoning

### 10. Multi-Task Single Model
- **One endpoint for all tasks** (vs. multiple fine-tuned BERT models)
- **Cost efficiency** (no training infrastructure needed)
- **Simplified MLOps** (no model versioning per task)
- **Dynamic task routing** based on document type

### 11. Conversational & Interactive Capabilities
- **Clarification questions** during processing
- **Interactive refinement** of classifications
- **Multi-turn dialogue understanding**
- **Context-aware responses**

### 12. Domain Adaptation Without Retraining
- **Prompt-based specialization** (add medical knowledge via instructions)
- **In-context learning** from examples in the prompt
- **Cross-domain transfer** (apply learnings from one domain to another)
- **Terminology adaptation** via custom glossaries in prompts

### 13. Enhanced Multi-Lingual Support
- **100+ languages** without language-specific models
- **Cross-lingual transfer** and translation
- **Code-switching handling** (documents mixing languages)
- **Cultural context awareness**

### 14. Creative & Transformation Tasks
- **Content rewriting** (formal ↔ casual tone)
- **Format conversion** (narrative → structured clinical note)
- **De-identification with context preservation**
- **Paraphrasing while maintaining medical accuracy**

### 15. Compliance & Governance
- **Audit trail generation** (explain every decision)
- **Bias detection and reporting**
- **Sensitive content flagging**
- **Regulatory compliance checking** (HIPAA, GDPR)

---

## Practical Example: Medical Record Processing

| **Task** | **BERT Approach** | **LLM Approach** |
|----------|-------------------|------------------|
| **Classify section** | Fine-tune on 1000+ labeled examples | Provide 20 categories in prompt, zero-shot |
| **Extract medications** | Train NER model on medical entities | Prompt: "Extract all medications with dosages as JSON" |
| **Generate summary** | Not possible (BERT is encoder-only) | Natural summarization with diagnosis highlighting |
| **Detect errors** | Requires separate error detection model | Ask: "Are there any contradictions or missing information?" |
| **Update for new category** | Retrain model with new labeled data | Add category to prompt instruction |
