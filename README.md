<div align="center">

<h1>🔍 VInFi-Check</h1>

<p><strong>Interpretable and Fine-Grained Fact-Checking for Vietnamese LLM Summaries</strong></p>

<p>
  <a href="https://arxiv.org/abs/2601.06666"><img src="https://img.shields.io/badge/Based%20on-InFi--Check%20arXiv%3A2601.06666-b31b1b?style=flat-square&logo=arxiv" alt="Paper"/></a>
  <a href="https://github.com/KrisEvK/VInfi-Check"><img src="https://img.shields.io/badge/GitHub-VInFi--Check-181717?style=flat-square&logo=github" alt="GitHub"/></a>
  <img src="https://img.shields.io/badge/Language-Vietnamese-blue?style=flat-square" alt="Language"/>
  <img src="https://img.shields.io/badge/Model-Qwen2.5--7B%20%2B%20QLoRA-orange?style=flat-square" alt="Model"/>
  <img src="https://img.shields.io/badge/Status-In%20Progress-yellow?style=flat-square" alt="Status"/>
  <img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" alt="License"/>
</p>

<p>
  Vietnamese adaptation of <a href="https://github.com/Phosphor-Bai/InFi-Check">InFi-Check</a> —
  fine-grained hallucination detection in LLM-generated summaries over Vietnamese news text,
  with a 6-class error taxonomy, majority-vote evaluation, and a QLoRA-tuned Qwen2.5-7B fact-checker.
</p>

