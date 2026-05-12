---
title: Sparse-Dense Hybrid Retrieval（稀疏 + 稠密混合检索）
type: topic
sources: [formal-2021-splade-v2, vespa-docs, weaviate-docs, turbopuffer-docs, pinecone-docs]
related: [../concepts/splade-sparse-retrieval.md, ../concepts/clip.md, ../concepts/matryoshka-embedding.md, ../concepts/product-quantization.md, ../systems/vespa.md, ../systems/weaviate.md, ../systems/turbopuffer.md, ../systems/pinecone.md, ../systems/milvus.md, ../systems/qdrant.md, adaptive-retrieval-shortlist-rerank.md, multimodal-embedding-retrieval.md, attribute-filtering.md, multi-vector-queries.md, index-selection.md]
created: 2026-05-12
updated: 2026-05-12
---

# Sparse-Dense Hybrid Retrieval（稀疏 + 稠密混合检索）

**TL;DR**: 现代 IR / RAG production retrieval pipeline 普遍**同时使用 sparse + dense 两种 retrieval**——sparse 提供 exact-match 与 interpretability (BM25 / SPLADE / DeepImpact), dense 提供 semantic similarity 与 expansion (CLIP / MRL / BGE / Voyage). 二者通过 **score fusion** (reciprocal-rank fusion / α-blend / rank-profile aggregation) 在 first-stage retrieval 一起 contribute, 之后 (optional) cross-encoder rerank 形成 multi-stage AR pipeline (per [topics/adaptive-retrieval-shortlist-rerank.md](./adaptive-retrieval-shortlist-rerank.md)). **wiki 内 vector DB hybrid 现状**: 5 vendor 全部 native 支持 sparse + dense 混合, 但**实现哲学显著不同**——Vespa first-class rank-profile (BM25 + SPLADE + dense + ML rerank 同 schema), Weaviate first-class `hybrid()` API + BlockMaxWAND BM25, Pinecone Sparse-Dense Hybrid Index, Turbopuffer multi-query + application RRF, Milvus 多 vector field + 应用层 fusion. **关键 frontier**: production state-of-the-art hybrid retrieval = **SPLADE sparse + CLIP/MRL dense + cross-encoder rerank**, vector DB 端无 vendor 把这套**统一 first-class production primitive**, Vespa 最接近.

## 问题陈述

[per formal-2021-splade-v2 §1 + production RAG reality]

Pure dense retrieval (BERT Siamese / CLIP / MRL):
- ✗ Vocabulary mismatch resilient — 解决 BM25 "查询/文档无共同词" 痛点
- ✗ **Exact match capability 丢失** — 罕见 token (产品 SKU / email / 数字 / 化学式) 退化
- ✗ ANN search cost — high-d dense index 计算成本高
- ✗ Black-box — 不可解释 (vs term weight)

Pure sparse retrieval (BM25):
- ✓ Inverted index 极其高效 (50 年成熟)
- ✓ Exact match — 罕见 token 自然处理
- ✓ Interpretable — term weight 直接观察
- ✗ Vocabulary mismatch — 查询/文档无共同词时 0 召回
- ✗ Semantic 失败 — "automobile" vs "car" 不能匹配 (无 expansion)

**Hybrid = 二者互补**:
- Sparse path 处理罕见 token / exact match / interpretability
- Dense path 处理 semantic similarity / cross-lingual / expansion
- Score fusion 让两路结果合并 → 比 single path 更高 recall + precision

[per formal-2021-splade-v2 §4 results]: DistilSPLADE-max (sparse with neural expansion) **已经 alone 击败 dense SOTA on BEIR** — 但实际 production hybrid 仍组合使用 sparse + dense 因为不同 corpus / workload sweet spot 不同.

## 相关概念

