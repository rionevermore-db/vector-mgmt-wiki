---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下索引架构选 (a) 全局单一索引 还是 (c) 层次路由结构？"
date: 2026-05-07
phase: post
ingest-context: xu-2023-spfresh
wiki-pages-total: 33
cited-pages: [queries/index-architecture-global-vs-routed.md, systems/spfresh.md, systems/spann.md, systems/diskann.md, systems/milvus.md, concepts/lire.md, topics/in-place-vs-out-of-place-updates.md]
cited-count: 7
---

# Post-snapshot (xu-2023-spfresh): index-architecture-global-vs-routed

## TL;DR (delta from milvus-docs post)

**(c) 层次路由的 update 维度被 [SPFresh](../../systems/spfresh.md) 显著强化**：之前路由结构主要解决 query-time fan-out，update 是后置（周期 rebuild）；SPFresh 在 SPANN-style (c) 路由上加 [LIRE](../../concepts/lire.md) 协议，让**路由结构本身可在 query path 上持续 in-place 演化**——routing 与 update 不再是两个独立轴。**(a) 全局 graph 与 (c) 路由的差距进一步扩大**——graph-based in-place update 仍未解，cluster-based 已有 SPFresh 解。

## Answer

### (c) 路由的 update 维度（NEW）

之前 (c) 路由的 trade-off 表[per queries/index-architecture-global-vs-routed.md]主要从 query 视角；SPFresh 加入 update 视角：

| (c) 路由系统 | 路由层 | Update 模型 | 路由是否随 update 演化 |
|---|---|---|---|
| Faiss IVF + HNSW coarse | HNSW centroids | freeze + 周期 retrain | ✗ |
| Meta 1.5T Faiss mmap | 10M centroids HNSW | freeze | ✗ |
| SPANN | SPTAG centroids | 周期 rebuild centroid | ✗（centroid 一次训完冻结） |
| Milvus 1.x segment | segment-level coarse | LSM merge | 部分（segment level） |
| Milvus 2.x（vchannel） | shard 路由 | + Streaming Node binding | shard 不 fluid，segment 内 LSM |
| **[SPFresh](../../systems/spfresh.md)** | **SPTAG centroids + 持续 split/merge** | **In-place LIRE** | **✓（centroid 集合持续演化）** |

→ SPFresh 是 wiki 现有 source 中**唯一的 routing-as-mutable** 系统。其他系统的 routing 层都是"训完冻结、周期重训"。

### (a) 与 (c) 在 update 维度的对比（NEW）

[per topics/in-place-vs-out-of-place-updates.md]：

| | (a) 全局 graph | (c) 路由（cluster-based） |
|---|---|---|
| Update in-place 解 | **未解**（graph-based） | ✓（SPFresh） |
| Update 周期 rebuild | streamingMerge（DiskANN） | 周期 partition 重训（SPANN/Faiss） |
| Routing 层是否 mutable | n/a | **SPFresh 实证可** |
| Recall 漂移修复 | rebuild 后才 | LIRE 持续修复（NPA） |

→ **(c) 在 update 维度上完胜 (a)**。给定持续 update workload，cluster-based 是更工程化路线。

### Milvus segment + SPFresh 集成的开放方向（NEW）

[per systems/spfresh.md "未覆盖"]：

**理论可行**：Milvus 的 segment 级 index 选择允许每 segment 独立 index_type；可在 segment 内用 SPFresh-LIRE 替代 DiskANN/HNSW。但：
- Milvus v2.6.x 索引族不含 SPFresh
- Milvus segment 默认是"成熟后冻结"语义（LSM merge），与 SPFresh 持续 in-place 矛盾
- 需要 Milvus DBMS 层与 SPFresh 算法层重新协调（segment 边界、tombstone GC、CDC 一致性）

是 **DBMS + algorithmic in-place update** 的最后一公里——目前无人做。

### 与 query archive 的差异

[queries/index-architecture-global-vs-routed.md] 当时归档说"千亿/万亿主流走 (c)"——SPFresh ingest 后这一结论强化：**(c) 在 update 维度也胜出**。但 archive 没修改（per CLAUDE.md "queries/ 是冻结的快照"）。

### 与之前两轮 ingest 的演进

| | wang-2021 post | milvus-docs post | **xu-2023-spfresh post (NEW)** |
|---|---|---|---|
| 路由层 mutability | 未讨论 | 隐含（segment merge） | **显式 mutable（LIRE）** |
| (a) vs (c) update 对比 | 未对比 | 仅 query | **+ update 维度** |
| (c) 系统形态数 | Meta + SPANN + Milvus | + Milvus 2.x cloud-native | **+ SPFresh routing-as-mutable** |
| Graph-based 在 (a) 的局限 | 未深入 | 未深入 | **明确：update 是核心局限** |

### 已知盲区

- **跨 shard SPFresh**：分布式 (c) 路由 + in-place
- **Pinecone pod-based**：仍未覆盖
- **Milvus + SPFresh 集成**：理论可行未实证
- **(a) graph-based + in-place 解**：FreshDiskANN 等后继工作 wiki 未 ingest

## Cited Pages

- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/lire.md](../../concepts/lire.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
