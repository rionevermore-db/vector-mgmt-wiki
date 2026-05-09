---
title: FreshDiskANN Streaming Workload on 800M SIFT
type: benchmark
sources: [singh-2021-freshdiskann]
related: [../systems/freshdiskann.md, ../systems/diskann.md, ../systems/spfresh.md, ../concepts/freshvamana.md, ../concepts/vamana.md, ../topics/in-place-vs-out-of-place-updates.md]
created: 2026-05-09
updated: 2026-05-09
---

# FreshDiskANN Streaming Workload on 800M SIFT

**TL;DR**: FreshDiskANN arXiv 2021 论文 §6 在 800M SIFT 上 week-long steady-state streaming 实测——sustaining **1800 inserts/sec + 1800 deletes/sec + 1000 search/sec @ 95+% 5-recall@5** on **single machine** (96 thread + 3.2 TB NVMe + ~125 GB RAM peak)。**核心发现**：(a) StreamingMerge **5.25× faster than full rebuild** (15832s vs 83140s on 800M / 30M change); (b) burst capacity **40,000 inserts/sec**; (c) FreshVamana build itself **1.48-1.83× faster than static Vamana** while supporting streaming; (d) recall stable over 50 cycles 5%/10%/50% change rate; (e) PLSH (state-of-art LSH-based fresh-ANNS) needs **25× more machines** for same task.

## 实验设置

[singh-2021-freshdiskann §6.1]

### 硬件

| 配置 | 用途 | 规格 |
|---|---|---|
| **mem-mc** | in-memory algorithms + FreshVamana RAM ablations | Azure E64d_v4 VM (64-vcore) |
| **ssd-mc** | full FreshDiskANN system | bare-metal 2× Intel Xeon 8160 (48 cores, 96 threads) + Samsung PM1725a 3.2 TB PCIe NVMe SSD |

### 数据集

| Dataset | Size | D | Type |
|---|---|---|---|
| **SIFT1M** | 1,000,000 | 128 | float32 (image descriptor) |
| **DEEP1M** | 1,000,000 | 96 | float32 (CNN-generated) |
| **GIST1M** | 1,000,000 | 960 | float32 (image descriptor) |
| **SIFT100M** | 100,000,000 | 128 | float32 (subset of SIFT1B) |
| **SIFT1B** | 1,000,000,000 | 128 | uint8 (largest public test) |

### 关键参数

[singh-2021-freshdiskann §6.2]

| Parameter | Value | 备注 |
|---|---|---|
| `R` (max degree) | 64 | FreshVamana out-degree |
| `L_c` (search list size during build) | 75 | candidate generation |
| `α` | **1.2** | streaming-stable α-RNG |
| PQ compression target | 32 bytes/vector | LTI compressed PQ codes |
| Max TempIndex 总大小 M | 30M points | merge 触发阈值 |
| StreamingMerge background threads T | 40 | 默认 |
| Search threads | 10 | 并发 |

## 主结果

### Result 1: FreshVamana Recall Stability（Fig 2, 50 cycles）

[singh-2021-freshdiskann §4.3]

50 cycles × {5%, 10%, 50%} delete + re-insert on million-scale + SIFT100M。L_s 选择使初始 5-recall@5 ≈ 95%：

| Dataset | 5% change recall after 50 cycles | 10% change | 50% change |
|---|---|---|---|
| SIFT1M | **95%** stable | **95%** stable | **95%** stable |
| DEEP1M | 95%+ stable | 95%+ stable | 95%+ stable |
| GIST1M | 95%+ stable | 95%+ stable | 95%+ stable |
| SIFT100M | 95% stable | 95% stable | 95% stable |

→ 4 datasets × 3 change rates = **12 个 stability 实验全部 pass**。对比 Vamana(α=1) / HNSW / NSG 在同实验下 95% → 88-90%——**FreshVamana α=1.2 是 graph fresh-ANNS 必要条件**的实证。

### Result 2: Effect of α on Recall Stability（Fig 3）

[singh-2021-freshdiskann §4.3]

