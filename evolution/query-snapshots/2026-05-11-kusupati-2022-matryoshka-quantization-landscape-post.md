---
query-key: quantization-landscape
query: "PQ、OPQ、Scalar Quantization、RaBitQ 这几种向量压缩方法各自的精度-速度-内存权衡是什么？工业系统通常如何组合使用（例如和 IVF / HNSW 配合）？"
date: 2026-05-11
phase: post
ingest-context: kusupati-2022-matryoshka
wiki-pages-total: 73
cited-pages: [concepts/product-quantization.md, concepts/rabitq.md, concepts/matryoshka-embedding.md, topics/adaptive-retrieval-shortlist-rerank.md, systems/turbopuffer.md, systems/vespa.md]
cited-count: 6
---

# Post-snapshot (kusupati-2022-matryoshka): quantization-landscape

## TL;DR (delta from radford-2021-clip post)

**MRL ingest 给 wiki quantization landscape 引入全新 axis: training-time dim reduction**——之前 wiki 内所有 quantization (PQ / OPQ / SQ / BQ / RaBitQ) 都是 **post-hoc lossy compression**, MRL 是 **training-time first-principle approach**: 训练时显式优化 prefix transferability, inference 0-cost truncate. **关键 NEW**: production state-of-the-art = **MRL + post-hoc quantization 双轨叠加**: voyage-3 (MRL-trained, 1024-d) → prefix 256-d → RaBitQ binary → 单 vec 32 bytes (vs 4096 bytes float32 = 128× compress). Turbopuffer "QAT model int8 → f16 namespace" 哲学**算法基础就是 MRL + QAT 双 train-time technique**. CLIP variant 矩阵 (RN50/ViT-B/L 多 dim 各自训练) 被 **MRL 单 model 多 prefix 取代**.

## Answer

### 与之前 ingest 的演进

| | radford-2021-clip post | **kusupati-2022-matryoshka post (NEW)** |
|---|---|---|
| Quantization landscape (PQ/OPQ/SQ/BQ/RaBitQ) | post-hoc lossy compression | **+ MRL: training-time dim reduction (lossless prefix)** |
| Quantization 维度 | 单 axis (post-hoc) | **正交 2 axis: training-time (MRL) + post-hoc (PQ/RaBitQ/...)** |
| Production state-of-the-art | per-quantizer | **MRL + quantizer 双轨叠加** |

### 训练时压缩 vs 后处理压缩 二分（NEW axis）

[per kusupati-2022-matryoshka §2 + wiki quantization landscape]

| 维度 | Training-time technique | Post-hoc technique |
|---|---|---|
| 介入阶段 | 训练 backbone 时 | 训练完后单独压缩 |
| 信息保留 | **Lossless prefix** (MRL) | Lossy (PQ/OPQ/SQ/BQ/RaBitQ) |
| 查询 overhead | 0 (取 prefix) | LUT lookup / bitwise / scalar comparison |
| 需要 retrain? | yes | no |
| Wiki 内 instances | **MRL (本 ingest), QAT (voyage-4 int8 output)** | PQ / OPQ / SQ / BQ / RaBitQ |
| 可叠加? | yes (双轨独立 axis) | yes |

### Quantization landscape 全景表（updated 2026-05-11 post kusupati-2022-matryoshka）

| 方法 | Axis | 压缩率 | Production usage |
|---|---|---|---|
| Float32 (baseline) | n/a | 1× | 全部 |
| bfloat16 | post-hoc | 2× | Vespa cell type |
| FP16 | post-hoc | 2× | Weaviate / Turbopuffer namespace |
| Scalar Quantization (int8) | post-hoc | 4× | Qdrant default / Weaviate SQ |
| RQ8 (rotation + scalar) | post-hoc | 4× | Weaviate default |
| Binary (1-bit) | post-hoc | 32× | Qdrant BQ / Weaviate BQ / Vespa single-bit |
| PQ (8x8 = 64-bit) | post-hoc | 24-48× | Faiss / Milvus / Pinecone |
| OPQ | post-hoc (learn rotation) | 同 PQ | Faiss / Milvus IVF_OPQ |
| RaBitQ | post-hoc | 32× (1-bit) | Faiss / Milvus future |
| **MRL prefix truncation** | **training-time** | **prefix/d ratio (e.g., 256/1024 = 4×)** | **OpenAI emb-3, Voyage-3, Cohere v4, Qwen3-VL, Vespa matryoshka cell** |
| **QAT model int8 output** | **training-time** | **4× (f32→int8)** | **voyage-3 / embed-v4 / Qwen3-VL (Turbopuffer recommended)** |

### Production state-of-the-art 双轨压缩组合

[per kusupati-2022-matryoshka + wiki + Turbopuffer docs]

**Most aggressive compression stack**:
- Embedding model: voyage-3 (MRL + QAT trained, 1024-d default, int8 output equivalent f32)
- Training-time compression: **MRL** prefix → 取 256-d
- Training-time compression: **QAT** int8 → 不损 precision
- Post-hoc compression: RaBitQ binary 1-bit per dim
- Vector DB: store 32 bytes per vector (256 dims × 1 bit = 256 bits = 32 bytes)
- vs Float32 4096 bytes/vec → **128× compress, 全部 lossless or sharp-bound**

### 工业组合策略（updated 2026-05-11 post kusupati-2022-matryoshka）

- **Turbopuffer + voyage-3 (MRL + QAT)**: model side 全 train-time, DBMS side 仅 cosine ANN——**架构最 clean**
- **Vespa matryoshka cell + binary**: 单 schema 多 prefix + DBMS side binary quantization
- **Pinecone slab + adaptive**: 黑盒 (推测 MRL-aware?)
- **Milvus IVF_OPQ on MRL-trained embedding**: 应用层 prefix + DBMS post-hoc PQ
- **Faiss + RaBitQ research path**: 学术 experimentation

### 已知盲区

- **MRL prefix × PQ subspace 对齐**: PQ subspace 边界与 MRL granularity 边界 (e.g., subspace_i = prefix_{16(i-1):16i}) 一致性 wiki + 论文都 zero coverage
- **RaBitQ on MRL prefix**: P 矩阵是否需要 per-prefix retrain? unbiased property 在 prefix 上保留? OPEN (详见 [concepts/rabitq.md Open Questions](../../concepts/rabitq.md))
- **OPQ on MRL embedding**: learn rotation 在 nested structure 上是否帮助? Unknown
- **Pinecone slab 是否 MRL-aware**: 黑盒, 不公开

## Cited Pages

- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [concepts/rabitq.md](../../concepts/rabitq.md)
- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
- [topics/adaptive-retrieval-shortlist-rerank.md](../../topics/adaptive-retrieval-shortlist-rerank.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/vespa.md](../../systems/vespa.md)
