---
title: BGE-M3（Multi-Functionality + Multi-Linguality + Multi-Granularity embedding）
type: concept
sources: [chen-2024-bge-m3, muennighoff-2023-mteb, formal-2021-splade-v2, santhanam-2022-colbertv2]
related: [clip.md, matryoshka-embedding.md, splade-sparse-retrieval.md, colbertv2.md, ../benchmarks/mteb-massive-text-embedding-benchmark.md, ../topics/sparse-dense-hybrid-retrieval.md, ../topics/adaptive-retrieval-shortlist-rerank.md, ../topics/multimodal-embedding-retrieval.md]
created: 2026-05-12
updated: 2026-05-12
---

# BGE-M3 (M3-Embedding)

**TL;DR**: BAAI Chen et al. 2024 ACL Findings [chen-2024-bge-m3] 提出**单 model 同时输出 sparse + dense + ColBERT-style multi-vector 三种 retrieval representation**——"M3" = **Multi-Functionality** (3 retrieval types) + **Multi-Linguality** (100+ 语言) + **Multi-Granularity** (8192 token 长文档). **对 wiki 内 vector DBs 的核心价值**: (1) **wiki 内首个 3-way unified embedding model concept**——之前 wiki sparse (SPLADE) + dense (CLIP/MRL) + late-interaction (ColBERTv2) 是三个独立 model 三类 algorithm; BGE-M3 把**三 representation 集成 single model**, 是 production hybrid retrieval pipeline 的 unified 实现; (2) **Self-knowledge distillation**——把不同 retrieval functionality 的 relevance score 集成作 teacher signal, 训练时同时优化 dense + sparse + ColBERT 三 output, 互相 distillation; (3) **Multi-linguality 100+ languages**——填 wiki 内 multilingual embedding 空白 (vs OpenAI text-emb-3 / Voyage / Cohere 也有多语言但 BGE-M3 OSS); (4) **Multi-granularity 8192 token**——支持长文档 retrieval (vs ColBERT 默认 512 token), 对 RAG over long PDF / book chapter 场景 critical; (5) **Multilingual + cross-lingual + long-doc retrieval SOTA**——MIRACL multilingual benchmark + MKQA cross-lingual + MLDR long-document benchmark 均 SOTA; (6) **Apache-2.0 OSS, BAAI 中科院开放发布**——production-ready, 与 OpenAI / Cohere / Voyage 闭源 commercial 形成对比. **production 现状**: BGE-M3 是当前 OSS embedding model 主流——Hugging Face / Ollama / 多 vector DB vendor 直接集成 (Vespa / Milvus / Qdrant / Weaviate native支持 BGE-M3 sparse + dense + colbert output). [chen-2024-bge-m3 §1-3]

## 提出背景

[per chen-2024-bge-m3 §1-2]

**Pre-BGE-M3 (single-purpose embedding) limit**:
- Single-vector dense (DPR / CLIP / BGE v1 / OpenAI / Cohere etc.): 仅 dense ANN retrieval
- Sparse (SPLADE / DeepImpact): 仅 sparse inverted index retrieval
- Late-interaction (ColBERTv2): 仅 multi-vector MaxSim retrieval
- Production 需要 hybrid (per [topics/sparse-dense-hybrid-retrieval.md](../topics/sparse-dense-hybrid-retrieval.md)) → **需要 3 个独立 model + 3 套 infrastructure**

**BGE-M3 insight**:
- Single BERT-base backbone 同时输出 3 种 representation
- Cross-distillation: 三 output 互为 teacher, 共同提升 quality
- 100+ language coverage via XLM-RoBERTa-large initialization
- Long-doc (8192 token) via RoPE position encoding extension

## 关键性质

### 1. 三 representation 同时输出（核心 Multi-Functionality）

[per chen-2024-bge-m3 §3.1]

```python
output = bge_m3_model(text)
# output 同时包含:
output.dense_embedding      # 1024-d normalized dense vector
output.sparse_lexical_weights  # vocab-level term weights (类似 SPLADE)
output.colbert_token_embeddings  # token-level multi-vector (类似 ColBERT v2)
```

**Single forward pass** 同时计算三 representation:
- Dense: [CLS] token's last hidden → linear projection → L2 normalize
- Sparse: per-token MLM logit projection (类似 SPLADE 但 simplified)
- ColBERT: per-token hidden state → linear projection (类似 ColBERTv2 但 simplified)

→ **wiki 内 first model 把 sparse + dense + late-interaction 在 single forward pass 同时输出**.

### 2. Self-knowledge distillation 训练

[per chen-2024-bge-m3 §3.3]

**Cross-distillation**:
- 三 representation 互为 teacher
- Loss: `L = L_dense + L_sparse + L_colbert + L_distill(dense ↔ sparse ↔ colbert)`
- Final retrieval score: weighted sum 三 representation contributions

→ 训练时三 representation 共享 backbone learning, 互相促进——比 3 个独立 model 训练 quality 更高 (per paper).

