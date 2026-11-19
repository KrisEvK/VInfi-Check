# VInFi-Check

**Interpretable and Fine-Grained Fact-Checking for Vietnamese LLM Summaries**

[![Paper](https://img.shields.io/badge/Based%20on-InFi--Check%20arXiv%3A2601.06666-b31b1b?style=flat-square&logo=arxiv)](https://arxiv.org/abs/2601.06666)
[![GitHub](https://img.shields.io/badge/GitHub-KrisEvK%2FVInfi--Check-181717?style=flat-square&logo=github)](https://github.com/KrisEvK/VInfi-Check)
[![Python](https://img.shields.io/badge/Python-3.10+-blue?style=flat-square&logo=python)](https://www.python.org/)
[![Model](https://img.shields.io/badge/Model-Qwen2.5--7B%20%2B%20QLoRA-orange?style=flat-square)](https://huggingface.co/Qwen/Qwen2.5-7B-Instruct)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](https://opensource.org/licenses/MIT)

VInFi-Check is an end-to-end framework for detecting and classifying factual hallucinations in Vietnamese text summaries. Adapted from [InFi-Check](https://arxiv.org/abs/2601.06666), the system introduces a 6-class fine-grained error taxonomy, an automated data synthesis engine utilizing multi-LLM consensus verification, hybrid document retrieval, and a QLoRA-tuned Qwen2.5-7B fact-checking model.

---

## Overview

Large language models frequently hallucinate unsupported claims when generating Vietnamese summaries. VInFi-Check addresses this through:

- **Fine-grained error taxonomy**: Classifies factual discrepancies into 6 distinct categories (Predicate, Entity, Circumstance, Co-reference, Discourse Link, Extrinsic).
- **Automated SFT data engine**: Synthesizes 4,800 verification pairs from Vietnamese news sources using multi-LLM consensus (GPT-4o-mini, Qwen2.5-72B, Gemini Flash) as automated judges.
- **Hybrid retrieval**: Combines BM25 lexical search with BGE-M3 dense embeddings to retrieve supporting evidence from source documents.
- **Fine-tuned fact-checker**: Qwen2.5-7B-Instruct trained via QLoRA on a single GPU to detect factual errors and identify supporting sentences.
- **Interactive UI**: A Gradio web application for document summarization, sentence-level fact verification, and model comparison.

---

## Pipeline

```
Source Documents (300-1000 words)
[VnExpress · BBC News · DetNet Wikipedia]
                    |
Summarization (100-200 words)
[deepseek-chat]
                    |
Consensus Evaluation & Reference Extraction
[Majority voting: GPT-4o-mini · Qwen2.5-72B · Gemini Flash]
                    |
Structured Error Injection (6 error types)
[deepseek-chat with prompt perturbation rules]
                    |
SFT Dataset Assembly (4,800 pairs, 1:5 pos-to-neg ratio)
[train / valid / test JSONL]
                    |
QLoRA Fine-Tuning (rank=16, alpha=32)
[Qwen2.5-7B-Instruct on single T4 GPU, 3 epochs]
                    |
Hybrid RAG & Inference Verification
[BM25 + BGE-M3 retrieval + fine-tuned checker]
                    |
Interactive Gradio Interface
```

---

## Error Taxonomy

Factual errors in generated summaries are mapped into 6 categories:

| # | Error Type | Vietnamese Name | Description | Concrete Example |
| :---: | :--- | :--- | :--- | :--- |
| 1 | **Predicate Error** | Lỗi Vị Ngữ | Altered or contradictory verb/action | *"tăng"* altered to *"giảm"* |
| 2 | **Entity Error** | Lỗi Thực Thể | Missing, swapped, or compressed entities | Dropping *"Xiaomi"* from vendor list |
| 3 | **Circumstance Error** | Lỗi Hoàn Cảnh | Inaccurate numbers, dates, or locations | Year *"2022"* altered to *"2020"* |
| 4 | **Co-reference Error** | Lỗi Đồng Tham Chiếu | Ambiguous or misassigned pronouns | Pronoun *"ông ấy"* assigned to wrong entity |
| 5 | **Discourse Link Error** | Lỗi Liên Kết Diễn Ngôn | Inverted causality or temporal sequencing | Reversing cause and effect relationships |
| 6 | **Extrinsic Error** | Lỗi Ngoại Lai | Fabricated facts absent from source text | Inserting synthetic statistics |

---

## Dataset Construction

### Splits and Statistics

| Split | Source Documents | Positive Pairs | Negative Pairs | Total |
| :--- | :---: | :---: | :---: | :---: |
| Train | 600 | 600 | 3,000 | 3,600 |
| Validation | 100 | 100 | 500 | 600 |
| Test | 100 | 100 | 500 | 600 |
| **Total** | **800** | **800** | **4,000** | **4,800** |

Data sources:
- [BBC News Summary](https://huggingface.co/datasets/gopalkalpande/bbc-news-summary): Business, technology, politics, entertainment.
- [DetNet Wikipedia (vi)](https://github.com/yumoxu/detnet): General, governance, health, law.
- Curated VnExpress articles.

---

## Model and Training Details

### Fact-Checking Backbone

- **Base Model**: `Qwen/Qwen2.5-7B-Instruct`
- **Tuning Method**: QLoRA (4-bit NormalFloat quantization)
- **LoRA Hyperparameters**: rank = 16, alpha = 32, dropout = 0.05
- **Target Modules**: `q_proj`, `k_proj`, `v_proj`, `o_proj`, `gate_proj`, `up_proj`, `down_proj`
- **Training Setup**: 3 epochs, effective batch size = 4, cosine learning rate scheduler with warmup
- **Hardware**: Single NVIDIA T4 GPU (Google Colab)

---

## Repository Structure

```
VInfi-Check/
├── Phosphor-Bai-InFi-Check/
│   ├── InFi-Check construct/
│   │   ├── summary_gen_prompt/          # Perturbation prompts (6 error types)
│   │   ├── summary_eval_prompt/         # Consensus evaluation prompts
│   │   ├── summary_gen.py               # Document summarization
│   │   ├── eval_and_reference_gen.py    # Reference sentence extraction
│   │   ├── structured_dataset_gen.py    # Error injection engine
│   │   └── finetune.py                  # QLoRA fine-tuning script
│   │
│   ├── training_dataset_construct/
│   │   ├── prompt/                      # Few-shot templates and SFT prompts
│   │   └── prepare_dataset_pipeline.ipynb # Dataset formatting pipeline
│   │
│   └── VInFiCheck_Gradio.ipynb          # Interactive Gradio demo notebook
│
└── README.md
```

---

## Quick Start

### 1. Installation

```bash
git clone https://github.com/KrisEvK/VInfi-Check.git
cd VInfi-Check

pip install -r requirements.txt
```

Core dependencies:

```bash
pip install torch transformers peft bitsandbytes accelerate \
            underthesea gradio datasets jsonlines rank-bm25
```

### 2. Environment Configuration

Export the required API keys:

```bash
export GROQ_API_KEY="your-groq-api-key"
export OPENAI_API_KEY="your-openai-api-key"
export DEEPSEEK_API_KEY="your-deepseek-api-key"
```

### 3. Execution Pipeline

```bash
# Step 1: Generate initial summaries
python "Phosphor-Bai-InFi-Check/InFi-Check construct/summary_gen.py"

# Step 2: Multi-LLM consensus evaluation & reference extraction
python "Phosphor-Bai-InFi-Check/InFi-Check construct/eval_and_reference_gen.py"

# Step 3: Inject structured factual errors
python "Phosphor-Bai-InFi-Check/InFi-Check construct/structured_dataset_gen.py"

# Step 4: Fine-tune fact-checker with QLoRA
python "Phosphor-Bai-InFi-Check/InFi-Check construct/finetune.py"
```

---

## Interactive Demo

To launch the web interface, open and run `Phosphor-Bai-InFi-Check/VInFiCheck_Gradio.ipynb` in Google Colab or locally with GPU support:

- **Summarization**: Generates an abstractive Vietnamese summary from an input document.
- **Sentence-Level Fact Verification**: Highlights unsupported statements and maps them to the 6 error categories.
- **Model Comparison**: Side-by-side evaluation between VInFi-Checker, GPT-4o-mini, and Llama-3.3-70B.

---

## References

```bibtex
@article{bai2025inficheck,
  title   = {InFi-Check: Interpretable and Fine-Grained Fact-Checking of LLMs},
  author  = {Bai, et al.},
  journal = {arXiv preprint arXiv:2601.06666},
  year    = {2025},
  url     = {https://arxiv.org/abs/2601.06666}
}

@misc{vinficheck2025,
  title  = {VInFi-Check: Interpretable and Fine-Grained Fact-Checking for Vietnamese LLM Summaries},
  author = {Nguyen, Huu Tri},
  year   = {2025},
  url    = {https://github.com/KrisEvK/VInfi-Check}
}
```

---

## Acknowledgements

- [InFi-Check](https://github.com/Phosphor-Bai/InFi-Check) (Bai et al.) for the underlying fine-grained verification methodology.
- [Qwen Team](https://huggingface.co/Qwen/Qwen2.5-7B-Instruct) for the open-weights foundation model.
- [underthesea](https://github.com/undertheseanlp/underthesea) for Vietnamese NLP text processing utilities.