50 cycles × 5% change on SIFT1M / DEEP1M with varying α：

| α | 50 cycles 后 recall | 备注 |
|---|---|---|
| **1.0** | **95% → 90%** | 等价于 HNSW/NSG 隐式参数；**fail** |
| 1.1 | 95% → 94% | 仍有下降 |
| **1.2** (default) | **95% stable** | FreshVamana default |
| 1.3 | 95% stable | 略低（degree higher） |

### Result 3: FreshVamana Build Speedup（Table 1, mem-mc）

[singh-2021-freshdiskann §B Table 1]

R=64, L_c=75, α=1.2 同 build params：

| Dataset | Static Vamana | **FreshVamana** | Speedup |
|---|---|---|---|
| SIFT1M | 32.3 s | **21.8 s** | **1.48×** |
| DEEP1M | 26.9 s | 17.7 s | 1.52× |
| GIST1M | 417.2 s | **228.1 s** | **1.83×** |
| SIFT100M | 7187.1 s | 4672.1 s | 1.54× |

→ FreshVamana 是 **strict superset of static Vamana**——build 更快 + 支持 streaming + 同 recall。逻辑上替代 default Vamana。

### Result 4: StreamingMerge vs Full Rebuild（Table 2, ssd-mc）

[singh-2021-freshdiskann §6.2 Table 2]

800M SIFT index + 30M change（30M inserts + 30M deletes, ~7.5% of index）：

| Method | Threads | Time | Speedup |
|---|---|---|---|
| DiskANN full rebuild | 96 | **83140 s ≈ 23 h** | 1× baseline |
| **FreshDiskANN StreamingMerge** | **40** | **15832 s ≈ 4.4 h** | **5.25×** |

**注意**：StreamingMerge 用 less than half threads (40 vs 96) 仍 5.25× faster——因为它**只处理 change set |D|+|N|** 而非全 index |P|。

### Result 5: FreshDiskANN Steady-State on 800M（Fig 6, ssd-mc）

[singh-2021-freshdiskann §6.2]

Week-long experiment, 800M index size, 持续 inserts + deletes + searches：

| Metric | Value |
|---|---|
| **Sustained insert rate** | **1800 inserts/sec** |
| Sustained delete rate | 1800 deletes/sec |
| Sustained search rate | 1000 searches/sec |
| Search recall | **95+% 5-recall@5** |
| **User insert latency mean** | **~1 ms** |
| User insert latency 99p | ~1.5 ms (Fig 6 right) |
| User delete latency | <0.1 μs (just append) |
| Search latency mean (steady) | 5-15 ms |
| Search latency 99p (steady) | <20 ms |
| Search latency 99p during merge | 25-40 ms（temporary spikes） |
| Peak RAM usage | ~125 GB |

### Result 6: Burst Capacity（§6.2）

[singh-2021-freshdiskann §6.2]

短期 burst：FreshDiskANN 可以接受 **40,000 inserts/sec** for short bursts（限制因素：RW-TempIndex 增长速度）。这是 cache 启动 / 数据 import / 紧急 backfill 场景的关键能力。

### Result 7: Cost vs PLSH（State-of-Art LSH Fresh-ANNS）

[singh-2021-freshdiskann §1.1]

PLSH (Sundaram 2013, parallel LSH) 是当时 fresh-ANNS state-of-art：

| 系统 | 1B SIFT machines |
|---|---|
| PLSH (32 GB RAM each) | **~25 machines** |
| **FreshDiskANN (128 GB RAM)** | **1 machine** |
| Saving | **5-10× cost reduction** |

PLSH RAM-heavy 因 LSH 需要存数百 hash function；FreshDiskANN SSD-resident → RAM 仅存压缩 PQ + TempIndex。

### Result 8: Recall During Steady-State Merge（Fig 4, 800M & 80M）

[singh-2021-freshdiskann §5.5]

Merge 过程中 recall：

| Dataset | Cycle 0 recall | Cycle 20+ steady recall | 备注 |
|---|---|---|---|
| 80M index, 5%+5% change | 99% | 97% | 初始降 2%, then stable |
| 800M index, 30M+30M change | 95% | **92.5%** | 初始降 2.5%, then stable |

