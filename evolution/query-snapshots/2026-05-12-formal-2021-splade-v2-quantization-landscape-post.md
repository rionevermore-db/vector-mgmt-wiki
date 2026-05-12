---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-12
phase: post
ingest-context: formal-2021-splade-v2
wiki-pages-total: 75
cited-pages: [concepts/product-quantization.md, concepts/splade-sparse-retrieval.md, topics/sparse-dense-hybrid-retrieval.md]
cited-count: 3
---

# Post-snapshot (formal-2021-splade-v2): quantization-landscape

## TL;DR (delta from kusupati-2022-matryoshka post)

**SPLADE 引入 wiki 内 quantization landscape 全新 axis: sparse vector 压缩**——之前 wiki quantization (PQ/OPQ/SQ/BQ/RaBitQ/MRL) 全部 dense vector compression. SPLADE output 本身**已是稀疏向量 (~30K dim, mostly 0)** → 不需要 dense quantization, 但需要**inverted index storage compression** (posting list 编码 / impact value 量化). **关键 NEW**: production sparse vector storage 用**term impact quantization** (DeepImpact-style) — 把 SPLADE float term weight 量化到 int8 / int4 / int2 — wiki 内未涵盖. FLOPS regularizer 直接优化 expected query-document FLOPS, 与 dense ANN 的 FLOP-recall trade-off **形式相同但不同压缩 axis**.

## Answer

### 与之前 ingest 的演进

| | kusupati-2022-matryoshka post | **formal-2021-splade-v2 post (NEW)** |
|---|---|---|
| Quantization axis | post-hoc (PQ/RaBitQ/SQ/BQ) + training-time (MRL prefix + QAT) | **+ sparse-side compression (SPLADE + impact quantization)** |
| Dense vs sparse storage | dense ANN index | **+ sparse inverted index posting list compression** |
| FLOPS optimization | dense ANN cost reduction (MRL prefix) | **+ sparse lookup cost reduction (FLOPS regularizer)** |

### SPLADE = 训练时优化的 sparse 表征（结构本身是 compression）

[per formal-2021-splade-v2 §3.1 + FLOPS regularizer]

**FLOPS regularizer**:
```
ℓ_FLOPS = Σ_{j∈V} ā_j^2 = Σ_{j∈V} (1/N · Σ_{i=1}^N w_j^(d_i))^2
```

- 直接惩罚 term-level activation 的平方均值
- **Effect**: SPLADE 产生**balanced sparse vectors** (no extremely heavy posting list)
- 这是与 ℓ_1 regularization 不同的 axis——ℓ_1 仅 minimize 总非零数; FLOPS minimize expected lookup cost

**与 dense quantization 的对比**:
- Dense quantization (PQ/RaBitQ): post-hoc lossy compression
- SPLADE: **训练时 explicitly enforce 稀疏 + balanced** — 不是 lossy compress, 是 **算法层 sparse representation learning**

### Sparse vector storage compression 全 axis

[per formal-2021-splade-v2 + production reality + DeepImpact / COIL]

| 方法 | Compression axis | wiki 内 ingest |
|---|---|---|
| **SPLADE** | training-time sparsity + FLOPS regularization | **本 ingest** |
| DeepImpact | doc2query + impact quantization (int8 term weights) | paper-cited, not ingested |
| COIL-tok | contextualized inverted (per-term dense vector) | paper-cited, not ingested |
| **Inverted index posting list compression** (VBE / Frame-of-Reference / SIMD-BP128) | **post-hoc encoding of doc IDs + impacts** | wiki 未涵盖 |
| **Term impact quantization** (int8 / int4 weights in inverted index) | post-hoc | wiki 未涵盖 |

→ **Sparse-side compression** axis 在 wiki 内仍是 large coverage gap. SPLADE 是开端 (training-time), 但 post-hoc sparse compression (inverted index encoding) wiki 仍 zero coverage.

### Quantization landscape 全景表（updated 2026-05-12 post formal-2021-splade-v2）

| 方法 | Axis | Dense or Sparse | Production usage |
|---|---|---|---|
| Float32 (dense baseline) | n/a | dense | 全部 |
| bfloat16 | post-hoc (dense) | dense | Vespa cell type |
| FP16 | post-hoc (dense) | dense | Weaviate / Turbopuffer |
| Scalar Quantization (int8) | post-hoc (dense) | dense | Qdrant default + Weaviate SQ |
| Binary (1-bit) | post-hoc (dense) | dense | Qdrant BQ / Weaviate BQ / Vespa |
| PQ / OPQ | post-hoc (dense) | dense | Faiss / Milvus / Pinecone |
| RaBitQ | post-hoc (dense) | dense | Faiss / Milvus future |
| **MRL prefix truncation** | training-time (dense) | dense | OpenAI emb-3 / Voyage / Cohere |
| QAT int8 output | training-time (dense) | dense | voyage-multimodal-3 / Qwen3-VL |
| **SPLADE FLOPS-regularized sparse** | **training-time (sparse)** | **sparse** | **本 ingest** |
| **Term impact quantization (int8 weight)** | **post-hoc (sparse)** | **sparse** | wiki 未涵盖 |
| **Inverted index posting list compression** | **post-hoc (sparse)** | **sparse** | wiki 未涵盖 |

### Production hybrid compression stack

[per formal-2021-splade-v2 + Phase 1-2]

**Most aggressive production stack**:
- Dense path: voyage-3 (MRL + QAT trained, 1024-d) → prefix 256-d → RaBitQ binary 32 bytes/vec
- Sparse path: SPLADE FLOPS-regularized → ~50 non-zero terms per doc → impact int8 → ~50 bytes/doc
- Storage: 32 + 50 = ~80 bytes/vec for full hybrid retrieval pipeline
- vs Float32 dense raw 4096 bytes/vec → **50× compression overall**

### 已知盲区

- **Sparse vector post-hoc compression**: inverted index posting list encoding (VBE / Roaring bitmaps / SIMD-BP128) wiki 内 zero coverage——production-relevant
- **SPLADE 的 term impact post-hoc quantization**: int8 / int4 / int2 impact value 实测 quality 退化曲线 不公开
- **SPLADE inverted index 与 dense ANN index 联合存储**: 5 vendor 内部如何 layout? docs 不深入
- **GPU sparse storage**: 全 wiki sparse path 假设 CPU + inverted index; GPU sparse retrieval (SPLADE on GPU) 不存在公开 case

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
