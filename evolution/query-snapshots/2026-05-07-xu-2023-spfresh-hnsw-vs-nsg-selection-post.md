---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是 proximity graph，工程上怎么选？"
date: 2026-05-07
phase: post
ingest-context: xu-2023-spfresh
wiki-pages-total: 33
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/lire.md, systems/diskann.md, systems/spfresh.md, topics/in-place-vs-out-of-place-updates.md]
cited-count: 6
---

# Post-snapshot (xu-2023-spfresh): hnsw-vs-nsg-selection

## TL;DR (delta from milvus-docs post)

**Graph-based 在 in-place update 上的劣势进一步坐实**：[per topics/in-place-vs-out-of-place-updates.md] 显式分析 graph index 的 update 成本根因——high-d 下"shortcut" 维护成本与图大小线性，每次 insert 需识别 hundreds of neighbors。**HNSW / NSG / Vamana 的 in-place update 仍是开放问题**，cluster-based 路线（[SPANN](../../systems/spann.md) → [SPFresh](../../systems/spfresh.md)）已有解。

## Answer

### Graph-based vs Cluster-based 的 update 经济学（NEW）

[per topics/in-place-vs-out-of-place-updates.md "为什么 graph-based 索引没有 in-place 解"]：

| | Graph-based (HNSW / NSG / Vamana) | Cluster-based ([SPANN](../../systems/spann.md)) |
|---|---|---|
| Insert 成本 | 识别 hundreds of neighbors，扫整个 high-d 空间 | 单 partition append + 邻近 NPA 检查 |
| Delete 成本 | 扫整个 unidirectional graph 找 in-edge | tombstone marker，单 posting 写 |
| 数据漂移修复 | 需 rebuild graph edges | LIRE incremental rebalance |
| 可证明收敛 | 无形式化 in-place 协议 | LIRE 形式化收敛证明 [concepts/lire.md] |
| 实测系统 | DiskANN streamingMerge（rebuild） | **[SPFresh](../../systems/spfresh.md) LIRE**（in-place） |

→ **若 workload 涉及持续更新**，cluster-based + SPFresh 是已验证路线；graph-based 必须接受周期 rebuild。

### Update 维度对 HNSW vs NSG 选型的影响（NEW）

| 场景 | 1.x 推荐 | + Update 维度 |
|---|---|---|
| 静态 + million-scale + 高 precision | NSG（更快、更省内存） | 仍是 NSG |
| 静态 + 增量 add 但不 delete | HNSW（NSG 不增量） | 仍是 HNSW |
| 持续 add + delete + 数据漂移 | 都不直接支持 | **都不行**——切到 cluster-based + SPFresh |
| Million-scale + DBMS 形态 | Milvus HNSW/RNSG | + 接受 LSM segment merge 周期成本 |
| Billion-scale + 持续 update | DiskANN streamingMerge（贵） | **SPFresh** 直接（in-place） |

### 与 wang-2021 / milvus-docs post 的差异

| | wang-2021 post | milvus-docs post | **xu-2023-spfresh post (NEW)** |
|---|---|---|---|
| HNSW 增量 add | ✓ 但不 delete | 同 | 同；**仍未解 delete + 数据漂移** |
| NSG 增量 | ✗ 完全不支持 | 同 | 同 |
| Graph-based update 整体 | wiki 未深入 | RNSG 与 DISKANN 分离 | **明确：graph in-place 仍开放** |
| Cluster-based update | wiki 未深入 | 略提 LSM segment | **SPFresh LIRE 完整覆盖** |

### Open / 未覆盖

- **FreshDiskANN [Singh 2021]**：Vamana graph 上的 streaming update 工作，wiki 未 ingest
- **Graph + in-place 的研究方向**：除 FreshDiskANN 外是否有其他方案
- **Multi-line 比较**：实际生产 graph + 周期 rebuild vs cluster + in-place 在 TCO 维度的比较未量化

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/lire.md](../../concepts/lire.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
