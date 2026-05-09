---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-09
phase: post
ingest-context: singh-2021-freshdiskann
wiki-pages-total: 61
cited-pages: [systems/freshdiskann.md, systems/diskann.md, systems/spann.md, systems/spfresh.md, topics/disk-vs-memory-ann.md]
cited-count: 5
---

# Post-snapshot (singh-2021-freshdiskann): memory-vs-disk-large-scale

## TL;DR (delta from wang-2024-starling post)

**FreshDiskANN 给 disk-resident graph 路径添加 streaming 维度**——之前 wiki 视 [DiskANN](../../systems/diskann.md) 为 static disk graph，rebuild 周期成本高（1B 1100 GB DRAM × 32 cores × 2 天）；FreshDiskANN 在同 Vamana base 上加 [FreshVamana](../../concepts/freshvamana.md) + StreamingMerge，让 disk graph **streaming-ready**：800M SIFT 1800+1800 inserts/deletes/sec sustained。**Memory-vs-disk landscape 现在多一个新维度**：**static vs streaming**——两者都有 single-server / segment-level 子分化。**对 single-machine 1B+ 部署的影响**：FreshDiskANN ~128 GB RAM 1B 远高于 SPFresh ~4 GB，但 graph path search recall 通常更高——**新决策点：memory budget vs recall ceiling**。

## Answer

### 与之前 ingest 的演进

| | wang-2024-starling post | **singh-2021-freshdiskann post (NEW)** |
|---|---|---|
| Disk graph static path | DiskANN single-server / Starling segment | **不变** |
| Disk graph streaming path | 完全空白 | **+ FreshDiskANN single-server 800M / 1B** |
| Cluster path streaming | SPFresh ~4 GB 1B | 不变 |
| Graph path streaming | **完全空白** | **+ FreshDiskANN ~128 GB 1B** |
| Memory budget vs recall | 仅有 cluster 路径选项 | **+ graph 路径选项（30× more memory but higher recall ceiling）** |

### 路线对比表（updated with FreshDiskANN）

| 路线 | 数据驻留 | static / streaming | 量化 | 1B memory budget | scale 实证 |
|---|---|---|---|---|---|
| 全内存 graph (HNSW/NSG) | DRAM | static | ✗ | OOM | 1B OOM |
| 磁盘 + 量化 + SSD re-rank（DiskANN）| DRAM (PQ) + SSD | **static** | PQ | ~64 GB | 1B SIFT |
| 磁盘 + IVF + SSD 全精度（SPANN）| DRAM (centroids) + SSD | **static** | ✗ | ~32 GB | 1B+ Bing |
| 磁盘 + Filter-aware Vamana + PQ DRAM（Filtered-DiskANN）| DRAM + SSD | static | PQ | ~64 GB | 28M DANN |
| 全内存 + Predicate-Agnostic HNSW（ACORN）| DRAM only | static | ✗ | OOM | 25M LAION |
| VBASE + HNSW (in-mem) | DRAM only | static | ✗ | OOM | 330K Recipe1M |
| VBASE + SPANN | DRAM + SSD | static | ✗ | Recipe1M demo | 330K |
| RaBitQ + IVF (RAM only) | DRAM only | static | RaBitQ | ~128 GB | 6 dataset 2.34M |
| Starling (segment-level disk graph) | DRAM + SSD | **static (with hint of streaming)** | PQ | per-segment | BIGANN 1B 31 segs |
| **SPFresh + LIRE (cluster path streaming)** | DRAM + SSD | **streaming** | ✗（继承 SPANN）| **~4 GB** | **1B SIFT 100 days** |
| **FreshDiskANN + FreshVamana (graph path streaming, NEW)** | **DRAM + SSD** | **streaming** | **PQ short codes for routing** | **~128 GB** | **800M SIFT week-long, 1B ramp-up** |

### 路径选择决策（NEW: streaming 维度加入）

[per concepts/freshvamana.md + topics/in-place-vs-out-of-place-updates.md]

| Workload | 路径选择 |
|---|---|
| **静态 1B + 单机** | DiskANN（64 GB）/ SPANN（32 GB）|
| **静态 1B + 多 segment per node** | Starling（segment-level） |
| **Streaming 1B + 内存严格预算** | **SPFresh + LIRE（~4 GB）** |
| **Streaming 1B + 高 recall 要求** | **FreshDiskANN + FreshVamana（~128 GB）** |
| **Streaming + filter / multi-vector** | **完全空白**（all of FreshDiskANN, SPFresh, FilteredVamana, ACORN are static or only partial）|

### 内存预算决定路径选择（NEW）

```
1B SIFT, streaming workload, 95+% recall target

  Memory budget:
  ┌──────────────────────────┐
  │ ≤ 8 GB                  │ → SPFresh + LIRE（cluster path）
  │ 8 - 64 GB                │ → SPFresh（首选） / DiskANN with periodic rebuild
  │ 64 - 256 GB              │ → FreshDiskANN（graph path 高 recall）
  │ ≥ 256 GB（多机分布式）   │ → distributed FreshDiskANN（论文 §1 future work）
  └──────────────────────────┘
```

→ **30× memory budget gap (4 vs 128 GB)** 让 cluster path 在 cost-sensitive 场景胜出；graph path 在 recall-sensitive 场景胜出。

### Update 成本对比（NEW）

[per benchmarks/freshdiskann-streaming-sift800m.md + benchmarks/spfresh-vs-diskann-spann-update.md]

| Method | 1B / 800M update mechanism | Resource cost |
|---|---|---|
| DiskANN full rebuild (1B SIFT) | rebuild from scratch | **1100 GB peak + 32 cores × 2 天** |
| DiskANN low-resource rebuild | rebuild from scratch | 64 GB + 16 cores × 5 天 |
| **SPFresh LIRE (1B steady-state)** | continuous in-place rebalance | **持续 10 GB + 2 cores** |
| **FreshDiskANN StreamingMerge (800M / 30M change)** | two-pass SSD + in-mem TempIndex | **15832 s ≈ 4.4 h on 96-thread + 128 GB** (5.25× faster than full rebuild) |

→ SPFresh 是连续低成本；FreshDiskANN 是周期 medium-cost——两种 update 模式 trade-off 不同。

### 已知盲区（仍未覆盖）

- **FreshDiskANN vs SPFresh head-to-head**：两个论文 2021 / 2023，相互不直接对比；wiki 内仅理论对比
- **FreshDiskANN + filter-aware**：wiki 内 zero coverage（FilteredVamana / NHQ / ACORN 都不支持 streaming）
- **FreshDiskANN + RaBitQ**：两者同年发表（FreshDiskANN 2021, RaBitQ 2024）相互不知；理论可叠加
- **FreshDiskANN + VBASE engine layer**：理论上 FreshVamana 满足 RM (graph 路径)，VBASE engine + FreshDiskANN 叠加 logical 但未实证
- **FreshDiskANN + Starling block shuffling**：StreamingMerge 后 OR(G) 漂移；周期重新 block shuffle 与 merge 的协调未涉及
- **HCPS + streaming + SSD**：完全空白
- **HCPS + streaming + memory**：完全空白

## Cited Pages

- [systems/freshdiskann.md](../../systems/freshdiskann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