</div>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Pipeline](#-pipeline)
- [Error Taxonomy](#-error-taxonomy)
- [Dataset](#-dataset)
- [Models](#-models)
- [Repository Structure](#-repository-structure)
- [Installation](#-installation)
- [Quick Start](#-quick-start)
- [Gradio Demo](#-gradio-demo)
- [Citation](#-citation)

---

## 🧠 Overview

**VInFi-Check** adapts the [InFi-Check](https://arxiv.org/abs/2601.06666) framework to Vietnamese, introducing:

1. **A Vietnamese hallucination benchmark** — the first systematic evaluation of hallucination patterns across multiple LLMs (LLaMA-3.3-70B, Qwen2.5, VinaLlama, Gemini Flash) on Vietnamese news text.
2. **A fine-grained Vietnamese fact-checking model** — Qwen2.5-7B-Instruct fine-tuned with QLoRA, trained on a synthetically constructed dataset with 6 error types.
3. **A full data pipeline** — from Vietnamese document collection → summary generation → multi-model majority-vote verification → structured error injection → SFT dataset preparation.

> **Research context:** University Scientific Research Competition (NCKH), 2024–2025.

---

## ⚙️ Pipeline

The pipeline consists of **7 stages**, each implemented as a resumable, batch-safe script:

```
Stage 1 — Document Filtering
         VnExpress · BBC News · DetNet Wikipedia (vi)
         Filter: 300–1000 words
                    ↓
Stage 2 — Summary Generation
         Llama-3.3-70B via Groq API
         Constraint: 100–200 words, all entities preserved
                    ↓
Stage 3 — Multi-Model Voting Evaluation
         GPT-4o-mini (w/ revision) · Qwen2.5-72B · Gemini Flash
         Majority vote → supported summary
                    ↓
Stage 4 — Reference Sentence Extraction
         GPT-4o → maps each summary sentence → supporting doc sentence(s)
                    ↓
Stage 5 — Structured Error Generation
         GPT-4o → 6 error types × multiple methods
         (Intrinsic: Predicate, Entity, Circumstance, Co-reference, Discourse Link
          Extrinsic: Extrinsic Information)
                    ↓
Stage 6 — SFT Dataset Construction
         prepare_dataset_base.py
         Output: train / valid / test JSONL
                    ↓
Stage 7 — QLoRA Fine-tuning
         Qwen2.5-7B-Instruct · rank=16 · alpha=32
         Google Colab T4 · 3 epochs
```

---

## 🗂 Error Taxonomy

VInFi-Check detects **6 categories** of factual errors in summaries:

| # | Error Type | Vietnamese | Description | Example |
|---|-----------|------------|-------------|---------|
| 1 | **Predicate Error** | Lỗi Vị Ngữ | Wrong verb or predicate changes the action/state | *"tăng"* → *"giảm"* |
| 2 | **Entity Error** | Lỗi Thực Thể | Missing, swapped, or compressed named entities | Omitting *"Xiaomi"* from a list of brands |
| 3 | **Circumstance Error** | Lỗi Hoàn Cảnh | Incorrect time, location, or number | *"2022"* → *"2020"*; *"Hà Nội"* → *"TP.HCM"* |
| 4 | **Co-reference Error** | Lỗi Đồng Tham Chiếu | Wrong pronoun or merged subjects | *"ông ấy"* used for the wrong person |
| 5 | **Discourse Link Error** | Lỗi Liên Kết Diễn Ngôn | Reversed causal / temporal relationship | Cause and effect swapped |
| 6 | **Extrinsic Error** | Lỗi Ngoại Lai | Hallucinated information not in the source | Adding unreferenced statistics |

---

## 📦 Dataset

| Split | Documents | Positive samples | Negative samples |
|-------|-----------|-----------------|-----------------|
| Train | ~600 | ~600 | ~3 000 |
| Valid | 100 | ~100 | ~500 |
| Test  | 100 | ~100 | ~500 |

**Sources:**
- [BBC News Articles](https://huggingface.co/datasets/gopalkalpande/bbc-news-summary) (business, entertainment, politics, sport, tech)
- [DetNet Wikipedia (vi)](https://github.com/yumoxu/detnet) — BUS, GOV, HEA, LAW, LIF, MIL, GEN categories
- VnExpress articles (collected via scraping)

**Error generation methods per type:**

| Error Type | Methods |
|-----------|---------|
| Predicate | Modifying Predictions, Swapping Relation Maskings |
| Entity | Swapping Entities, Compressing Words |
| Circumstance | Swapping Numbers, Swapping Locations/Times |
| Co-reference | Swapping Pronouns, Merging Sentences |
| Discourse Link | Reverse Logical Relationship |
| Extrinsic | Introducing Extrinsic Information |

---

## 🤖 Models

### Summary Generation
| Model | Role | API |
|-------|------|-----|
| `llama-3.3-70b-versatile` | Primary summary generator | Groq |
| `llama-3.1-8b-instant` | Fallback (token limit) | Groq |

### Evaluation (Majority Vote)
| Model | Role |
|-------|------|
| `gpt-4o-mini` | Evaluator + revision |
| `Qwen/Qwen2.5-72B-Instruct` | Evaluator |
| `gemini-flash` | Evaluator |

### Error Generation
| Model | Role |
|-------|------|
| `gpt-4o-2024-11-20` | Structured error injection |

### Fact-Checking (Fine-tuned)
| Model | Config |
|-------|--------|
| `Qwen2.5-7B-Instruct` | Base model |
| QLoRA | rank=16, alpha=32, dropout=0.05 |
| Training | Google Colab T4, 3 epochs, batch=4 |

---

## 📁 Repository Structure

```
VInFi-Check/
│
├── InFi-Check construct/              # Data pipeline scripts
│   ├── dataset_bbc.py                 # BBC News data loader & filter
│   ├── dataset_detnet_wiki.py         # DetNet Wikipedia data loader
│   ├── summary_gen.py                 # Stage 2: Summary generation (Groq/GPT)
│   ├── eval_and_reference_gen.py      # Stage 3–4: Evaluation + reference extraction
│   ├── structured_dataset_gen.py      # Stage 5: Error injection (6 types)
│   │
│   ├── summary_gen_prompt/            # Error injection prompts
│   │   ├── structured_intrinsic/
│   │   │   ├── semantic frame/        # Predicate, Entity, Circumstance
│   │   │   └── discourse/             # Co-reference, Discourse Link
│   │   └── structured_extrinsic/      # Extrinsic error prompts
│   │
│   └── summary_eval_prompt/           # Evaluation prompts (find_support, critics)
│
├── training_dataset_construct/        # SFT dataset construction
│   ├── prepare_dataset_base.py        # Stage 6: Build train/valid/test JSONL
│   └── prompt/                        # SFT prompt templates & few-shot examples
│       ├── sft_prompt.txt
│       ├── claude_prompt_unified.txt
│       └── few_shot_unified.jsonl
│
├── notebooks/                         # Google Colab notebooks
│   ├── 01_summary_gen.ipynb
│   ├── 02_eval_and_reference.ipynb
│   ├── 03_structured_dataset.ipynb
│   ├── 04_prepare_dataset.ipynb
│   ├── 05_finetune_qwen.ipynb
│   └── VInFiCheck_Gradio.ipynb        # Interactive demo
│
└── README.md
```

---

## 🛠 Installation

```bash
git clone https://github.com/KrisEvK/VInfi-Check.git
cd VInfi-Check

pip install -r requirements.txt
```

**Core dependencies:**

```bash
pip install openai groq underthesea transformers peft \
            datasets jsonlines gradio faiss-cpu chromadb \
            bitsandbytes accelerate tqdm
```

**API keys** (add to Colab Secrets or `.env`):

```
GROQ_API_KEY=...
OPENAI_API_KEY=...
```

---

## 🚀 Quick Start

### 1. Prepare documents

```bash
# BBC News
python "InFi-Check construct/dataset_bbc.py"

# Wikipedia (DetNet)
python "InFi-Check construct/dataset_detnet_wiki.py"
```

### 2. Generate summaries

```bash
python "InFi-Check construct/summary_gen.py"
```

### 3. Evaluate & extract references

```bash
python "InFi-Check construct/eval_and_reference_gen.py"
```

### 4. Generate error dataset

```bash
python "InFi-Check construct/structured_dataset_gen.py"
```

### 5. Build SFT dataset

```bash
python training_dataset_construct/prepare_dataset_base.py
# Output: sft_dataset/jsonl/summary_sft_train_pos1neg5_with_ref.jsonl
#                           summary_sft_valid_with_ref.jsonl
#                           summary_sft_test_with_ref.jsonl
```

### 6. Fine-tune

Run `notebooks/05_finetune_qwen.ipynb` on Google Colab (T4 GPU).

---

## 🎮 Gradio Demo

Launch the interactive fact-checking demo:

```bash
# In Google Colab — run VInFiCheck_Gradio.ipynb
# Or locally:
python app.py
```

The demo supports:
- **✦ Sinh tóm tắt** — auto-generate a summary from source text (Llama-3.3-70B via Groq)
- **⬡ Kiểm chứng** — sentence-level fact-checking with error type classification
- **◈ So sánh mô hình** — side-by-side comparison: VInFi-Checker vs GPT-4o-mini vs Llama-3.3-70B

---

## 📄 Citation

If you use VInFi-Check in your research, please cite the original InFi-Check paper:

```bibtex
@article{bai2025inficheck,
  title     = {InFi-Check: Interpretable and Fine-Grained Fact-Checking of LLMs},
  author    = {Bai, et al.},
  journal   = {arXiv preprint arXiv:2601.06666},
  year      = {2025},
  url       = {https://arxiv.org/abs/2601.06666}
}
```

```bibtex
@misc{vinficheck2025,
  title     = {VInFi-Check: Interpretable and Fine-Grained Fact-Checking for Vietnamese LLM Summaries},
  author    = {KrisEvK and contributors},
  year      = {2025},
  url       = {https://github.com/KrisEvK/VInfi-Check},
  note      = {University Scientific Research Competition (NCKH)}
}
```

---

## 🙏 Acknowledgements

- [InFi-Check](https://github.com/Phosphor-Bai/InFi-Check) (Bai et al.) — original framework
- [Qwen2.5](https://huggingface.co/Qwen/Qwen2.5-7B-Instruct) — base model for fine-tuning
- [underthesea](https://github.com/undertheseanlp/underthesea) — Vietnamese NLP toolkit
- [Groq](https://groq.com/) — fast inference API for Llama models
- [BBC News Dataset](https://huggingface.co/datasets/gopalkalpande/bbc-news-summary) · [DetNet](https://github.com/yumoxu/detnet)

---

<div align="center">
  <sub>Made with ❤️ for Vietnamese NLP · NCKH 2024–2025</sub>
</div>
