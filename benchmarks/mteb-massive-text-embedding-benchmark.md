---
title: MTEB（Massive Text Embedding Benchmark, 行业标准 embedding eval）
type: benchmark
sources: [muennighoff-2023-mteb]
related: [../concepts/clip.md, ../concepts/matryoshka-embedding.md, ../concepts/splade-sparse-retrieval.md, ../concepts/colbertv2.md, ../topics/sparse-dense-hybrid-retrieval.md, ../topics/multimodal-embedding-retrieval.md, ../topics/adaptive-retrieval-shortlist-rerank.md, ../topics/ann-benchmarking-methodology.md]
created: 2026-05-12
updated: 2026-05-21 (邻域 benchmark cross-link: ANN benchmarking methodology)
---

# MTEB: Massive Text Embedding Benchmark

**TL;DR**: Muennighoff et al. 2023 EACL [muennighoff-2023-mteb] (HuggingFace + Cohere) 提出**行业标准 text embedding evaluation benchmark**——8 个 embedding task × 58 datasets × 112 languages, 33 model 系统化对比. **对 wiki 内 vector DBs 的核心价值**: (1) **wiki 内首个 embedding model selection benchmark**——之前 wiki 无任何 embedding-side eval framework, MTEB 填补"选哪个 embedding model"客观依据空白; (2) **Public leaderboard at github.com/embeddings-benchmark/mteb** — 持续更新, OpenAI text-embedding-3 / Cohere v3-v4 / Voyage / BGE / NV-Embed / Linq / SFR / GTE / Jina 等 production embedding model **全部 MTEB 评测**, leaderboard 是 "production embedding model 选哪个"事实标准; (3) **8 task type cover production retrieval** workload: Classification / Clustering / PairClassification / Reranking / Retrieval / STS / Summarization / BitextMining——其中 Retrieval task 直接对接 vector DB workload; (4) **"No model dominates all tasks" 关键 finding**——不同 embedding model 在不同 task 上 trade-off, **production 通常 task-aware 选 model**, 不存在 universal best embedding model; (5) **跨 model migration cost 客观依据**: MTEB score gap 量化 "升级 embedding 是否值得" decision; (6) **MMTEB 扩展**: 多语言 MTEB (arXiv 2502.13595) 覆盖 112+ languages production embedding multilingual case. **Talk relevance**: live demo SIGMOD 2026 cross-model migration 主题 — MTEB 是评估 "升级前后 embedding 性能差异" 标准 framework.

## 实验设置

[per muennighoff-2023-mteb §2-3]

### 8 embedding task

| Task | 输入 → 输出 | Production workload 对应 |
|---|---|---|
| **Classification** | text → label (linear probe on embedding) | content moderation / category 分类 |
| **Clustering** | texts → groups (k-means on embeddings) | topic discovery / dedup |
| **PairClassification** | (text_A, text_B) → binary label | duplicate detection / paraphrase |
| **Reranking** | (query, docs) → ordered docs | RAG 2nd-stage rerank |
| **Retrieval** | query → top-k docs | **vector DB primary workload** |
| **STS** (Semantic Textual Similarity) | (text_A, text_B) → similarity score | embedding quality direct |
| **Summarization** | (doc, machine-summary) → quality score | summarization eval |
| **BitextMining** | parallel multilingual texts → matching | translation pair mining |

### 58 datasets

涵盖 8 task × diverse domains: 学术 / web / 医学 / 财经 / 法律 / 客服 / 编程 / etc. 112 languages.

### 33 model evaluated (paper time)

包括 Sentence-BERT / SimCSE / GTR / E5 / Instructor / OpenAI text-embedding-ada-002 等. **leaderboard 持续更新**, 当前 production 主流 (OpenAI text-embedding-3 / Cohere v3-v4 / Voyage / BGE / NV-Embed 等) 全部 MTEB 评测.

## 结果（updated production reality）

[per MTEB leaderboard + community knowledge]

**Key paper finding**: "no particular text embedding method dominates across all tasks"
- Task-specific best model 各不相同
- Production embedding model 选择 = **task-aware decision**