- **[SPLADE](../concepts/splade-sparse-retrieval.md)**: sparse-side neural retrieval, BM25 现代替代
- **[CLIP](../concepts/clip.md)**: dense-side multimodal foundational
- **[Matryoshka](../concepts/matryoshka-embedding.md)**: dense-side multi-tier embedding compression
- **BM25**: classical sparse baseline, BlockMaxWAND optimization (Weaviate 默认)
- **doc2query-T5 / DeepImpact / DeepCT**: SPLADE family 前辈
- **ColBERT** (Khattab 2020, wiki 未 ingest): late-interaction sparse-dense bridge
- **BGE-M3** (wiki 未 ingest): 单 model 同时输出 sparse + dense + ColBERT-style 多模态向量

## 工业方案对比

### 5 vendor hybrid 实现哲学

[per wiki 内 vendor docs + SPLADE production usage]

| Vendor | Sparse path | Dense path | Hybrid API | Fusion 策略 |
|---|---|---|---|---|
| **Vespa** | **BlockMaxWAND BM25 + SPLADE weightedset + WAND/weakAnd** | tensor framework (CLIP/MRL native) | **rank-profile (first-class)** | YQL planner + ranking expression (BM25 + dot_product(splade) + closeness(vector)) |
| **Weaviate** | **BlockMaxWAND BM25** | HNSW + RQ8/BQ/PQ | **`collection.query.hybrid(alpha=...)`** (first-class) | α-blend (default) OR relative score fusion |
| **Pinecone** | Sparse Vector Index | Dense Vector Index | **Sparse-Dense Hybrid Index** | server-side fusion (公式不公开) |
| **Turbopuffer** | BM25 (BlockMaxWAND-style) | SPFresh + cosine ANN | **`multi_query` API** | **application-side RRF (推到客户端)** |
| **Milvus** | sparse_inverted_index + SPLADE-compatible | 多 vector field (HNSW/DISKANN/CAGRA) | multi-vector query | application fusion |
| **Qdrant** | sparse vector (v1.7+) | HNSW + multi-vector | multi-vector hybrid query | server-side fusion |

→ **Vespa 唯一同时 first-class native** BM25 + SPLADE-style weightedset + dense tensor + 4-phase ranking (sparse-dense-rerank-global) **全 schema 一等公民**.

→ **Weaviate / Pinecone first-class hybrid API** (单 API call 内置 sparse + dense)

→ **Turbopuffer 哲学最 minimal**: 推到 application layer + RRF.

→ **Milvus / Qdrant** 多 vector field, hybrid 路径 through application.

### Score Fusion 策略全景

[per formal-2021-splade-v2 + Weaviate docs + Vespa rank-profile]

**1. α-blend (linear weighted sum)**:
```
score = α × dense_score + (1-α) × sparse_score
```
- Weaviate default; α=0 纯 BM25, α=1 纯 vector
- 简单但需 normalize 不同 score 量纲 (BM25 raw vs cosine [-1,1])

**2. Reciprocal Rank Fusion (RRF)**:
```
score = Σ_r 1/(k + rank_r)  for r ∈ {sparse, dense}, k=60 typical
```
- Turbopuffer / 应用层主流, no normalize 需求
- 对极端 score outlier 鲁棒

**3. Relative Score Fusion**:
```
score = (s - s_min) / (s_max - s_min) for each path, then sum
```
- Weaviate alternative, per-query normalize

**4. Vespa rank-profile expression**:
```
expression: 0.4*bm25(field) + 0.4*dot_product(splade_field, query_splade) + 0.2*closeness(vector_field)
```
- 任意可执行 expression (constant 加权或学习的 LR coefficient)
- 是 wiki 内最 flexible

**5. ML-trained fusion (LR / GBDT)**:
- production 实际 case: LR / GBDT 学 sparse/dense/freshness/popularity 多 signal
- Vespa global-phase ONNX 一等公民; 其他 vendor 推到 application

## 历史 hybrid retrieval 路径

[per formal-2021-splade-v2 §2 + IR history]

