---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-12
phase: post
ingest-context: santhanam-2022-colbertv2
wiki-pages-total: 79
cited-pages: [concepts/colbertv2.md, systems/vespa.md, systems/milvus.md]
cited-count: 3
---

# Post-snapshot (santhanam-2022-colbertv2): index-architecture-global-vs-routed

## TL;DR (delta from lancedb-docs post)

**ColBERTv2 architecture 是 (c) cluster routing 在 multi-vector context 的特化**——token-level centroids (IVF-style) + per-centroid posting list (PLAID-style). 不引入新 architecture variant, 但**多 vector setting 内 routing 是 token-level 而非 doc-level**——这是 wiki 内 routing granularity 的全新细分.

## Answer

### ColBERTv2 routing 是 (c) hierarchical 的 multi-vector 特化

[per santhanam-2022-colbertv2 §3.3 PLAID]

PLAID engine:
1. Token-level centroid clustering (类似 IVF)
2. Per-centroid token posting list
3. Query token 找最近 centroids → posting list lookup → MaxSim aggregation

→ **(c) cluster routing axis 内 routing granularity 二分**:
- Doc-level routing (SPANN @ Bing / Vespa / Milvus segment / Pinecone slab): doc → cluster → posting
- **Token-level routing (ColBERTv2 PLAID, NEW)**: token → cluster → posting

### 8 architecture variant + routing granularity sub-axis

(a) Global single | (b-c) Cluster + partition | (d) No-index | (e) Namespace-primitive | (f) Reuse RDBMS | (g) Versioned fork | (h) Format-first

Sub-axis within (c): **doc-level routing vs token-level routing (ColBERTv2 unique)**

### 已知盲区

- **ColBERTv2 PLAID + (a) global single graph 联合**: 是否 single multi-vector graph 跨 thousand machines? Open
- **ColBERTv2 + DistributedANN-style architecture**: 不存在 production case
- **Vespa weightedset 实际 routing granularity**: doc-level vs token-level docs 不深入

## Cited Pages

- [concepts/colbertv2.md](../../concepts/colbertv2.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/milvus.md](../../systems/milvus.md)
