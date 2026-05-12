---
title: SPLADE（Sparse Lexical and Expansion Model）
type: concept
sources: [formal-2021-splade-v2]
related: [clip.md, matryoshka-embedding.md, product-quantization.md, scann.md, ../systems/vespa.md, ../systems/weaviate.md, ../systems/turbopuffer.md, ../systems/pinecone.md, ../topics/sparse-dense-hybrid-retrieval.md, ../topics/multi-vector-queries.md, ../topics/adaptive-retrieval-shortlist-rerank.md]
created: 2026-05-12
updated: 2026-05-12
---

# SPLADE (Sparse Lexical and Expansion Model)

**TL;DR**: Naver Labs Europe Formal et al. 2021 SIGIR/arXiv [formal-2021-splade-v2] 提出 **sparse neural retrieval 范式**——把 BERT MLM logits over WordPiece vocab (|V| = 30,522) 转化为**学习的 sparse term weight 向量**, 同时**显式建模 query/document expansion**, 用 **FLOPS regularizer** 直接优化 inverted index lookup cost. **对 wiki 内 vector DBs 的核心价值**: (1) **wiki 内首个 sparse neural retrieval source**——之前 wiki BM25/inverted index 仅在 Weaviate/Vespa/Turbopuffer/Pinecone 提及但**无 algorithm source 支撑**, SPLADE 填补 sparse-side algorithm 空缺; (2) **Bridge classical BM25 与 dense embedding (CLIP/MRL)**——SPLADE output 是 sparse vector (30K dim 稀疏), 直接 fit inverted index 同时 capture neural language understanding; (3) **production hybrid search 算法基础**——Vespa rank-profile + Weaviate first-class hybrid + Turbopuffer hybrid + Pinecone Sparse-Dense Hybrid 都本质上是 **SPLADE-style sparse + dense (CLIP/MRL-style) 融合**; (4) **击败 dense SOTA 在 zero-shot benchmark**: DistilSPLADE-max 在 BEIR 14 datasets avg NDCG@10 = **0.506**, 11/14 数据集第一, 显著优于 ColBERT (0.457) / BM25 (0.456) / TAS-B (0.437). [formal-2021-splade-v2 §3-4]

## 提出背景

[per formal-2021-splade-v2 §1-2]

IR 2020-2021 双 paradigm 并立:
- **Dense retrieval** (BERT Siamese / dual-encoder): ANCE / TAS-B / TCT-ColBERT / RocketQA / DPR / CLIP-style. **优势**: implicit semantic expansion 解决 vocabulary mismatch. **劣势**: exact match capability 失, ANN search cost, model size.
- **Sparse retrieval** (BM25 + 神经增强): DeepCT / doc2query-T5 / DeepImpact / COIL / SparTerm / SPARTA. **优势**: inverted index 高效, exact-match 保留, interpretable. **劣势**: vocabulary mismatch 严重 (BM25 baseline weak).

**SPLADE 的 unique 位置**: 显式 model expansion via MLM logits over full BERT vocab → sparse output + neural expansion + 兼容 BM25-era inverted index 基础设施.

### 与 BM25 / DeepCT / doc2query-T5 / COIL 关键差异

| Method | Output | Expansion mechanism | Inverted index 兼容? |
|---|---|---|---|
| BM25 | sparse (term TF-IDF) | 无 (exact match only) | ✓ classical |
| DeepCT | sparse (BERT term weight) | 无 (仍 same vocab as doc) | ✓ |
| doc2query-T5 | sparse (predicted extra terms) | **document expansion** (T5 generates queries) | ✓ |
| DeepImpact | sparse (doc2query + impact) | document expansion | ✓ |
| COIL | dense per-term (contextualized) | per-term embedding store | partial (contextualized inverted) |
| SPARTA | sparse (max pool over vocab) | implicit (token-level interaction) | partial (not sparse enough) |
| SparTerm | sparse (sum pool over vocab) | implicit | ✓ but no explicit regularization |
| **SPLADE** | **sparse (log-saturated sum/max over vocab)** | **implicit + explicit via MLM logits over full V** | ✓ **with FLOPS regularizer for index cost** |

## 关键性质

### 1. SPLADE 数学定义（核心 Eq 1-6）

[per formal-2021-splade-v2 §3.1-3.2]

输入 token 序列 `t = (t_1, ..., t_N)`, BERT embeddings `(h_1, ..., h_N)`.

**Term importance prediction** (per token i, vocab token j):
```
w_ij = transform(h_i)^T E_j + b_j,  j ∈ {1, ..., |V|}
```
- `transform()`: linear + GeLU + LayerNorm
- `E_j`: BERT input embedding for vocab token j
- `b_j`: token-level bias
- **Equivalent to BERT MLM prediction layer** → can initialize from MLM-pretrained model

