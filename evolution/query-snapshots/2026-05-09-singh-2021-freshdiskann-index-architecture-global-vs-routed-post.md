---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下，索引架构 (a) 全局单一索引 vs (c) 层次路由结构的 trade-off？"
date: 2026-05-09
phase: post
ingest-context: singh-2021-freshdiskann
wiki-pages-total: 61
cited-pages: [systems/freshdiskann.md, systems/spfresh.md, concepts/freshvamana.md, queries/index-architecture-global-vs-routed.md]
cited-count: 4
---

# Post-snapshot (singh-2021-freshdiskann): index-architecture-global-vs-routed

## TL;DR (delta from wang-2024-starling post)

**FreshDiskANN 给 (a) global 单一索引路径 添加 streaming-friendly 维度**——之前 wiki 视 (a) global 路径不可行（单实例 RAM 不够装千亿）；FreshDiskANN 实证 1B SIFT single-machine 128 GB RAM streaming——**(a) 路径在 streaming workload 下重新可行**（虽然限制在 ~1B per machine）。**对 (c) 层次路由的影响**：(c) 内 leaf 现在有 streaming-ready 选项（FreshDiskANN per leaf segment）——之前 leaf 是 static DiskANN，update 触发周期 rebuild。**核心 insight**：**update model 与 architecture model 正交**——(a) / (c) 是 architecture 选择；static / streaming 是 update 选择；4 种组合并存。

## Answer

### 与之前 ingest 的演进

| | wang-2024-starling post | **singh-2021-freshdiskann post (NEW)** |
|---|---|---|
| (a) global 单一索引 实证 | + RaBitQ 让单机 ~10B 可行 | **+ FreshDiskANN single-machine 1B streaming** |
| (c) 层次路由 实证 | per-machine 多 segment | 不变 |
| (c) leaf 形态 | static (DiskANN/Starling) | **+ streaming-ready (FreshDiskANN per-segment)** |
| Update model 与 architecture model 关系 | 隐含同一 | **explicitly 正交（4 种组合 matrix）** |

### Architecture × Update Model 矩阵（NEW）

[per queries/index-architecture-global-vs-routed.md + 累积 ingest]

| | **Static index** | **Streaming index** |
|---|---|---|
| **(a) Global single index** | DiskANN 1B SIFT / Faiss 1.5T mmap | **FreshDiskANN 1B SIFT (NEW)** / SPFresh 1B (cluster path) |
| **(c) Hierarchical routing** | SPANN @ Bing / Milvus segment / Starling segment | per-segment FreshDiskANN / per-segment SPFresh **(空白 frontier)** |

→ 4 种组合都有 viable 选项；之前 wiki 隐含"static + global"（DiskANN）/"static + routed"（SPANN）；FreshDiskANN 解锁 streaming 维度。

### (a) global streaming 的现实约束（NEW）

[per systems/freshdiskann.md "Scale 边界" + 推断]

FreshDiskANN single-machine 实证 800M / 1B SIFT；理论 (a) global streaming 上限：

| Constraint | Value |
|---|---|
| Per-machine RAM budget | ~128 GB（FreshDiskANN 实证）|
| Per-machine disk budget | ~3.2 TB NVMe（实证）|
| 1B 768-d uint8 vec data size | ~750 GB raw |
| 1B 768-d float32 + neighbor IDs | ~3 TB（fits in 3.2 TB SSD） |
| **Single-machine streaming 1B 1024-d 上限** | **~800M-1.5B vectors** |

→ **Single-machine streaming 不到千亿**——千亿仍必须 (c) 层次路由 + 多 segment。

### (c) routing 内 leaf 的 streaming 升级（NEW）

[per systems/starling.md "Open Questions" + 推断]

之前 wiki 视 (c) routing leaf 为 **static**——leaf segment 内 disk index（DiskANN / Starling）；update 触发周期 rebuild。

FreshDiskANN 让 leaf 可以**每个都是 streaming-ready**：

```
Routing Layer 1: cluster routing
   ↓
Routing Layer 2: per-machine multi-segment
   ↓
Layer 3: per-segment streaming index
   ┌────────────────────────────────────────┐
   │  FreshDiskANN per segment              │
   │  - LTI on SSD                          │
   │  - TempIndex in DRAM                   │
   │  - StreamingMerge background           │
   │  - 1800+1800 inserts/deletes per sec  │
   │  - 5.25× faster merge vs full rebuild  │
   └────────────────────────────────────────┘
```

→ 这种"streaming routing"组合 wiki 内 zero coverage——是 logical work。

### 在 16 节点 × 1TB 私有云的具体决策（updated）

[per queries/index-architecture-global-vs-routed.md + benchmarks/freshdiskann-streaming-sift800m.md]

| 决策 | 推荐 | FreshDiskANN 影响 |
|---|---|---|
| 总体架构 | (c) 层次路由 + per-machine 多 segment | 不变 |
| Per-segment 大小 | 800M-1B (per FreshDiskANN benchmark) | **NEW: per-segment 大小由 FreshDiskANN single-machine RAM bound** |
| Per-segment update model | 周期 rebuild | **NEW: 持续 streaming insert/delete** |
| Per-machine segment 数 | 8 (1TB / 128GB FreshDiskANN per segment) | NEW |
| 总 segment 数 | 千亿 / 1B = ~100 segments | 总 1000 segments → 16 nodes × 60 segments per node 假设较小 segment |
| Update 触发频率 | 周级 batch | **streaming + 周级 merge consolidation** |
| Build time per segment | n/a (streaming) | **StreamingMerge 4.4 h per 800M segment per 30M change** |

→ FreshDiskANN 让"周级别 update"从"5 天 unrealistic" 变成"每 segment 4.4h merge consolidation, 100 segments parallel ≈ 8 days continuous"——但 streaming model 让"实时 freshness"成为新 default。

### 已知盲区

- **跨 segment streaming routing**：单 segment FreshDiskANN OK；跨 segment update routing + search aggregation 协议未涉及
- **(a) + (c) hybrid + streaming**：完全空白（整 cluster 用一种 architecture，混用未深入）
- **Streaming + filter / multi-vector / iterator**：完全空白
- **HCPS + (c) + streaming**：完全空白

## Cited Pages

- [systems/freshdiskann.md](../../systems/freshdiskann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [concepts/freshvamana.md](../../concepts/freshvamana.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
