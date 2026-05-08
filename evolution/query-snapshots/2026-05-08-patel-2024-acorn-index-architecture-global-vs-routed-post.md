---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下索引架构选 (a) 全局单一索引 还是 (c) 层次路由结构？"
date: 2026-05-08
phase: post
ingest-context: patel-2024-acorn
wiki-pages-total: 48
cited-pages: [queries/index-architecture-global-vs-routed.md, concepts/acorn.md, concepts/filtered-vamana.md, topics/attribute-filtering.md]
cited-count: 4
---

# Post-snapshot (patel-2024-acorn): index-architecture-global-vs-routed

## TL;DR (delta from gollapudi-2023-filtered-diskann post)

**ACORN 是 (a) 全局单一索引的延伸**：不是 (c) 路由——ACORN 把 HNSW dense 化让 search-time predicate-induced subgraph 自然成为 routing。这是 wiki 已有 (c) 形态外**新的"single dense graph + dynamic subgraph routing"模式**——routing 决策完全在 query-time，不依赖 index-time partition 决策。**与 [FilteredVamana](../../concepts/filtered-vamana.md) (filter-aware single graph) 同模式**——但 ACORN 的 predicate-agnostic 让 subgraph routing 无需 known predicate set。

## Answer

### ACORN 在 (a) vs (c) 维度的位置（NEW）

[per concepts/acorn.md "Predicate Subgraph Traversal"]

ACORN 既不是经典 (a) 也不是经典 (c)：
- 不是 (a)（单一 HNSW）：ACORN 是 dense HNSW + predicate-aware traversal
- 不是 (c)（per-predicate index 或 per-shard index）：ACORN 是单一 graph，subgraph 在 search 时 induce

→ **第 8 种 (c) 路由形态：dynamic subgraph routing**：

```
ACORN structure:
   single dense HNSW graph
      ↓ search-time
   predicate p_q 决定 G(X_p)
      ↓ greedy traversal in G(X_p)
   返回 top-k

→ "subgraph routing" 在 search time 完成（not index time）
```

### 完整 (c) 形态（updated with ACORN）

| (c) 形态 | 起点 | 路由维度 | 是否 known predicate set |
|---|---|---|---|
| Library + manual sharding | Faiss | hash / range | n/a |
| 算法系统 routing-as-mutable | SPFresh LIRE | NPA centroids | n/a |
| Vector-first DBMS | Milvus / Pinecone | shard / namespace | n/a |
| OLAP-extended | ADBV cluster-based | vector geometry partition | n/a |
| OLTP-extended app-layer | PASE | hash by app | n/a |
| Filter-aware single graph | FilteredVamana | label intersection in graph traversal | **✓ known**（≤1000 equality） |
| Per-filter subgraph + stitching | StitchedVamana | per-filter Vamana + global merge | **✓ known**（partial） |
| **Predicate-induced dynamic subgraph**（NEW）| **ACORN** | **predicate-induced subgraph in single dense graph** | **✗ predicate-agnostic** |

→ ACORN 引入"**predicate-agnostic dynamic subgraph routing**"——不需要 index-time know predicate set，subgraph 在 search 时 induce。

### ACORN 与 FilteredVamana 的 (c) routing 哲学差异（NEW）

[per concepts/acorn.md "与 FilteredVamana 的对比"]

| | FilteredVamana | ACORN |
|---|---|---|
| Routing 决策时机 | **Index-time** + search-time | **Search-time only** |
| 假设 known predicate set | ✓（≤1000 equality） | ✗（predicate-agnostic） |
| Graph 结构 | filter-aware（label baked-in） | predicate-blind（dense HNSW + truncated edges） |
| Subgraph induce 方法 | 基于 stored label intersection at edge | 基于 query-time predicate filter at traversal |
| Cardinality 上限 | ≤1000 unique filters | unbounded |

→ FilteredVamana：**"baked-in routing"**——filter 信息固化在 edge selection；ACORN：**"emergent routing"**——subgraph 由 query 动态 induce。

### 千亿规模 (c) 路由 + filter 维度（推断）

理论组合（实证 zero）：
- 16 节点 ACORN + 应用层 routing → 每节点 ~390M ACORN 实例
- 跨节点 filter routing 无需 ACORN 自身改动（每节点独立做 predicate-induced subgraph）
- Cross-node merge 应用层处理

→ 千亿规模 HCPS workload 理论可行用 ACORN + 应用层 routing；实证空白。

### 与之前 ingest 的演进

| | gollapudi-2023 post | **patel-2024-acorn post (NEW)** |
|---|---|---|
| (c) 形态数 | 7（含 Filter-aware + StitchedVamana） | **8**（+ predicate-induced dynamic subgraph） |
| Routing 决策时机 | mostly index-time | **+ search-time only（ACORN）** |
| Predicate-agnostic 路由 | 未出现 | **首次出现** |
| Filter cardinality 上限 | 1000 | **>10^11**（ACORN） |

### 已知盲区

- **跨节点 ACORN routing**：分布式 ACORN 完全空白
- **ACORN + (c) 系统层 routing**：理论组合（如 Milvus segment + ACORN segment-internal）未尝试
- **ACORN scale 到千亿**：仅 25M LAION 实证

## Cited Pages

- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
- [concepts/acorn.md](../../concepts/acorn.md)
- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