**1980s-1990s**: BM25 / TF-IDF + LSI / pLSA (latent semantic) — 早期 sparse-dense 雏形, 但 dense path uncompetitive  
**2010s early**: word2vec / GloVe + BM25 — fixed-feature dense + classical sparse, 应用层 score combination  
**2018-2019 BERT-era**: BM25 + dense BERT Siamese — DPR / Sentence-BERT 出现, hybrid 标配  
**2020 DeepCT / doc2query-T5**: sparse-side 开始 BERT-augmented (term weight learned)  
**2021 SPLADE**: 完整 neural sparse + neural expansion, 单 sparse model 接近 dense SOTA  
**2022 ColBERT v2 / PLAID**: late-interaction (token-level dense) 进一步 bridge  
**2024 BGE-M3**: 单 model 同时输出 sparse + dense + ColBERT-style (sparse-dense-late-interaction 三合一)  
**当前 production**: SPLADE + CLIP/MRL/Voyage/Cohere + cross-encoder rerank, 多 stage AR pipeline

## 各 wiki vendor 推荐 hybrid 配置

[per wiki 内 vendor + SPLADE + Phase 1-2 (CLIP/MRL)]

| Workload | 推荐 stack |
|---|---|
| E-commerce / RAG production | **Vespa**: BM25 + SPLADE + CLIP/MRL dense + rank-profile + global-phase cross-encoder rerank |
| Multimodal hybrid (text + image) | **Vespa**: CLIP image embedding + SPLADE text + tensor framework |
| Per-tenant SaaS hybrid | **Turbopuffer**: namespace per tenant + multi_query (BM25 + dense) + application RRF |
| AI-native primary DB + agent stack | **Weaviate**: BlockMaxWAND BM25 + RQ8 dense + first-class `hybrid()` API |
| Managed simplicity + hybrid | **Pinecone**: Sparse-Dense Hybrid Index (闭源 fusion) |
| 多 index_type 选择空间 | **Milvus**: sparse_inverted + 多 vector field + application fusion |

## Open Questions

- **SPLADE 与 BlockMaxWAND BM25 production migration cost**: Weaviate / Vespa / Turbopuffer 客户从 BM25 → SPLADE 实际 quality gain vs infrastructure cost 不公开
- **Hybrid α / RRF k 自适应优化**: 当前 α (Weaviate) 与 k (RRF) 是 ad-hoc; per-workload optimal 选择算法 不存在公开
- **Cross-encoder rerank cost vs hybrid quality gain trade-off**: 在 hybrid + rerank pipeline 内, rerank 何时 worth latency cost? 不公开
- **BGE-M3-style 单 model 同时输出 sparse + dense + late-interaction**: vector DB 端集成 (single namespace 3 vector types) production case zero
- **Hybrid retrieval × MRL prefix shortlist combine**: dense path 用 MRL 256-d prefix shortlist + sparse SPLADE shortlist 并 fuse → 应用层 AR pipeline; production case 不公开
- **Sparse retrieval × multimodal**: SPLADE 是 text-only; image/audio sparse retrieval (类似但 sparse-side) production case wiki zero
- **GPU sparse retrieval**: 全 wiki sparse path 都假设 CPU + inverted index; GPU SPLADE / GPU BM25 production case zero
- **Hybrid retrieval × spatial**: 5 vendor 仅 Vespa native spatial + 全 vendor native hybrid → 唯一可同时 tri-modal (text BM25 + dense vector + spatial bbox) 的 system. 实测 production case zero
- **DistilSPLADE-max BEIR 11/14 best 在 production 真实 case 是否复现**: 论文 zero-shot BEIR 仅 test set; production 真实 query distribution / re-train 周期 / fine-tune cost 不公开
- **SPLADE 在 long-context (8K+ token)**: BERT 256 token max → chunk + per-chunk SPLADE; chunk-level vs full-doc-level retrieval 论文不深入
- **Hybrid retrieval 公平 benchmark methodology**: 类似 `vector-scalar-bench-methodology` query, hybrid 现 zero coverage of fair head-to-head between SPLADE+CLIP+RRF (Turbopuffer-style) vs Vespa rank-profile vs Weaviate hybrid(α)
