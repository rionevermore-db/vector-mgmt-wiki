---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-12
phase: post
ingest-context: pgvector-docs
wiki-pages-total: 76
cited-pages: [concepts/product-quantization.md, concepts/matryoshka-embedding.md, systems/pgvector.md]
cited-count: 3
---

# Post-snapshot (pgvector-docs): quantization-landscape

## TL;DR (delta from formal-2021-splade-v2 post)

**pgvector 在 quantization landscape 引入独特 SQL-native pattern**: **expression-based indexing on `binary_quantize(embedding)` + 二阶段 rerank**——用 Postgres native "index on expression" feature 实现 binary shortlist + cosine full rerank, **不需要 vendor-specific binary 配置**. **关键 NEW**: pgvector 通过 vector type 直接暴露 4 种 storage cost tier (`vector` f32 / `halfvec` f16 / `bit` 1-bit / `sparsevec` SPLADE-friendly), 是 wiki 内 vendor **vector type 选择最 explicit** 的. 与 Vespa tensor cell type 类似哲学 (schema-level cost decision), 但 pgvector 通过 SQL type system 而非 cell type.

## Answer

### 与之前 ingest 的演进

| | formal-2021-splade-v2 post | **pgvector-docs post (NEW)** |
|---|---|---|
| Quantization SQL-native pattern | 未涵盖 | **NEW: pgvector expression-based index + 二阶段 rerank** |
| Vector type 一等公民选择 | Vespa tensor cell type | **+ pgvector 4 type (`vector`/`halfvec`/`bit`/`sparsevec`)** |
| sparse vector storage | SPLADE conceptually | **+ pgvector `sparsevec` type 直接存 SPLADE 输出** |

### pgvector expression-based binary quantization（NEW SQL pattern）

[per sources/docs/pgvector/README.md §Binary Quantization]

```sql
-- 1. Index on quantized expression (Postgres native, 不需 separate column)
CREATE INDEX ON items USING hnsw 
  ((binary_quantize(embedding)::bit(768)) bit_hamming_ops);

-- 2. Two-stage query: binary shortlist + cosine rerank
SELECT * FROM (
    SELECT * FROM items 
    ORDER BY binary_quantize(embedding)::bit(768) 
             <~> binary_quantize('[1,-2,3,...]') 
    LIMIT 20
) ORDER BY embedding <=> '[1,-2,3,...]' LIMIT 5;
```

**对应 wiki 内已 ingest concept**:
- 二阶段 retrieval = [topics/adaptive-retrieval-shortlist-rerank.md](../../topics/adaptive-retrieval-shortlist-rerank.md) 的 instance
- Binary quantization = 与 Qdrant BQ / Weaviate BQ / Vespa single-bit binary 类似
- **不同点**: pgvector 利用 Postgres expression index 复用 storage, 不需要 vendor-specific binary index 配置 / 单独 binary column

### pgvector 4 vector type schema-level cost tier (NEW)

[per README §Vector Types]

| Type | Per-element | Max dim | Best for |
|---|---|---|---|
| `vector` | 4 bytes (float32) | 16,000 | dense embedding default |
| `halfvec` | 2 bytes (float16) | 16,000 | **storage 减 50%, recall 损失 minimal** |
| `bit` | 1 bit | 64,000 | binary 向量 (binary quantization 输出, 32× compress vs vector) |
| `sparsevec` | variable | 16,000 non-zero | **SPLADE / BM25 sparse output 直接存** |

**比较其他 vendor 的等价**:
- Qdrant: 通过 `quantization_config` 配置 (scalar / binary / 1.5-bit / 2-bit / asymmetric / PQ)
- Weaviate: 通过 `vectorIndexConfig.bq/rq/pq/sq` 配置
- Vespa: `tensor<float>(...)` / `tensor<bfloat16>(...)` / `tensor<int8>(...)` cell type
- Milvus: 通过 `index_type` 选 IVF_PQ / IVF_SQ8 / SCANN
- **pgvector**: 通过 SQL column type 直接选 — **schema-first decision**

→ pgvector + Vespa 是 wiki 内**最 schema-level quantization decision** 的两个 vendor.

### Quantization landscape 全景表（updated 2026-05-12 post pgvector-docs）

| 方法 | Axis | Production usage |
|---|---|---|
| Float32 (baseline) | n/a | 全部 + **pgvector `vector` type** |
| bfloat16 | post-hoc (dense) | Vespa cell type |
| **Float16 / halfvec** | post-hoc (dense) | Weaviate / Turbopuffer / **pgvector `halfvec` type** |
| Scalar Quantization (int8) | post-hoc (dense) | Qdrant / Weaviate / Vespa |
| Binary (1-bit) | post-hoc (dense) | Qdrant BQ / Weaviate BQ / Vespa single-bit / **pgvector `bit` type + expression index** |
| PQ / OPQ | post-hoc (dense) | Faiss / Milvus / Pinecone |
| RaBitQ | post-hoc (dense) | Faiss / Milvus future |
| MRL prefix truncation | training-time (dense) | OpenAI / Voyage / Cohere / Vespa matryoshka cell |
| **SPLADE FLOPS sparse** | training-time (sparse) | Vespa weightedset / **pgvector `sparsevec` type** |

→ **pgvector 是 wiki 内 vendor 中, 通过 SQL column type 同时支持 sparse + dense + binary + halfvec 4 种 vector storage 的唯一 implementation**.

### Production state-of-the-art combinations with pgvector

[per pgvector + Phase 1-2-3]

```sql
-- Production stack: MRL-trained voyage-3 + halfvec + binary + sparse hybrid
CREATE TABLE products (
  id bigserial PRIMARY KEY,
  description text,
  -- Dense: MRL prefix 512-d in halfvec (2-byte/elem = 1 KB/vec)
  dense_emb halfvec(512),
  -- Binary quantization expression index for fast shortlist
  -- Sparse: SPLADE-encoded for hybrid retrieval
  sparse_emb sparsevec(30522)
);

-- Index: HNSW on halfvec + expression index on binary + FTS on description
CREATE INDEX ON products USING hnsw (dense_emb halfvec_cosine_ops);
CREATE INDEX ON products USING hnsw ((binary_quantize(dense_emb)::bit(512)) bit_hamming_ops);
```

### 已知盲区

- **pgvector × MRL prefix-aware**: 当前 pgvector 不感知 MRL nested structure; query-time prefix truncate via SQL substring 可行但 native indexing 不存在
- **`sparsevec` 是否能用 HNSW index**: 当前 sparsevec inverted index only? 文档不明确
- **expression-based binary quantization performance vs vendor-specific BQ**: 实测 head-to-head 不公开
- **pgvector + RaBitQ**: RaBitQ 论文是 SIGMOD 2024, pgvector 当前不支持; 是否 future extension?

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
- [systems/pgvector.md](../../systems/pgvector.md)