**Current production embedding model 主流** (per MTEB leaderboard ~ 2024-2025):
- **NV-Embed** / **Linq-Embed-Mistral** / **SFR-Embedding-Mistral**: top MTEB scores
- **OpenAI text-embedding-3-large** (3072-d MRL): commercial standard
- **Cohere embed-v3 / v4**: commercial multilingual
- **Voyage voyage-3 / voyage-multimodal-3**: commercial finance/legal
- **BGE-M3** (BAAI): OSS, hybrid sparse+dense+colbert single model
- **GTE / Jina v3 / mxbai**: OSS production-ready

## 可信度评估

[per muennighoff-2023-mteb]

**优势**:
- Comprehensive: 8 task × 58 datasets 是 paper-time 最 comprehensive
- Open leaderboard: 持续更新 community vetted
- Standardized eval: 不同 model 在 same protocol 下 fair compare
- Multilingual coverage: 112 languages

**Limitations** (per community feedback):
- Retrieval task (vector DB primary workload) 仅 BEIR datasets, 不包含 production-specific case
- LoTTE-style long-tail 不充分 covered (ColBERTv2 paper 引入 LoTTE 补充)
- Embedding-only eval, 不包括 hybrid retrieval system-level evaluation
- Multimodal embedding (CLIP / SigLIP / ImageBind) 不在 scope (text-only)

## 与 wiki 内其他 ingest 的关系

[per CLIP / MRL / SPLADE / ColBERTv2 / BGE-M3 (TBD) 等]

### MTEB 评估涵盖

- **Dense embedding** ([CLIP](../concepts/clip.md) / [MRL](../concepts/matryoshka-embedding.md)-trained / Voyage / Cohere / etc.): retrieval + STS + reranking 等 task
- **Sparse embedding** ([SPLADE](../concepts/splade-sparse-retrieval.md)): retrieval task (BEIR subset within MTEB)
- **Late-interaction** ([ColBERT v2](../concepts/colbertv2.md)): retrieval task (虽 ColBERT 输出多 vector, MTEB 仅 single-vector retrieval baseline 主流)

### MTEB → BGE-M3 production unified

BGE-M3 (BAAI 2024) 把 dense + sparse + ColBERT-style 三 vector type 集成 single model, MTEB retrieval 任务 multi-axis 评估首次 production-validated. 详见 [topics/sparse-dense-hybrid-retrieval.md](../topics/sparse-dense-hybrid-retrieval.md).

### MTEB 在 vendor 端

MTEB **不是 vector DB benchmark**, 是 embedding model benchmark. 但所有 vector DB vendor 都把 MTEB leaderboard 作 "客户选 embedding model 时 reference":
- [Turbopuffer](../systems/turbopuffer.md) docs §performance 推荐 voyage-4 / embed-v4 / Qwen3-VL-Embedding-8B (per their MTEB scores)
- [Chroma](../systems/chroma.md) docs 推荐 ChromaCloudSpladeEmbeddingFunction + 多 MTEB-validated embedding model
- 其他 vendor 类似

## Open Questions

- **MTEB Retrieval task vs BEIR independent**: 重叠程度 + production realism 不公开
- **MMTEB multilingual extension**: 实际 production multilingual case adoption 不公开
- **MTEB vector-DB-aware extension**: 是否有 "embedding × vector DB" joint benchmark (考虑 ANN approximation effect)? Not in original MTEB scope
- **Multimodal MTEB**: text-only; 图像 / video / audio embedding 公认 benchmark 缺
- **Embedding model + quantization joint MTEB**: 量化后 embedding 在 MTEB 上 quality 退化曲线 不公开
- **MTEB ↔ ColBERTv2 / BGE-M3 三-way retrieval evaluation**: MTEB 当前主要 single-vector retrieval, 3-way hybrid 评估缺
- **Cost-aware MTEB**: embedding 性能 vs API cost / latency / model size trade-off 不在 scope

Cited by: [queries/hybrid-retrieval-benchmark-landscape.md](../queries/hybrid-retrieval-benchmark-landscape.md)