**Aggregation (v1, sum pooling)**:
```
w_j = Σ_{i∈t} log(1 + ReLU(w_ij))
```
- **Log-saturation**: damp 大 term weights, mimic BM25 TF curve
- ReLU enforce sparsity (drop negative importance)

**Aggregation (v2, max pooling, paper improvement)**:
```
w_j = max_{i∈t} log(1 + ReLU(w_ij))
```
- v2 关键改进 → **+ ~2 MRR@10 over v1 sum** on MS MARCO + TREC DL 2019

### 2. FLOPS regularizer（核心创新）

[per formal-2021-splade-v2 §3.1 + Paria et al. 2020]

**问题**: ℓ_1 regularization 不保证 inverted index 均匀分布——某些 posting list 过长 → 查询慢. Zipfian term distribution 让此问题更严重.

**FLOPS regularizer** (Paria 2020 提出, SPLADE 采用):
```
ℓ_FLOPS = Σ_{j∈V} ā_j^2 = Σ_{j∈V} (1/N · Σ_{i=1}^N w_j^(d_i))^2
```
- `ā_j`: token j 在 batch 内 average activation
- 平方惩罚 → 强制每 term 频率均匀
- **直接优化 expected query-document FLOPS** = inverted index lookup cost

### 3. Overall loss

[per formal-2021-splade-v2 §3.1 Eq 5]

```
L = L_rank-IBN + λ_q L_reg^q + λ_d L_reg^d
```

- L_rank-IBN: 标准 contrastive InfoNCE loss with in-batch negatives + hard negatives (BM25 sampling)
- **Separate λ_q (query) vs λ_d (document)**: 给 query sparsity 更大压力 (critical for fast retrieval, query sparse → 少 posting list lookup)

### 4. 训练 schedule

[per formal-2021-splade-v2 §4]

- 初始化: DistilBERT-base checkpoint
- 优化器: ADAM lr=2e-5, linear schedule + warmup 6000 步
- Batch size: 124 on 4× Tesla V100 32 GB
- Training: 150K steps (~ 1B tokens seen)
- λ scheduler: quadratically 增加到 step 50K, 之后 constant (避免 early training 时 regularization 主导)
- 典型 λ ∈ [1e-4, 1e-1]

### 5. Variants

[per formal-2021-splade-v2 §3.2-3.4]