### 3. Multi-Linguality 100+ 语言

[per chen-2024-bge-m3 §3.4]

- Backbone: XLM-RoBERTa-large (多语言 pre-trained)
- Training data: CCnet 多语 corpus + machine translation pairs + dataset extension
- Coverage: 100+ working languages
- Cross-lingual retrieval: query in language A → docs in language B 同 retrieve

→ **wiki 内 first multilingual hybrid embedding model**——vs Voyage / Cohere 多语言但闭源, BGE-M3 OSS.

### 4. Multi-Granularity (long-document) 8192 token

[per chen-2024-bge-m3 §3.4]

- 默认 BERT-style model 512 token; BGE-M3 extends 到 8192 via RoPE position encoding extension
- Long-doc retrieval: 不需要 chunk + aggregate, 直接 encode 8K token document
- 对 RAG over 长 PDF / book / legal document workflow critical

→ vs ColBERT 默认 512 token + chunk; BGE-M3 single 8K context 更 unified.

## 性能 results

[per chen-2024-bge-m3 §4-5]

### MIRACL (multilingual retrieval)

BGE-M3 nDCG@10 outperforms baselines on **18 languages**——supporting 100+ working languages with strong cross-lingual transfer.

### MKQA (cross-lingual retrieval)

BGE-M3 cross-lingual nDCG@10 outperforms multilingual baselines (mE5 / mContriever / etc.).

### MLDR (multilingual long-document retrieval)

BGE-M3 establishes new SOTA on long-document retrieval across 13 languages.

### Hybrid retrieval performance gain

Single dense / sparse / colbert each: strong but not best
**Hybrid (dense + sparse + colbert) of M3**: SOTA in most benchmarks

→ Cross-distillation training让 hybrid quality 超过任意 single representation.

## 与 wiki 内 ingest 的关系

### BGE-M3 = SPLADE + CLIP/MRL + ColBERTv2 单 model 实现

| | SPLADE | CLIP / MRL | ColBERTv2 | **BGE-M3** |
|---|---|---|---|---|
| Sparse output | ✓ | ✗ | ✗ | **✓ unified** |
| Dense output | ✗ | ✓ | ✗ | **✓ unified** |
| Late-interaction output | ✗ | ✗ | ✓ | **✓ unified** |
| Multilingual | partial | partial (multi CLIP variants) | text-only | **✓ 100+ languages** |
| Long-doc | ✗ (256 token typical) | ✗ (model-dependent) | partial (chunk) | **✓ 8192 token** |
| Model count needed for production hybrid | 3 separate | n/a | n/a | **1 model** |
| OSS | ✓ | ✓ | ✓ | **✓ Apache-2.0** |

→ BGE-M3 是 wiki 内 first **3 retrieval algorithm unified single model**, simplify production hybrid pipeline 显著.

### BGE-M3 + vector DB vendor

Production vector DB vendor 集成 BGE-M3:
- **Vespa**: tensor framework + weightedset 直接存 dense + sparse + colbert 3 output
- **Milvus**: 多 vector field native 直接支持
- **Weaviate**: named vectors per object
- **Qdrant**: multiple vectors per point
- **Chroma**: SparseVectorIndexConfig + dense
- **LanceDB**: Lance format multi-column
- **Turbopuffer**: multi_query 内 BGE-M3 三 output 分别 query
- **pgvector**: 多 column (vector + sparsevec + multi-vector)

→ **几乎所有 wiki vendor 都支持 BGE-M3 hybrid 输出**.

### BGE-M3 + MRL / RaBitQ 量化

- BGE-M3 dense 1024-d full output, 未明示 MRL-trained (vs OpenAI emb-3 / Voyage MRL native)
- Quantization 可应用: dense float32 → int8 / RaBitQ binary
- ColBERT-style token vectors 可用 ColBERTv2 residual compression
- Sparse 已自然稀疏

## Open Questions

- **BGE-M3 vs 单独 SPLADE + CLIP + ColBERTv2 hybrid head-to-head**: 量化 cross-distillation 实际收益 不公开 ablation
- **BGE-M3 + MRL prefix-aware extension**: 当前 BGE-M3 不 MRL-trained, dim truncation 是否 work? 未实测
- **BGE-M3 production case at scale**: 公开 large-scale (>100M docs) BGE-M3 production deployment 不公开
- **BGE-M3 long-doc 8192 token retrieval cost**: 8K context inference cost vs chunk + aggregate trade-off 不公开
- **BGE-M3 vs OpenAI text-emb-3 / Cohere v3 / Voyage commercial head-to-head on production workload**: MTEB scores 可比但 production 实际选择 driver 不公开
- **后续 BGE-M3 variants** (BGE-Large / BGE-Small): wiki 未涵盖
- **3-way unified vendor adoption rate**: vendor 支持但实际 BGE-M3 production case rate 不公开

Cited by: 待 query 引用
