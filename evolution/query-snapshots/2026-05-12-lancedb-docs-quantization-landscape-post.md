---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-12
phase: post
ingest-context: lancedb-docs
wiki-pages-total: 78
cited-pages: [concepts/product-quantization.md, systems/lancedb.md]
cited-count: 2
---

# Post-snapshot (lancedb-docs): quantization-landscape

## TL;DR (delta from chroma-docs post)

**LanceDB 通过 Lance columnar format 提供 wiki 内独特 quantization storage layer**——Lance format 是 OSS standard columnar format (类似 Parquet/Arrow), 与 vector index storage 集成. 与 pgvector "vector type schema-level decision" 哲学相近 — type 直接决定 storage cost, 但 LanceDB Lance format 更**ecosystem-friendly** (可被 DuckDB / Pandas / Polars 等直接 read, 不需要 vendor-specific protocol).

## Answer

### 与之前 ingest 的演进

| | chroma-docs post | **lancedb-docs post (NEW)** |
|---|---|---|
| Storage format quantization awareness | Chroma 黑盒 | **+ LanceDB Lance OSS format 集成 vector index storage** |
| Quantization 与 ML ecosystem 兼容性 | per-vendor binary format | **LanceDB Lance format DuckDB / Pandas / Polars 可直接 query** |

### LanceDB Lance format storage layer 哲学

[per sources/docs/lancedb/]

**Lance format key quantization-relevant properties**:
- Columnar storage (类似 Parquet)
- Vector index metadata in same format
- Random access optimization (fast slicing for ML batch)
- Schema evolution + version control

**Implication for quantization**:
- Quantized vector 作为 Lance format column type (similar to Parquet quantization)
- Index metadata (HNSW/IVF graph) 在 Lance format 中 native
- 与 ML pipeline (DuckDB / Pandas / Polars) 共享 data layer

### Quantization landscape 全景（updated 2026-05-12 post lancedb-docs）

| 方法 | 工业 production usage |
|---|---|
| Float32 baseline | 全部 (+ pgvector `vector` + LanceDB Lance dense column) |
| FP16 / halfvec | Weaviate / Turbopuffer / pgvector `halfvec` |
| Scalar Quantization int8 | Qdrant default + Weaviate SQ + Vespa cell |
| Binary 1-bit | Qdrant BQ + Weaviate BQ + Vespa single-bit + pgvector `bit` |
| PQ / OPQ | Faiss + Milvus + Pinecone |
| RaBitQ | Faiss + Milvus future |
| MRL prefix | OpenAI / Voyage / Cohere |
| SPLADE FLOPS sparse | Vespa + Chroma SparseVectorIndexConfig + pgvector `sparsevec` |
| **Lance format columnar quantization** | **LanceDB OSS standard** |

→ LanceDB Lance format 是 wiki 内**首个 quantization-relevant storage 哲学是 OSS standard columnar format 的 vendor** (其他都 proprietary format).

### 已知盲区

- **Lance format 具体 quantization encoding**: docs 不深入 (e.g., 是否支持 PQ/RaBitQ 作 column type)
- **LanceDB quantization vs Milvus IVF_PQ 实测对比**: 不公开
- **Lance format 兼容性 vs Parquet/Arrow + Hudi/Iceberg/Delta**: 不公开

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [systems/lancedb.md](../../systems/lancedb.md)
