---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-12
phase: post
ingest-context: santhanam-2022-colbertv2
wiki-pages-total: 79
cited-pages: [concepts/colbertv2.md, concepts/product-quantization.md, concepts/rabitq.md, concepts/matryoshka-embedding.md]
cited-count: 4
---

# Post-snapshot (santhanam-2022-colbertv2): quantization-landscape

## TL;DR (delta from lancedb-docs post)

**ColBERTv2 residual compression 是 wiki 内 multi-vector quantization 首次系统化**——centroid (4 bytes, encode 2^32 centroids) + 1-2 bit residual (16-32 bytes) = 20-36 bytes/vector total, vs v1 256 bytes float16 = **6-10× space reduction**. **关键 NEW**: 这是 wiki 内**first multi-vector quantization scheme** (vs PQ/RaBitQ/MRL 都是 single-vector 压缩). ColBERTv2 paper §3.3 明示"a natural extension of product quantization to multi-vector".

## Answer

### 与之前 ingest 的演进

| | lancedb-docs post | **santhanam-2022-colbertv2 post (NEW)** |
|---|---|---|
| Quantization axis 单/多 vector | single-vector only | **+ multi-vector (ColBERTv2 residual)** |
| Quantization landscape 完整性 | dense single + sparse + MRL prefix | **+ multi-vector centroid + residual (token-level)** |

### Quantization landscape 全景表（updated 2026-05-12 post colbertv2）

| 方法 | Axis | 单/多 vector | 工业 production usage |
|---|---|---|---|
| Float32 baseline | n/a | single | 全部 |
| bfloat16 / FP16 | post-hoc | single | Vespa cell type + pgvector halfvec |
| Scalar Quantization int8 | post-hoc | single | Qdrant default + Weaviate / Vespa |
| Binary 1-bit | post-hoc | single | Qdrant BQ + Weaviate BQ + Vespa + pgvector `bit` |
| PQ / OPQ | post-hoc | single | Faiss + Milvus + Pinecone |
| RaBitQ | post-hoc | single | Faiss + Milvus future |
| MRL prefix truncation | training-time | single | OpenAI / Voyage / Cohere |
| SPLADE FLOPS sparse | training-time | sparse | Vespa + Chroma + pgvector |
| **ColBERTv2 residual compression (NEW)** | **post-hoc** | **multi-vector token-level** | **Stanford ColBERTv2 / BGE-M3 / Vespa weightedset / Milvus multi-vector** |

→ wiki 内 quantization landscape **首次包含 multi-vector axis**——ColBERTv2 是 multi-vector quantization production-grade 实例.

### ColBERTv2 residual compression 与 PQ 关系

[per santhanam-2022-colbertv2 §3.3]

> "This centroid-based encoding can be considered a natural extension of product quantization to multi-vector..."

- PQ (single-vector): vector 切 subspace, each subspace 独立 quantize
- ColBERTv2 residual (multi-vector): per-token vector → centroid + residual
- **不同点**: PQ 在 vector 内部切 subspace; ColBERTv2 在 multi-vector set 上 cluster centroids

### Production state-of-the-art multi-axis compression

- Dense single-vector: voyage-3 (MRL + QAT trained) → prefix 256-d → RaBitQ binary = **32 bytes**
- Sparse: SPLADE-encoded → ~50 non-zero × 8 bytes = **400 bytes per doc**
- Multi-vector (late-interaction): ColBERTv2 → 128 tokens × 20 bytes = **2.56 KB per doc**

→ 三 axis quantization 各自 trade-off 不同, production hybrid pipeline 同时使用 3 representation.

### 已知盲区

- **ColBERTv2 + RaBitQ 联合**: 是否 binary residual 可叠加 RaBitQ unbiased? Open
- **PLAID-style inverted file 内部 PQ residual 性能**: 实际 production scale benchmark 不公开
- **ColBERTv2 residual vs PQ subspace 在 multi-vector context 头对头**: 不公开

## Cited Pages

- [concepts/colbertv2.md](../../concepts/colbertv2.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
