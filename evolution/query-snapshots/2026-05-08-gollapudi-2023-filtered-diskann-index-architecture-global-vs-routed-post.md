---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下索引架构选 (a) 全局单一索引 还是 (c) 层次路由结构？"
date: 2026-05-08
phase: post
ingest-context: gollapudi-2023-filtered-diskann
wiki-pages-total: 46
cited-pages: [queries/index-architecture-global-vs-routed.md, concepts/filtered-vamana.md, systems/diskann.md, topics/attribute-filtering.md]
cited-count: 4
---

# Post-snapshot (gollapudi-2023-filtered-diskann): index-architecture-global-vs-routed

## TL;DR (delta from yang-2020-pase post)

**Filtered-DiskANN 引入第 7 种 (c) 路由形态：filter-as-routing-criterion**——`F_p ∩ F_q = ∅` 的邻居在 FilteredGreedySearch 中**直接被跳过**，相当于 graph 自身做 filter-based routing。这与系统层 routing（Milvus shard / Pinecone namespace / Manu vchannel）正交：filter-routing 在**单 graph 内**实现选择性遍历。**StitchedVamana 是另一种 (c)**：per-filter 子图 + stitch——本质是"per-filter 局部索引 + 全局合并"的 (c) routing。

## Answer

### Filtered-DiskANN 引入两种 filter-aware (c) 形态（NEW）

[per concepts/filtered-vamana.md "Alg 1 / Alg 5"]

**形态 1：FilteredGreedySearch 在单图内 routing**：
```
扩展候选时：
   N'_out(p*) = {p' ∈ N_out(p*) : F_p' ∩ F_q ≠ ∅, p' ∉ V}
                                    ↑ filter intersection 检查
                                    ↑ 不相交的邻居直接跳过
```

→ **graph 边的"语义路由"**：每跳实际是"基于 label 选择性 traversal"。与传统 (c) routing（先 coarse quantizer 选 partition）不同——是**inline filter-routing**。

**形态 2：StitchedVamana per-filter 子图 + stitching**：
```
对每 filter f：
   G_f = Vamana(P_f, ...)   ← 局部 (c) 索引
对每 vertex v:
   FilteredRobustPrune(v, ∪_f N_f(v), ...)  ← stitching = global merge
```

→ 经典 (c) 路由形态：per-filter 索引 + 全局合并。但 stitching 阶段做得**比 OLAP-style 多 partition 更精细**——每 vertex 边集合并 + RobustPrune 不是简单 union。

### (c) 形态完整对比（updated with Filtered-DiskANN）

| (c) 形态 | 起点 | 路由维度 | scale 上限 | wiki 已 ingest |
|---|---|---|---|---|
| Library + manual sharding | Faiss | hash / range | 1.5T (Meta) | ✓ |
| 算法系统 routing-as-mutable | SPFresh LIRE | NPA centroids | 1B 实证 | ✓ |
| Vector-first DBMS | Milvus / Pinecone | shard / namespace | 千亿+ | ✓ |
| OLAP-extended | ADBV cluster-based | vector geometry partition | 13B production | ✓ |
| OLTP-extended app-layer | PASE | hash by app | million per instance | ✓ |
| **Filter-aware single graph** | **FilteredVamana** | **label intersection in graph traversal** | **28M 实证** | **✓**（本次 ingest） |
| **Per-filter subgraph + stitching** | **StitchedVamana** | **per-filter Vamana + global merge** | **同上** | **✓**（本次 ingest） |

### Filter-as-routing 的概念延伸（NEW）

[per topics/attribute-filtering.md "工业方案对比"]

Filter routing 在 wiki 已有 attribute filter 系统中的实现：

| 系统 | Filter routing 方式 |
|---|---|
| Faiss IDSelector | bitmap check during scan（O(N) hot spot） |
| Milvus Strategy E | 按 filter 属性预 partition（外部 routing） |
| ADBV CBO Plan D | VGPQ + filter 联合 plan（cost-based）|
| Pinecone | namespace + metadata filter |
| **FilteredVamana / StitchedVamana** | **graph traversal 自带 filter check（inline）**|

→ Filtered-DiskANN 是**第一个 graph traversal 自身做 filter routing 的方案**——不依赖 partition / coarse quantizer。

### Microsoft 广告 production 的 (c) 路由实证（NEW）

[per benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md "Microsoft 广告 A/B"]

Microsoft 广告搜索 47 region filter 部署 FilteredVamana：
- **+34.61% clicks, +48.95% revenue**（A/B test, P=0.009-0.03）
- 小 region (<1% share, 27 个): **+70.67%/+79.77%**

→ filter-aware (c) routing 在 production 业务指标 **直接转化为 +35-50% 总营收增益**——是 wiki (c) routing 第一个有 production business metric 的实证。

### 与 query archive 的关系

[queries/index-architecture-global-vs-routed.md] 当时归档说"千亿/万亿主流走 (c) 层次路由"——本 ingest 关闭 query archive 已 flag 但未 expand 的盲区："filter-heavy 场景下 (c) routing 是否有专门设计"——FilteredVamana 与 StitchedVamana 是 wiki 内首个回答。

### 与之前 ingest 的演进

| | yang-2020-pase post | **gollapudi-2023-filtered-diskann post (NEW)** |
|---|---|---|
| (c) 形态数 | 6 | **7-8**（+ Filter-aware single graph + Per-filter stitching） |
| Filter-as-routing 维度 | partition-based / inline-search | **+ graph traversal inline filter check** |
| Production business metric 实证 | 无 | **+ Microsoft A/B +35-50% revenue** |
| (c) 与 attribute filter 关系 | search-time post-processing | **build-time filter-aware** |

### 已知盲区

- **多 filter conjunction (c) routing**：单 filter exact match 是论文限制
- **跨 16 节点的 filter-aware (c) routing**：Filtered-DiskANN 单机 28M 实证；分布式扩展 zero coverage
- **HTAP DBMS + filter-aware (c)**：仍未覆盖
- **Filter set 演化下 (c) 路由的稳定性**：新增 filter 类型时 stitching 是否需重建？StitchedVamana batch only

## Cited Pages

- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
- [systems/diskann.md](../../systems/diskann.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