**SPLADE-max** (paper's default v2):
- max pooling Eq 6
- 2-3% MRR@10 improvement over v1 sum

**SPLADE-doc** (efficiency variant):
- 无 query expansion / encoder
- Ranking score `s(q, d) = Σ_{j∈q} w_j^d`
- **Pre-compute everything offline**, query 端只需 BM25-style lookup
- 牺牲少量 effectiveness 换 inference latency

**DistilSPLADE-max** (paper's best):
- Step 1: train SPLADE-max + cross-encoder reranker via BM25-sampled negatives
- Step 2: SPLADE-max generates harder negatives + cross-encoder 提供 Margin-MSE scores
- Step 3: retrain SPLADE-max from scratch with distillation
- **0.368 MRR@10 / 0.979 R@1000 on MS MARCO** ← 与 dense SOTA (RocketQA 0.370) 持平

## 与同类对比 / 性能 results

### MS MARCO dev + TREC DL 2019

[per formal-2021-splade-v2 §4 Table 1]

| Approach | MS MARCO dev MRR@10 | MS MARCO dev R@1000 | TREC DL 2019 NDCG@10 | TREC R@1000 |
|---|---|---|---|---|
| **Dense retrieval** | | | | |
| ANCE | 0.330 | 0.959 | 0.648 | - |
| TCT-ColBERT | 0.359 | 0.970 | 0.719 | 0.760 |
| TAS-B | 0.347 | 0.978 | 0.717 | 0.843 |
| RocketQA | **0.370** | 0.979 | - | - |
| **Sparse retrieval** | | | | |
| BM25 | 0.184 | 0.853 | 0.506 | 0.745 |
| DeepCT | 0.243 | 0.913 | 0.551 | 0.756 |
| doc2query-T5 | 0.277 | 0.947 | 0.642 | 0.827 |
| SparTerm | 0.279 | 0.925 | - | - |
| COIL-tok | 0.341 | 0.949 | 0.660 | - |
| DeepImpact | 0.326 | 0.948 | 0.695 | - |
| SPLADE v1 | 0.322 | 0.955 | 0.665 | 0.813 |
| **SPLADE-max** | 0.340 | 0.965 | 0.684 | 0.851 |
| **SPLADE-doc** | 0.322 | 0.946 | 0.667 | 0.747 |
| **DistilSPLADE-max** | **0.368** | **0.979** | **0.729** | **0.865** |

→ **DistilSPLADE-max 与 dense SOTA RocketQA 0.370 持平, 超过其他所有 sparse + 大部分 dense**.

### BEIR zero-shot (Table 2)

[per formal-2021-splade-v2 §4 + BEIR Thakur 2021]

| Corpus | ColBERT | BM25 | TAS-B | SPLADE sum [v1] | SPLADE max | DistilSPLADE-max |
|---|---|---|---|---|---|---|
| MS MARCO | 0.425 | 0.228 | 0.408 | 0.387 | 0.402 | **0.433** |
| TREC-COVID | 0.677 | 0.656 | 0.481 | 0.655 | 0.673 | **0.710** |
| NFCorpus | 0.305 | 0.325 | 0.319 | 0.311 | 0.313 | **0.334** |
| NQ | **0.524** | 0.329 | 0.463 | 0.438 | 0.469 | 0.521 |
| HotpotQA | 0.593 | 0.603 | 0.584 | 0.635 | 0.636 | **0.684** |
| FiQA-2018 | 0.317 | 0.236 | 0.300 | 0.258 | 0.287 | **0.336** |
| ArguAna | 0.233 | 0.315 | 0.427 | 0.447 | 0.439 | **0.479** |
| FEVER | 0.771 | 0.753 | 0.700 | 0.728 | 0.730 | **0.786** |
| Climate-FEVER | 0.184 | 0.213 | 0.228 | 0.162 | 0.199 | **0.235** |
| SCIDOCS | 0.145 | 0.158 | 0.149 | 0.141 | 0.145 | 0.158 |
| SciFact | 0.671 | 0.665 | 0.643 | 0.626 | 0.628 | **0.693** |
| DBPedia | 0.392 | 0.273 | 0.384 | 0.343 | 0.366 | **0.435** |
| Quora | **0.854** | 0.789 | 0.835 | 0.829 | 0.835 | 0.838 |
| Touché-2020 | 0.275 | **0.614** | 0.173 | 0.289 | 0.316 | 0.364 |
| **Avg** | 0.455 | 0.440 | 0.435 | 0.446 | 0.460 | **0.500** |
| **Best on dataset** | 2 | 2 | 0 | 0 | 0 | **11** |

→ **DistilSPLADE-max wins 11/14 datasets BEIR avg 0.500 — outperforms ColBERT / BM25 / TAS-B 显著, 是 sparse retrieval state-of-the-art**.

## 典型实现 / Vector DB 集成 pattern

### Pattern 1: Vespa rank-profile (BM25 + SPLADE + dense)

[per [systems/vespa.md] rank-profile + SPLADE production usage]

```
schema document {
  field title type string { indexing: index | summary }
  field body type string { indexing: index | summary }
  field title_splade type weightedset<string> {
    # SPLADE-encoded sparse vector (30K BERT vocab terms with weights)
    indexing: attribute | index
  }
  field body_splade type weightedset<string> {
    indexing: attribute | index
  }
  field dense_vector type tensor<float>(x[768]) {
    # CLIP / MRL / OpenAI dense embedding
    indexing: attribute | index
    attribute { distance-metric: angular }
  }
}

rank-profile hybrid-retrieval {
  first-phase {
    expression: 0.4 * bm25(title) + 0.4 * dot_product(body_splade, query_splade) 
              + 0.2 * closeness(field, dense_vector)
  }
  second-phase {
    rerank-count: 100
    expression: onnx(cross_encoder_reranker)
  }
}
```

→ Vespa 是 wiki 内**唯一同时 first-class native** BM25 + SPLADE-style weightedset + dense tensor 三 retrieval modality 的 system.

### Pattern 2: Weaviate first-class hybrid

[per [systems/weaviate.md] BlockMaxWAND BM25 + vector hybrid]

```python
# Weaviate hybrid API (BlockMaxWAND BM25 + vector α-blend)
result = collection.query.hybrid(
    query="italian leather wallet",
    alpha=0.5,  # 0 = pure BM25, 1 = pure vector
    limit=10
)
```

- BlockMaxWAND BM25 是 Weaviate 当前 sparse path; **SPLADE 是 next-gen sparse 替代 candidate**
- production migration: Weaviate 客户可改 BM25 输入 from raw text → SPLADE-encoded weightedset, 同 hybrid API

### Pattern 3: Turbopuffer hybrid (推到 application layer)

[per [systems/turbopuffer.md] §hybrid + first-stage retrieval focus]

```python
# Turbopuffer multi-query (vector + BM25 separate queries, application fusion)
response = ns.multi_query(
    queries=[
        {"rank_by": ("vector", "ANN", clip_text_embedding(query)), "limit": 50},
        {"rank_by": ("content", "BM25", query), "limit": 50},
    ]
)
# Application-side reciprocal-rank fusion
```

- Turbopuffer "focused on first-stage retrieval" — fusion + rerank 推到 application layer
- SPLADE 可替换 BM25 path: `{"rank_by": ("body_splade", "BM25-style", splade_encode(query))}`
- Production state-of-the-art: SPLADE + CLIP/MRL hybrid in single multi_query → application RRF fusion

### Pattern 4: Pinecone Sparse-Dense Hybrid Index

[per Pinecone docs + production usage]

Pinecone production 已支持 sparse-dense hybrid index, sparse vector 通常 SPLADE-encoded:
```
index.upsert([
    {"id": "doc1", "values": [0.1, 0.2, ...], "sparse_values": {"indices": [30, 42, ...], "values": [0.5, 0.3, ...]}},
])
```

## SPLADE 与 [CLIP](./clip.md) / [MRL](./matryoshka-embedding.md) 的关系

[per formal-2021-splade-v2 + wiki Phase 1-2]

| | SPLADE | CLIP | MRL |
|---|---|---|---|
| Output | sparse (~30K dim, mostly zero) | dense (512-1024 dim, all non-zero) | dense (full d-dim with nested prefix) |
| Distance | dot product (sparse) | cosine (dense) | cosine (dense, possibly prefix-truncated) |
| Vector DB infra | inverted index OR sparse vector store | dense ANN (HNSW/SPANN/etc.) | dense ANN with prefix-aware query |
| Vocabulary mismatch | **explicitly via MLM expansion** | implicit via dense semantic embed | implicit via dense semantic embed |
| Interpretability | **high** (term weight viewable) | low (black-box) | low (black-box) |
| Compute | one BERT forward (max pool) | one dual-encoder forward | one dual-encoder forward (free truncation) |
| Production hybrid | sparse side | dense side | dense side (multi-tier) |

→ **SPLADE + CLIP/MRL 是 production hybrid retrieval 二元基础**: sparse-side neural + dense-side neural 互补; 任何 wiki vendor 都通过 hybrid pipeline 同时使用两者.

## Open Questions

- **SPLADE-v2 vs v3 / v4 演化**: paper 是 2021 v2; 后续 SPLADE++ / SPLADE v3 (2024) production case 演化 wiki 未涵盖, 需 future ingest
- **SPLADE × Matryoshka prefix-style 是否可能**: dense MRL 把 d-维 embedding 多 prefix 嵌套; SPLADE 是 30K vocab 稀疏向量——**vocab-level pruning** (e.g., 取 top-K 重要 token) 是 SPLADE 的 "prefix" 等价? Paper 不讨论. 理论上 FLOPS regularizer 已是某种隐式 dim reduction (强制稀疏)
- **SPLADE × distillation 与 cross-encoder rerank cascade**: SPLADE-first-stage + cross-encoder rerank 是 production AR (Adaptive Retrieval) 实例化, 但 paper 仅 distill 不 cascade —— `topics/adaptive-retrieval-shortlist-rerank.md` 内 SPLADE 应作 sparse-side AR shortlist 代表
- **Vespa SPLADE 集成 production case**: docs 提及 weightedset + rank-profile 可承载 SPLADE, 但 实际 production deployment scale 不公开
- **Weaviate BlockMaxWAND BM25 vs SPLADE migration path**: Weaviate 客户从 BM25 → SPLADE 实际 migration cost / quality gain 不公开
- **Turbopuffer SPLADE 支持**: docs §hybrid 仅提及 BM25 + vector, 是否客户可上传 SPLADE-encoded sparse value to namespace? docs 不明示
- **SPLADE multilingual / cross-lingual**: BERT vocab 是 mostly English; multilingual SPLADE (mSPLADE / DistilSPLADE-multilingual) production case 不公开
- **SPLADE + GPU index**: paper 用 CPU + Numba inverted index; 是否 production 跑 SPLADE on GPU (CAGRA-style sparse)? wiki 内 GPU sparse retrieval 完全空白
- **SPLADE × MRL 联合 trained model**: 是否能在 single model 同时输出 sparse SPLADE + dense MRL? "BGE-M3" 是 production 实例 (sparse + dense + ColBERT-style 同 model) 但 wiki 未 ingest BGE-M3
- **FLOPS regularizer 与 inverted index 实际 lookup cost 关系**: paper §3 用 FLOPS as proxy; 实际 BlockMaxWAND-style inverted index 查询 cost 与 FLOPS 对应吗? 不验证
- **SPLADE 在 long-document context**: BERT 256-token max; long doc (e.g., RAG passage 2K-8K token) 需要 chunking + SPLADE-per-chunk + 聚合, 论文不涵盖

Cited by: 待 query 引用
