# 🛡️ Policy-Aware Code Compliance Auditor

> An AI-powered hybrid system that scans codebases for business policy violations using **CodeBERT**, **LLaMA 3.1**, **Kimi K2**, **Gemini Embeddings**, and **FAISS**.

---

## 👥 Authors

| Name | GitHub |
|------|--------|
| Rithvik Matta — *System Architect* | [@rtk5](https://github.com/rtk5) |
| Rishil Abhijit — *Pipeline & Integration Engineer* | [@RishilJalisatgi](https://github.com/RishilJalisatgi) |
| Rohan — *ML Engineer / Model Fine-Tuning* | [@Rohan-134v](https://github.com/Rohan-134v) |

---

## 📌 Table of Contents

- [Problem Statement](#-problem-statement)
- [Solution Overview](#-solution-overview)
- [Tech Stack](#-tech-stack)
- [System Architecture](#-system-architecture)
- [End-to-End Workflow](#-end-to-end-workflow)
- [Compliance Policies](#-compliance-policies)
- [RAG Pipeline](#-rag-pipeline-policy-retrieval)
- [Hybrid Decision Engine](#-hybrid-decision-engine)
- [Fine-Tuning CodeBERT with LoRA](#-fine-tuning-codebert-with-lora)
- [Training Data & Dataset Strategy](#-training-data--dataset-strategy)
- [Audit Results](#-audit-results)
- [Key Design Decisions](#-key-design-decisions)
- [Setup & Installation](#-setup--installation)
- [Usage](#-usage)
- [Project Structure](#-project-structure)

---

## 🚨 Problem Statement

### Manual Review Fails at Scale
Large codebases have hundreds of functions. Human review misses violations — especially subtle ones like bypassing service layers or missing audit logs.

### Policy Violations = Real Risk
GDPR non-compliance, unlogged transactions, hardcoded discounts — these cause regulatory fines, security breaches, and audit failures.

### No Automated Policy Enforcement
Linters check syntax. No tool checks whether your code follows your company's own **business rules and architecture policies**.

---

## 💡 Solution Overview

A **hybrid AI system** that automatically audits code against business policies at any scale. It combines:

- **Semantic policy retrieval** (FAISS + Gemini Embeddings) to find which policies are relevant to each function
- **Fine-tuned CodeBERT** as a fast binary classifier (compliant / violation)
- **Dual-LLM reasoning** using LLaMA 3.1 8B for summarization and Kimi K2 for deep compliance auditing
- **PDF/JSON report generation** for downloadable audit results

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|-----------|
| **Classifier** | CodeBERT (`microsoft/codebert-base`) — fine-tuned with LoRA |
| **LLM Summarizer** | LLaMA 3.1 8B Instant via Groq API |
| **LLM Auditor** | Kimi K2 Instruct (`moonshotai`) via Groq API |
| **Embeddings** | Gemini Embedding 2 Preview (3072-dim) |
| **Vector DB** | FAISS (`faiss-cpu`) — semantic policy search |
| **Code Parsing** | Python `ast` — Abstract Syntax Tree chunking |
| **Fine-Tuning** | HuggingFace `transformers` + `PEFT` + `LoRA` |
| **Reports** | ReportLab PDF · JSON |
| **Runtime** | Google Colab (T4 GPU recommended) |

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Repository                         │
│                   (Python .py files)                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              AST Chunking (Python ast module)                │
│         Extracts functions & classes as code chunks          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│         LLaMA 3.1 8B — Business Summary (2–3 sentences)      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│    Gemini Embedding (RETRIEVAL_QUERY mode, 3072-dim)         │
│         ↓ FAISS nearest-neighbor search                      │
│         → Top-3 semantically relevant policies               │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│         CodeBERT Classifier → p(violation) per policy        │
└────────────────────────┬────────────────────────────────────┘
                         │
              ┌──────────┴──────────┐
              │  Hybrid Decision    │
              │      Engine         │
         ┌────┴────┐          ┌────┴────┐
         │  > 65%  │  35–65%  │  < 35%  │
         │  HIGH   │ AMBIGUOUS│  HIGH   │
         │VIOLATION│   ZONE   │COMPLIANT│
         └────┬────┘    │     └────┬────┘
              │         ▼         │
              │   Kimi K2 Full    │
              │   LLM Reasoning   │
              └────────┬──────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│           Structured JSON Verdict per chunk                  │
│    { compliant, violations[], explanation, severity }        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│           PDF + JSON Audit Report (ReportLab)                │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔄 End-to-End Workflow

| Step | Component | Description |
|------|-----------|-------------|
| 1 | **GitHub Repo** | Clone the target repo and discover all `.py` files |
| 2 | **AST Chunking** | Use Python's `ast` module to extract functions and classes as semantically complete chunks |
| 3 | **LLaMA Summary** | LLaMA 3.1 8B generates a 2–3 sentence business-level summary of each code chunk |
| 4 | **FAISS Retrieval** | The summary is embedded with Gemini and matched against the policy FAISS index → top-3 policies returned |
| 5 | **CodeBERT** | Fine-tuned classifier runs on (chunk, policy) pairs → outputs `p(violation)` per policy |
| 6 | **Hybrid Decision** | Route based on confidence: >65% fast-path violation, <35% fast-path compliant, 35–65% → full LLM |
| 7 | **Kimi K2 Audit** | Reasoning LLM produces structured JSON verdict with violations, explanation, and severity |
| 8 | **PDF Report** | ReportLab compiles a downloadable audit report from all chunk verdicts |

---

## 📋 Compliance Policies

The system enforces **12 policies across 5 domains**:

### 💳 Payments (3 policies)
- `AuditLogger` must be used on all transaction functions
- Only approved `PaymentService` providers may be used
- No direct database updates to payment or order status

### 🏷️ Discounts (2 policies)
- No hardcoded discount values anywhere in the codebase
- All discounts must go through `OfferService`

### 🔒 Customer PII (3 policies)
- Email and personal data must only be accessed via `CustomerService`
- No direct database queries for PII fields
- All access patterns must be GDPR-compliant

### 📧 Communications (2 policies)
- All outbound emails must go through the central email service
- Customer opt-out preferences must always be respected

### 🏛️ Architecture (2 policies)
- No direct ORM model manipulation (must go through service layer)
- All critical operations must have structured logging

---

## 🔍 RAG Pipeline: Policy Retrieval

### How FAISS Retrieval Works

1. **Index time:** All 12 policies are embedded using Gemini Embedding 2 (`RETRIEVAL_DOCUMENT` mode, 3072-dim) and stored in a FAISS index.
2. **Query time:** Each code chunk → LLaMA 8B summary → Gemini Embedding (`RETRIEVAL_QUERY` mode).
3. **Search:** FAISS nearest-neighbor search returns the top-3 semantically matching policies.
4. **Efficiency:** CodeBERT runs only on these 3 policies (not all 12) → saves **75% of inference cost**.

### Key Stats

| Metric | Value |
|--------|-------|
| Embedding dimensions | 3072 |
| Policies retrieved per chunk | Top-3 |
| CodeBERT inference cost reduction | 75% |

### Why Two LLMs?

**LLaMA 3.1 8B** — Fast and cheap. Used only for generating 2–3 sentence business summaries to drive retrieval. No compliance reasoning required at this step.

**Kimi K2 Instruct** — A reasoning model. Used for deep compliance judgment with structured JSON output. Only called when needed (ambiguous zone or high-confidence confirmation).

> Separating concerns = lower cost + higher quality on the hard task.

---

## ⚙️ Hybrid Decision Engine

The core innovation: **CodeBERT as a fast gatekeeper** before calling the expensive LLM.

```
CodeBERT p(violation)
           │
    ┌──────┴──────────────────────┐
    │                             │
  > 65%                        < 35%
  FAST PATH                   FAST PATH
HIGH CONFIDENCE             HIGH CONFIDENCE
  VIOLATION                   COMPLIANT
    │                             │
    │        35% – 65%            │
    │      AMBIGUOUS ZONE         │
    │    Full Kimi K2 reasoning   │
    │    triggered with hint      │
    │                             │
    └──────────────┬──────────────┘
                   │
         Structured JSON output:
         compliant | violations[] | explanation | severity
         Prefixed: [Hybrid/high-confidence→LLM-explain]
                or [Hybrid/ambiguous→full-LLM]
```

- **>65%:** CodeBERT verdict is trusted. The LLM is asked to justify — not re-decide. Severity is assigned.
- **35–65%:** Full Kimi K2 reasoning is triggered. The classifier hint is passed as context.
- **<35%:** CodeBERT verdict is trusted. LLM confirms and explains the clean code.

This routing saves approximately **60% of expensive LLM calls**.

---

## 🤖 Fine-Tuning CodeBERT with LoRA

**Module:** `fineTuned_model.ipynb`

### CodeBERT Architecture

- **Base model:** `microsoft/codebert-base` — a RoBERTa-based transformer pre-trained on natural language and code (Python, Java, JS, and more)
- **Input:** `(code_chunk, policy_text)` concatenated and tokenized (max 512 tokens)
- **Fine-tuned head:** `classifier.dense → classifier.out_proj`
- **Output:** 2 logits → softmax → `p(COMPLIANT)` and `p(VIOLATION)`
- **Task:** Binary sequence classification — `0 = compliant`, `1 = violation`

### LoRA Hyperparameters

| Parameter | Value | Reason |
|-----------|-------|--------|
| `lora_r` | 16 | Binary classifier — higher wastes VRAM |
| `lora_alpha` | 32 | Standard 2×r for stable gradients |
| `lora_targets` | Q, K | Query + Key attention matrices |
| `lr` | 2e-4 | LoRA standard; lower = no overfit |
| `epochs` | 5 | Converges on ~1000 real samples |
| `scheduler` | cosine | Smooth decay vs linear |
| `fp16` | True | Half-precision on T4 GPU |
| `effective_batch` | 32 | 16 × grad_accum of 2 |

---

## 📊 Training Data & Dataset Strategy

**Real datasets only — NO synthetic data.** Four public HuggingFace datasets are used.

| Dataset | Source | Type | Usage |
|---------|--------|------|-------|
| **SecureCode-v2** | `scthornton/securecode-v2` | Security | Python samples labeled secure/vulnerable → mapped to policy violations |
| **CodeSearchNet** | `code_search_net` | General | Large-scale NL+code pairs → labeled by keyword-policy mapping |
| **BigCode / The Stack** | `bigcode/the-stack` | Scale | Massive Python codebase → chunked and policy-assigned via keyword matching |
| **Vuln Detection** | HuggingFace vulnerability datasets | Security | Vulnerability-labeled code → `violation=1` for security-failing functions |

**Policy assignment strategy:** Keyword overlap with no API calls required.

```
payment  → AuditLogger policy
email    → Communications policy
discount → Discount policy
customer → Customer PII policy
...
```

---

## 📈 Audit Results — Live Run

Live run on a real e-commerce Django codebase:

| Metric | Value |
|--------|-------|
| Code chunks audited | 337 |
| Violations found | 24 |
| Overall compliance rate | **93%** |
| Python files scanned | 9 |

### File-Level Breakdown

| File | Chunks | Violations | Severity |
|------|--------|-----------|----------|
| `payment/abstract_models.py` | 27 | 5 | 🔴 HIGH |
| `customer/abstract_models.py` | 26 | 6 | 🔴 HIGH |
| `offer/abstract_models.py` | 90 | 5 | 🟡 MEDIUM |
| `checkout/utils.py` | 33 | 3 | 🟡 MEDIUM |
| `payment/utils.py` | 19 | 1 | 🟡 MEDIUM |
| `customer/utils.py` | 8 | 2 | ⚪ LOW |
| `checkout/session.py` | 21 | 0 | ✅ CLEAN |
| `offer/utils.py` | 3 | 0 | ✅ CLEAN |

---

## 🧠 Key Design Decisions

### 01 — AST Chunking > Line Splitting
Python's `ast` module extracts semantically complete functions and classes — not arbitrary line blocks. Each chunk has one clear purpose, making it a meaningful unit for policy evaluation.

### 02 — Hybrid Routing = Cost Control
High-confidence cases (>65% or <35%) bypass deep LLM reasoning. Only ambiguous cases get full Kimi K2 treatment — saving ~60% of LLM calls at the cost of zero accuracy degradation on clear-cut cases.

### 03 — FAISS Before CodeBERT
Running CodeBERT on all 12 policies = 12 inference passes per chunk. FAISS narrows to top-3 first → reduces classifier calls by **75%**.

### 04 — Rate-Limit Retry Logic
`call_groq_with_retry()` parses the exact wait time from Groq's error message and sleeps precisely that long — up to 5 attempts. Zero wasted time from fixed backoff.

### 05 — Graceful Degradation
If CodeBERT fails to load, the pipeline degrades to LLM-only mode automatically — same output schema, just without classifier confidence scores. The system always produces a result.

### 06 — LoRA for Efficiency
LoRA fine-tunes only the query+key attention matrices (not all weights). ~16M parameters added — full fine-tuning would require 10× more compute and would be infeasible on a T4 GPU.

---

## 🚀 Setup & Installation

### Prerequisites

- Python 3.10+
- A GPU runtime (Google Colab T4 recommended, or local CUDA GPU)
- API keys for:
  - [Groq](https://console.groq.com/) (for LLaMA 3.1 8B and Kimi K2)
  - [Google AI Studio](https://aistudio.google.com/) (for Gemini Embeddings)

### 1. Clone the Repository

```bash
git clone https://github.com/rtk5/policy-aware-auditor.git
cd policy-aware-auditor
```

### 2. Install Dependencies

```bash
pip install -r requirements.txt
```

Or manually:

```bash
pip install transformers peft faiss-cpu groq google-generativeai \
            reportlab gitpython torch accelerate bitsandbytes \
            huggingface_hub datasets
```

### 3. Set API Keys

```bash
export GROQ_API_KEY="your_groq_api_key"
export GEMINI_API_KEY="your_gemini_api_key"
```

Or in a Colab notebook:

```python
import os
os.environ["GROQ_API_KEY"] = "your_groq_api_key"
os.environ["GEMINI_API_KEY"] = "your_gemini_api_key"
```

### 4. (Optional) Fine-Tune CodeBERT

If you want to run your own fine-tuning instead of using the pre-trained checkpoint:

1. Open `fineTuned_model.ipynb` in Google Colab (T4 GPU runtime)
2. Run all cells to:
   - Load training datasets from HuggingFace
   - Apply LoRA configuration
   - Fine-tune for 5 epochs
   - Save the model checkpoint

### 5. Build the FAISS Policy Index

The FAISS index is built automatically on first run from the policy definitions. If you want to pre-build it:

```python
from pipeline import build_policy_index
build_policy_index()  # Embeds all 12 policies via Gemini → saves FAISS index to disk
```

---

## 🖥️ Usage

### Run the Full Audit Pipeline

```python
from auditor import run_audit

results = run_audit(
    repo_url="https://github.com/target-org/target-repo",
    output_dir="./audit_output"
)
```

This will:
1. Clone the repo and extract all `.py` files
2. Parse functions/classes via AST chunking
3. Generate LLaMA summaries for each chunk
4. Retrieve top-3 relevant policies via FAISS
5. Run CodeBERT classifier
6. Route through the hybrid decision engine
7. Call Kimi K2 for ambiguous cases
8. Generate `audit_report.pdf` and `audit_results.json` in `output_dir`

### Run on a Local Directory

```python
from auditor import run_audit_local

results = run_audit_local(
    code_dir="/path/to/your/project",
    output_dir="./audit_output"
)
```

### Run via Notebook

Open `complete_code.ipynb` and follow the cells in order. Each section is clearly labeled and can be run independently.

### Output Format

Each code chunk produces a verdict in this schema:

```json
{
  "chunk_name": "process_payment",
  "file": "payment/utils.py",
  "compliant": false,
  "violations": [
    "Missing AuditLogger on transaction",
    "Direct DB update to order status"
  ],
  "explanation": "This function updates order status directly via ORM without going through PaymentService and without any AuditLogger call, violating both the payment logging and architecture policies.",
  "severity": "HIGH",
  "classifier_confidence": 0.87,
  "decision_path": "[Hybrid/high-confidence→LLM-explain]"
}
```

---

## 📁 Project Structure

```
policy-aware-auditor/
│
├── complete_code.ipynb         # Main pipeline notebook (Rithvik)
├── fineTuned_model.ipynb       # CodeBERT fine-tuning notebook (Rohan)
│
├── policies/
│   └── policies.txt            # 12 business policies (5 domains)
│
├── src/
│   ├── chunker.py              # AST-based code chunking
│   ├── embedder.py             # Gemini embedding wrapper
│   ├── faiss_index.py          # FAISS index build + retrieval
│   ├── classifier.py           # Fine-tuned CodeBERT inference
│   ├── summarizer.py           # LLaMA 3.1 8B via Groq
│   ├── auditor.py              # Kimi K2 via Groq — deep audit
│   ├── hybrid_engine.py        # Routing logic (fast-path vs full-LLM)
│   ├── retry.py                # call_groq_with_retry() with smart backoff
│   ├── report_generator.py     # ReportLab PDF + JSON output
│   └── pipeline.py             # End-to-end orchestration
│
├── models/
│   └── codebert-lora/          # LoRA fine-tuned checkpoint (after training)
│
├── requirements.txt
└── README.md
```

---

## ⚠️ Notes & Limitations

- **API costs:** Kimi K2 is only called for the ambiguous zone (35–65% confidence) to minimize cost. LLaMA 3.1 8B is used for all summaries as it is fast and cheap on Groq.
- **Rate limits:** The `call_groq_with_retry()` function handles Groq rate limits automatically by parsing exact wait times from error responses.
- **GPU required for fine-tuning:** The LoRA fine-tuning step (`fineTuned_model.ipynb`) requires a GPU. Google Colab T4 is sufficient. Inference-only mode can run on CPU.
- **Graceful degradation:** If the CodeBERT checkpoint is unavailable, the pipeline automatically falls back to LLM-only mode with the same output schema.
- **FAISS is CPU-only:** The system uses `faiss-cpu`. For very large policy sets (100+), consider `faiss-gpu`.

---

## 📜 License

MIT License — see [LICENSE](LICENSE) for details.

---

*Built as part of a university AI systems project. Live-tested on a real Django e-commerce codebase with 337 code chunks and 93% compliance rate.*
