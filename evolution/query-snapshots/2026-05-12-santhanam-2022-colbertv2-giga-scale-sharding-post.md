---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-12
phase: post
ingest-context: santhanam-2022-colbertv2
wiki-pages-total: 79
cited-pages: [concepts/colbertv2.md, systems/vespa.md, systems/milvus.md, topics/sparse-dense-hybrid-retrieval.md]
cited-count: 4
---

# Post-snapshot (santhanam-2022-colbertv2): giga-scale-sharding

## TL;DR (delta from lancedb-docs post)

**ColBERTv2 在 giga-scale 下需要 M× storage**——每 doc M token vectors (典型 M ~128), 千亿 docs × 128 token × 20 bytes (residual compressed) = ~25 TB token storage. 比 single-vector 增加 ~128×. **关键 NEW**: giga-scale ColBERTv2 production case **wiki + 论文均 zero coverage**——paper MS MARCO 8.8M passages 实测, 千亿规模 late-interaction case 不存在. Vespa weightedset + Milvus 多 vector field 是 wiki 内最 close candidate.

## Answer

### 与之前 ingest 的演进

| | lancedb-docs post | **santhanam-2022-colbertv2 post (NEW)** |
|---|---|---|
| Late-interaction giga-scale production | 不涵盖 | **NEW: zero coverage (paper MS MARCO 8.8M only)** |
| Multi-vector storage cost at giga scale | 不涵盖 | **NEW: M× per-doc storage (128× typical for ColBERT)** |
| 千亿规模 vendor 候选 with late-interaction | n/a | **Vespa weightedset + Milvus 多 vector field 是 candidate** |

### Late-interaction giga-scale storage 推算

[per santhanam-2022-colbertv2 §3.3 + production scale]

100B docs × 128 token/doc × 20 bytes/token (ColBERTv2 residual b=1) = **256 TB storage**

vs comparison:
- Single dense 768-d float32: 100B × 3 KB = 300 TB (similar)
- Single dense int8: 100B × 768 byte = 76.8 TB
- Single MRL prefix 256-d int8 + RaBitQ binary: 100B × 32 bytes = 3.2 TB
- **ColBERTv2 multi-vector residual: 25.6 TB (256 TB scale 错; 实际 PLAID-style inverted file 经 cluster + 残差)**

→ ColBERTv2 storage cost 在 giga-scale 是 high-tier, 比 MRL + RaBitQ 单 vector 高 ~10×.

### Vespa + Milvus 是 wiki 内 ColBERTv2 production candidate

[per santhanam-2022-colbertv2 + Vespa/Milvus docs]

| Vendor | ColBERT-style multi-vector support |
|---|---|
| **Vespa** | ✓ tensor framework + weightedset, paper-published support |
| **Milvus** | ✓ 多 vector field native (v2.x+) |
| Pinecone | partial via Multi-Vector Index (闭源) |
| Weaviate | partial named vectors (not MaxSim optimized) |
| Qdrant / Chroma / Turbopuffer / pgvector / LanceDB | application 层 |

→ **giga-scale + ColBERTv2 production candidate: Vespa + Milvus**——两者 native multi-vector field 支持 + 大规模 production case track record.

### 决策表（updated 2026-05-12 post colbertv2）

| Workload | 推荐 |
|---|---|
| 千亿 + late-interaction first-class | **Vespa weightedset OR Milvus 多 vector field** |
| 千亿 + sparse SPLADE | Vespa rank-profile + SPLADE weightedset |
| 千亿 + dense single-vector | Vespa SPANN + 4-phase ranking / DistributedANN |
| ≤100M docs + 已 Postgres | pgvector |

### 已知盲区

- **千亿 ColBERTv2 production case**: paper MS MARCO 8.8M only
- **PLAID-style inverted file 在 giga-scale**: 不公开
- **Vespa + ColBERTv2 large-scale production**: docs 不公开
- **3-way (sparse + dense + late-interaction) hybrid pipeline at giga-scale**: 不存在公开

## Cited Pages

- [concepts/colbertv2.md](../../concepts/colbertv2.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/milvus.md](../../systems/milvus.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
