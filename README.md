# 🔍 VInFi-Check

**Interpretable and Fine-Grained Fact-Checking for Vietnamese LLM Summaries**

[![Paper](https://img.shields.io/badge/Based%20on-InFi--Check%20arXiv%3A2601.06666-b31b1b?style=flat-square&logo=arxiv)](https://arxiv.org/abs/2601.06666)
[![GitHub](https://img.shields.io/badge/GitHub-KrisEvK%2FVInfi--Check-181717?style=flat-square&logo=github)](https://github.com/KrisEvK/VInfi-Check)
![Language](https://img.shields.io/badge/Language-Vietnamese-blue?style=flat-square)
![Model](https://img.shields.io/badge/Model-Qwen2.5--7B%20%2B%20QLoRA-orange?style=flat-square)
![Status](https://img.shields.io/badge/Status-In%20Progress-yellow?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)

Vietnamese adaptation of [InFi-Check](https://github.com/Phosphor-Bai/InFi-Check) — fine-grained hallucination detection in LLM-generated summaries over Vietnamese news text, with a 6-class error taxonomy, majority-vote evaluation, and a QLoRA-tuned Qwen2.5-7B fact-checker.

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

1. **A Vietnamese hallucination benchmark** — the first systematic evaluation of hallucination patterns across multiple LLMs on Vietnamese news text.
2. **A fine-grained Vietnamese fact-checking model** — Qwen2.5-7B-Instruct fine-tuned with QLoRA, trained on a synthetically constructed dataset with 6 error types.
3. **A full data pipeline** — from Vietnamese document collection → summary generation → multi-model majority-vote verification → structured error injection → SFT dataset preparation.

> **Research context:** University Scientific Research Competition (NCKH), 2024–2025.

---

## ⚙️ Pipeline

```
Bước 1 — Thu thập tài liệu
         VnExpress · BBC News · DetNet Wikipedia (vi)
         Lọc: 300–1000 từ
                    ↓
Bước 2 — Sinh tóm tắt
         deepseek-chat
         Ràng buộc: 100–200 từ, giữ đủ thực thể quan trọng
                    ↓
Bước 3 — Đánh giá & trích xuất câu tham chiếu
         Majority voting: GPT-4o-mini · Qwen2.5-72B · Gemini Flash
         → tóm tắt được xác nhận + câu nguồn
                    ↓
Bước 4 — Tạo lỗi nhân tạo có cấu trúc
         deepseek-chat → 6 loại lỗi × nhiều phương pháp
                    ↓
Bước 5 — Xây dựng dataset SFT
         prepare_dataset_pipeline.ipynb
         Đầu ra: train / valid / test JSONL
                    ↓
Bước 6 — Fine-tune QLoRA
         Qwen2.5-7B-Instruct · rank=16 · alpha=32
         Google Colab T4 · 3 epochs
                    ↓
Bước 7 — Demo Gradio
         VInFiCheck_Gradio.ipynb · public link qua share=True
```

---

## 🗂 Error Taxonomy

| # | Error Type | Tiếng Việt | Mô tả | Ví dụ |
|---|-----------|------------|-------|-------|
| 1 | **Predicate Error** | Lỗi Vị Ngữ | Sai động từ/vị ngữ | *"tăng"* → *"giảm"* |
| 2 | **Entity Error** | Lỗi Thực Thể | Thiếu, nhầm hoặc nén thực thể | Bỏ mất *"Xiaomi"* khỏi danh sách |
| 3 | **Circumstance Error** | Lỗi Hoàn Cảnh | Sai thời gian, địa điểm, con số | *"2022"* → *"2020"* |
| 4 | **Co-reference Error** | Lỗi Đồng Tham Chiếu | Nhầm đại từ hoặc gộp chủ ngữ sai | *"ông ấy"* dùng nhầm cho người khác |
| 5 | **Discourse Link Error** | Lỗi Liên Kết Diễn Ngôn | Đảo ngược quan hệ nhân quả / thời gian | Nguyên nhân và kết quả bị hoán đổi |
| 6 | **Extrinsic Error** | Lỗi Ngoại Lai | Thêm thông tin không có trong tài liệu | Thêm thống kê hallucinated |

---

## 📦 Dataset

| Split | Tài liệu | Mẫu tích cực | Mẫu tiêu cực |
|-------|----------|-------------|--------------|
| Train | ~600 | ~600 | ~3 000 |
| Valid | 100 | ~100 | ~500 |
| Test  | 100 | ~100 | ~500 |

**Nguồn dữ liệu:**
- [BBC News Articles](https://huggingface.co/datasets/gopalkalpande/bbc-news-summary) — business, entertainment, politics, sport, tech
- [DetNet Wikipedia (vi)](https://github.com/yumoxu/detnet) — BUS, GOV, HEA, LAW, LIF, MIL, GEN
- VnExpress (thu thập thủ công)

**Phương pháp tạo lỗi:**

| Loại lỗi | Phương pháp |
|---------|-------------|
| Predicate | Modifying Predictions, Swapping Relation Maskings |
| Entity | Swapping Entities, Compressing Words |
| Circumstance | Swapping Numbers, Swapping Locations/Times |
| Co-reference | Swapping Pronouns, Merging Sentences |
| Discourse Link | Reverse Logical Relationship |
| Extrinsic | Introducing Extrinsic Information |

---

## 🤖 Models

### Sinh tóm tắt (Pipeline)
| Model | Vai trò | API |
|-------|---------|-----|
| `deepseek-chat` | Sinh tóm tắt tiếng Việt | DeepSeek |

### Đánh giá — Majority Vote
| Model | Vai trò |
|-------|---------|
| `gpt-4o-mini` | Evaluator + revision |
| `Qwen/Qwen2.5-72B-Instruct` | Evaluator |
| `gemini-flash` | Evaluator |

### Tạo lỗi nhân tạo
| Model | Vai trò |
|-------|---------|
| `deepseek-chat` | Structured error injection (6 loại) |

### Fact-Checking — Fine-tuned
| Thành phần | Chi tiết |
|-----------|---------|
| Base model | `Qwen2.5-7B-Instruct` |
| Adapter | `sunflowerbiii/infi-check-qwen25-7b-qlora-c` |
| Method | QLoRA — rank=16, alpha=32, dropout=0.05 |
| Training | Google Colab T4, 3 epochs, batch=4 |

### Demo Gradio
| Model | Vai trò | API |
|-------|---------|-----|
| `llama-3.3-70b-versatile` | Sinh tóm tắt trong demo | Groq |
| `gpt-4o-mini` | Fact-checking qua API | OpenAI |

---

## 📁 Repository Structure

```
VInfi-Check/
│
├── Phosphor-Bai-InFi-Check/                  # Codebase chính (based on InFi-Check)
│   │
│   ├── InFi-Check construct/                 # Scripts xây dựng dữ liệu
│   │   ├── summary_eval_prompt/              # Prompt đánh giá (find_support, critics)
│   │   ├── summary_gen_prompt/               # Prompt tạo lỗi nhân tạo
│   │   │   ├── structured_intrinsic/
│   │   │   │   ├── semantic frame/           # Predicate, Entity, Circumstance
│   │   │   │   └── discourse/                # Co-reference, Discourse Link
│   │   │   └── structured_extrinsic/         # Extrinsic error prompts
│   │   ├── eval_and_reference_gen.py         # Bước 3: Đánh giá + trích xuất tham chiếu
│   │   ├── structured_dataset_gen.py         # Bước 4: Tạo lỗi nhân tạo (6 loại)
│   │   └── summary_gen.py                    # Bước 2: Sinh tóm tắt
│   │
│   └── training_dataset_construct/           # Xây dựng dataset SFT
│       ├── prompt/                           # SFT prompt templates & few-shot examples
│       └── prepare_dataset_pipeline.ipynb    # Bước 5: Xuất train/valid/test JSONL
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

**API keys** (thêm vào Colab Secrets hoặc `.env`):

```
GROQ_API_KEY=...
OPENAI_API_KEY=...
DEEPSEEK_API_KEY=...
```

---

## 🚀 Quick Start

### 1. Sinh tóm tắt

```bash
python "Phosphor-Bai-InFi-Check/InFi-Check construct/summary_gen.py"
```

### 2. Đánh giá & trích xuất câu tham chiếu

```bash
python "Phosphor-Bai-InFi-Check/InFi-Check construct/eval_and_reference_gen.py"
```

### 3. Tạo dataset lỗi nhân tạo

```bash
python "Phosphor-Bai-InFi-Check/InFi-Check construct/structured_dataset_gen.py"
```

### 4. Xây dựng dataset SFT

Chạy `Phosphor-Bai-InFi-Check/training_dataset_construct/prepare_dataset_pipeline.ipynb` trên Google Colab.

```
Output:
  sft_dataset/jsonl/summary_sft_train_pos1neg5_with_ref.jsonl
  sft_dataset/jsonl/summary_sft_valid_with_ref.jsonl
  sft_dataset/jsonl/summary_sft_test_with_ref.jsonl
```

### 5. Fine-tune

Chạy notebook fine-tune trên Google Colab (T4 GPU) với adapter `sunflowerbiii/infi-check-qwen25-7b-qlora-c`.

---

## 🎮 Gradio Demo

```bash
# Trên Google Colab — chạy VInFiCheck_Gradio.ipynb
```

Demo hỗ trợ:
- **✦ Sinh tóm tắt** — tự động sinh tóm tắt từ văn bản gốc (Llama-3.3-70B via Groq)
- **⬡ Kiểm chứng** — kiểm tra từng câu với phân loại 6 loại lỗi
- **◈ So sánh mô hình** — so sánh song song: VInFi-Checker · GPT-4o-mini · Llama-3.3-70B

---

## 📄 Citation

```bibtex
@article{bai2025inficheck,
  title   = {InFi-Check: Interpretable and Fine-Grained Fact-Checking of LLMs},
  author  = {Bai, et al.},
  journal = {arXiv preprint arXiv:2601.06666},
  year    = {2025},
  url     = {https://arxiv.org/abs/2601.06666}
}
```

```bibtex
@misc{vinficheck2025,
  title  = {VInFi-Check: Interpretable and Fine-Grained Fact-Checking for Vietnamese LLM Summaries},
  author = {KrisEvK and contributors},
  year   = {2025},
  url    = {https://github.com/KrisEvK/VInfi-Check},
  note   = {University Scientific Research Competition (NCKH)}
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

*Made with ❤️ for Vietnamese NLP · NCKH 2024–2025*
