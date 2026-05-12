---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-12
phase: post
ingest-context: santhanam-2022-colbertv2
wiki-pages-total: 79
cited-pages: [concepts/hnsw.md, concepts/colbertv2.md, concepts/splade-sparse-retrieval.md]
cited-count: 3
---

# Post-snapshot (santhanam-2022-colbertv2): hnsw-vs-nsg-selection

## TL;DR (delta from lancedb-docs post)

**ColBERTv2 引入 wiki 内 multi-vector retrieval ANN 选择全新维度**——单 doc 是 M 个 token vectors (M ~128 typical), HNSW/NSG 选择不仅作 single-vector ANN, 还需考虑 **token-level inverted file (PLAID-style) vs per-token graph index** 二选一. ColBERTv2 paper 用 IVFPQ-style index (centroid + residual), 不是 graph. wiki 内 vendor 支持 ColBERT-style 多 vector typically 用 inverted file: Vespa weightedset / Milvus 多 vector field 共享 HNSW. NSG production case 仍零.

## Answer

### 与之前 ingest 的演进

| | lancedb-docs post | **santhanam-2022-colbertv2 post (NEW)** |
|---|---|---|
| HNSW production OSS DBMS | 7 | **不变 (ColBERTv2 是 algorithm 非 system)** |
| Multi-vector ANN index choice | application 层 | **NEW: HNSW vs PLAID-style token inverted file** |
| NSG production | Taobao 2B + NSSG | **不变** |

### ColBERTv2 PLAID-style inverted file vs HNSW

[per santhanam-2022-colbertv2 §3.3 + PLAID followup]

ColBERTv2 paper 用 **IVFPQ-style index**: cluster token vectors into centroids → centroid-based inverted file → MaxSim search.

vs HNSW for multi-vector:
- HNSW per-token: 每 doc M token vectors, 各自 in HNSW → query 时 N × M MaxSim
- Inverted file: 仅 cluster centroids, query token-level lookup posting → 更高效大规模

→ ColBERTv2 选 inverted file approach, **不是 HNSW**——这是 multi-vector ANN 的 unique aspect.

### 选择决策（updated 2026-05-12 post santhanam-2022-colbertv2）

- **Single-vector dense retrieval** → HNSW (7 OSS vendor)
- **Sparse retrieval** → inverted index (BM25/SPLADE BlockMaxWAND)
- **Late-interaction multi-vector retrieval (ColBERTv2)** → token-level inverted file (PLAID-style)
- 学术 NSG → 仍零 production

### 已知盲区

- **ColBERTv2 token-level HNSW vs inverted file 实测对比**: paper 仅 inverted file approach
- **HNSW 在 multi-vector setting 的退化曲线**: 不公开
- **Vespa weightedset 实现 MaxSim 与 paper PLAID 差异**: docs 不深入

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/colbertv2.md](../../concepts/colbertv2.md)
- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
