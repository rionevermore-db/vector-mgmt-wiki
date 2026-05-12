---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-12
phase: post
ingest-context: chroma-docs
wiki-pages-total: 77
cited-pages: [concepts/product-quantization.md, systems/chroma.md]
cited-count: 2
---

# Post-snapshot (chroma-docs): quantization-landscape

## TL;DR (delta from pgvector-docs post)

**Chroma docs 不深入 quantization detail**——Chroma 哲学是 "quantization 是 SPANN/HNSW internal 实现细节, 不向用户暴露 first-class config". 与 Turbopuffer 哲学相似 (推到 model side QAT). **关键 NEW**: Chroma 通过 `VectorIndexConfig.hnsw` / `VectorIndexConfig.spann` 暴露 advanced tuning, 但默认 pre-optimized. 用户主要交互点是**dense embedding function 选择** (auto compute) + **sparse embedding function 选择** (SPLADE / BM25 / HF).

## Answer

### 与之前 ingest 的演进

| | pgvector-docs post | **chroma-docs post (NEW)** |
|---|---|---|
| Quantization vendor 哲学 | pgvector schema-level (4 vector type) | **+ Chroma 黑盒 (HNSW/SPANN internal pre-optimized)** |
| User-facing quantization config | pgvector explicit type + Vespa cell type | Chroma 仅 expert tuning 接口 |
| Sparse vector quantization | pgvector `sparsevec` type | **Chroma SparseVectorIndexConfig + SPLADE/BM25/HF sparse functions first-class** |

### Chroma quantization 哲学（与其他 vendor 对比）

| Vendor | User config level | Quantization exposure |
|---|---|---|
| Qdrant | quantization_config (scalar/binary/1.5/2-bit/PQ/asymmetric) | very explicit |
| Weaviate | vectorIndexConfig.bq/rq/pq/sq | explicit |
| Vespa | tensor cell type (float/bfloat16/int8/single-bit) | schema-level |
| pgvector | vector type (vector/halfvec/bit/sparsevec) | **schema-level** |
| Milvus | index_type (IVF_PQ / IVF_SQ8 / SCANN) | index-config |
| Pinecone | 黑盒 slab adaptive | not exposed |
| Turbopuffer | f32/f16 cell type + QAT philosophy | 推到 embedding model side |
| **Chroma** | **advanced HNSW/SPANN tuning + sparse embedding function** | **mostly 黑盒 + advisory** |

→ Chroma 与 Pinecone + Turbopuffer 哲学一致: quantization 是 vendor 内部决策, 不暴露给用户.

### 已知盲区

- **Chroma Cloud SPANN 是否 native PQ / OPQ**: docs 不深入
- **Chroma 客户在 SPLADE function 之外的 sparse function**: 自定义 sparse model 支持不公开
- **Chroma quantization vs Turbopuffer QAT philosophy 对比**: 都推到 model side, 但 Chroma 不像 Turbopuffer 明示推荐 voyage-3 等 MRL+QAT model

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [systems/chroma.md](../../systems/chroma.md)
