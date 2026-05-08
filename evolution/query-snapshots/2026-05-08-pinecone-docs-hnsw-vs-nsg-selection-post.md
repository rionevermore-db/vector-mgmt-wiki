---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是 proximity graph，工程上怎么选？"
date: 2026-05-08
phase: post
ingest-context: pinecone-docs
wiki-pages-total: 36
cited-pages: [concepts/hnsw.md, concepts/nsg.md, systems/pinecone.md, concepts/pinecone-serverless-slabs.md, topics/index-selection.md]
cited-count: 5
---

# Post-snapshot (pinecone-docs): hnsw-vs-nsg-selection

## TL;DR (delta from xu-2023-spfresh post)

**Pinecone 哲学：用户根本不选 index_type**。[per systems/pinecone.md "Index type 不暴露用户"] Serverless 索引的具体算法 + 参数完全 Pinecone 内部决定。这与 Faiss / Milvus / DiskANN / SPFresh "用户选 HNSW vs NSG vs ..." 哲学**根本对立**。

## Answer

### Pinecone vs 其他系统的 index_type 哲学（NEW）

[per topics/index-selection.md "Pinecone 哲学：index_type 不暴露给用户"]

| | 用户选 index | 用户选 quantizer | 用户调参 |
|---|---|---|---|
| Faiss | ✓（factory string） | ✓ | ✓（M, efConstruction, ...） |
| Milvus | ✓（IVF_FLAT/HNSW/DISKANN/...） | ✓ | ✓ |
| DiskANN | n/a（only Vamana） | n/a（PQ default） | 部分 |
| SPFresh | n/a（only SPANN+LIRE） | n/a（全精度） | 部分 |
| **Pinecone Serverless** | **✗** | **✗** | **✗（仅 dense/sparse/document field 类型）** |

→ 在 Pinecone 上"HNSW 还是 NSG" 的问题**没有意义**——用户无法选。

### Pinecone Adaptive Indexing 对此问题的颠覆（NEW）

[per concepts/pinecone-serverless-slabs.md "Layer 2：Slab"]

Pinecone slab 架构内部可能**同时用多种算法**：
- 小 slab → fast indexing（推断：HNSW 小 ef，或 IVF 小 nprobe）
- 大 slab merge 后 → sophisticated indexing（推断：HNSW 大 ef，或 SPANN-style，或 DiskANN-style）

**意味着同一 namespace 内查询会**简单地** fan-out 到不同算法 index 的 slab**——结果合并由 query router 透明处理。

→ "HNSW vs NSG" 这种二选一在 adaptive 模式下变成"何时切换 + 怎么切换"问题。**wiki 内首次见这种 hybrid + lifecycle-aware 思路**。

### 对私有部署的启示

如果用户必须自托管选 HNSW 或 NSG：参考之前 [hnsw-vs-nsg-selection xu-2023-spfresh post] 决策树（数据量 + 增量需求 + 内存预算）。但 Pinecone 启发：**可在 single namespace 同时 host 多种 index** 让"小段 fast / 大段 sophisticated" 平衡 ingest cost 与 query quality。

### 与之前几轮 ingest 的差异

| | 各 ingest post | **pinecone-docs post (NEW)** |
|---|---|---|
| HNSW 选型问题 | 在 self-host 系统里讨论 | **Pinecone 让此问题在 SaaS 不存在** |
| Adaptive index | 未出现 | **首次出现**（slab lifecycle） |
| index_type 哲学维度 | 隐含（用户都能选） | **显式：用户能选 vs 不能选**对立 |

### Open / 未覆盖

- **Pinecone slab 内部实际算法**：完全不公开
- **Adaptive transition 触发条件**：何时小 slab merge 升级到大 slab 算法，docs 不公开
- **Pinecone 的 index_type 黑盒在 high-recall 区是否优于 Milvus 用户调优 HNSW**：实测对比 wiki 仍 zero coverage

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [systems/pinecone.md](../../systems/pinecone.md)
- [concepts/pinecone-serverless-slabs.md](../../concepts/pinecone-serverless-slabs.md)
- [topics/index-selection.md](../../topics/index-selection.md)