→ **Recall 因为 merge 时用 PQ approximate distance** 略降但**不再下降**。Trade-off acceptable for the 5.25× faster merge.

### Result 9: Search Latency During Merge Phases（Fig 6 middle, Fig 8）

[singh-2021-freshdiskann §6.2]

Merge 三阶段对 search latency 的影响：

| Phase | Search latency mean | 备注 |
|---|---|---|
| Steady (no merge) | 5-15 ms | normal |
| **Delete phase** | 7-12 ms | minor SSD I/O contention |
| **Insert phase** | 8-15 ms | large GreedySearch 在 intermediate-LTI |
| **Patch phase** | **15-25 ms** | dual sequential SSD passes 引发争抢 |

→ 用户感知 latency degradation **小且短期**——大部分时间 steady。

### Result 10: Scaling with Threads（Fig 7）

[singh-2021-freshdiskann §6.2]

| # threads | StreamingMerge time | Search throughput |
|---|---|---|
| 10 | reference | 1500 QPS |
| 20 | 1.88× faster | 3000 QPS |
| **40** | **2.97× faster** | 5500 QPS |
| 64 | 3.59× faster | **6500 QPS** |

→ Near-linear scaling for both merge speed + search throughput.

## 可信度评估

### 实验设计

- ✓ 完整 hardware spec 公开（Azure VM + bare-metal Xeon）
- ✓ 多 dataset (1M to 1B subset)
- ✓ Week-long steady-state 实证（不是短期 burst）
- ✓ Recall stability over 50 cycles
- ✓ Open source 实现（microsoft/DiskANN）
- ✗ **未直接对比 SPFresh** — SPFresh 2023 后发表，FreshDiskANN 2021 自然不知 SPFresh
- ✗ **billion-scale 实测仅 ramp-up to 800M**——1B steady-state 未直接 demo

### 复现难度

- microsoft/DiskANN 开源代码
- SIFT1B / SIFT100M / SIFT1M 公开 dataset
- Azure E64d_v4 / Xeon 8160 + PCIe NVMe 是标准 enterprise hardware
- 全 parameter (R=64, L_c=75, α=1.2, PQ B=32) 公开

### 偏向

- **作者团队是 DiskANN 原作者**——对照 baseline DiskANN 自然 favorable
- **PLSH baseline 数据来自 PLSH 原论文 [54]**（2013）——非 FreshDiskANN 团队 re-run；可能旧
- **未对比 HNSW**——HNSW 在 fresh-ANNS 不可行（recall 退化），但工业仍广泛 deploy；论文应给 head-to-head
- 但 Open source + multiple datasets + ablation 充分；偏向受限

### 数据集偏向

- SIFT 是 image descriptor，分布相对 well-behaved
- Random delete + insert 是 best case；adversarial delete pattern（删 hub vertices）未实证
- 分布漂移（不再是 same dataset 的 random subset）未实证

## Open Questions

- **vs SPFresh head-to-head**：两者 2023 同时存在但 SPFresh 论文比较 baseline 时仅对比 DiskANN/SPANN，未对比 FreshDiskANN。Memory budget 比较（~128 GB FreshDiskANN vs ~4 GB SPFresh for 1B）暗示 graph path 更耗内存
- **HNSW α-augmented streaming variant**：理论上同样可工作；未在 FreshDiskANN paper 实证（仅作为 negative baseline）
- **Filter / multi-vector + streaming**：FreshDiskANN 仅 pure ANNS；与 [FilteredVamana](../concepts/filtered-vamana.md) / multi-vector 联合未实证
- **Adversarial workload**：Random delete OK；删 hub vertices / 集中删某 cluster 等 adversarial pattern 未测
- **Cross-segment streaming**：单 LTI 实证；多 segment（Milvus / Manu / Starling 模式）跨 segment update coordination 未涉及
- **embedding model migration**：所有 wiki source 一致——zero coverage
