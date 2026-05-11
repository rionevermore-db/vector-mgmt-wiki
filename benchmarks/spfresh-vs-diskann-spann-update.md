---
title: SPFresh vs DiskANN/SPANN+ on 100-Day Update Simulation
type: benchmark
sources: [xu-2023-spfresh]
related: [../systems/spfresh.md, ../systems/diskann.md, ../systems/spann.md, ../concepts/lire.md, ../topics/in-place-vs-out-of-place-updates.md]
created: 2026-05-07
updated: 2026-05-07
---

# SPFresh vs DiskANN / SPANN+ on 100-Day Update Simulation

**TL;DR**: SPFresh 论文 §5 的核心实验——100 天 × 1% daily update 模拟。SPFresh 在 search latency / accuracy / resource 三方面**全程稳定**击败 DiskANN（out-of-place rebuild）+ SPANN+（无 rebalance baseline）。**P99.9 latency 平均 2.41× 更低**于 DiskANN；**memory 5.30× 更小**；rebuild 资源 1100 GB→**10 GB**、32 cores→**2 cores**。Billion stress test 单 NVMe SSD 饱和 400K IOPS @ 4K QPS search + 2K QPS update。[xu-2023-spfresh §5]

## 实验设置

[xu-2023-spfresh §5.1]

- **平台**：Azure lsv3 VM（storage-optimized）
  - 16 hyper-threaded vCPU Intel Xeon Platinum 8370C (Ice Lake)
  - 128 GB RAM
  - 本地 NVMe SSD（max guaranteed 400K IOPS）
- **数据集**：
  - **SIFT1B**（128-d byte vectors，1B base + 10K query）
  - **SPACEV1B**（100-d byte vectors，1B base + 29316 query；Microsoft Bing 真实生产数据，**deep natural language encoding**）
- **Workload 设计**：
  - **A**：100M scale (SPACEV)，1% daily update × 100 days
    - 选 100M 而非 1B 因为 DiskANN 1B in-memory rebuild 需 TB 级 DRAM，超 VM 容量
    - 每天：1% delete random + 1% insert from 100M candidate pool
  - **B**：100M scale (SIFT)，同 A workload
  - **C**：1B 全规模 stress test，20 天 × 1% daily update
- **Baselines**：
  - **DiskANN**（out-of-place 全局 rebuild，graph-based）：streamingMerge + insert candidate list 75 + degree R=64 + beamwidth=2 + recall10@10 search list L=40
  - **SPANN+**（消融 baseline）：modified SPANN with append-only + tombstone，**without** split/reassign。即 SPFresh 去掉 LIRE Local Rebuilder
- **Metrics**：
  - Search latency: tail (P90/P95/P99/P99.9) + QPS（10ms hard cut，超时返回当前结果）
  - Search accuracy: ground truth recall%
  - Update perf: insert/delete throughput
  - Resource: memory + CPU consumption
- **Thread allocation**（Workload A，[xu-2023-spfresh Table 2]）：
  - DiskANN: insert 3 / delete 1 / search 2 / background 10 = **16 total**
  - SPANN+ / SPFresh: insert 1 / delete 1 / search 2 / background 2 = **6 total**

## Workload A 主结果（SPACEV 100M, 100 days, [Fig 7]）

### Search Tail Latency

| | DiskANN | SPANN+ | **SPFresh** |
|---|---|---|---|
| P90 | 4-6 ms（rebuild 时跳到 10ms） | 4-8 ms 缓涨 | **稳定 ~3 ms** |
| P95 | 6-10 ms（rebuild 跳到 12ms） | 5-10 ms 缓涨 | **稳定 ~4 ms** |
| P99 | 8-15 ms（rebuild 跳到 20ms） | 6-10 ms 缓涨 | **稳定 ~4 ms** |
| **P99.9** | **波动 4-25 ms**（rebuild 飙到 >20ms） | **缓涨 4 → >10 ms** | **稳定 ~4 ms** 全程 |

[xu-2023-spfresh Fig 7]：定性观察：
- DiskANN 的 P99.9 在 global rebuild 周期内被全堵塞（10ms hard cut 触发）→ 飙到 20ms+
- SPANN+ 的 P99.9 缓慢涨——partition 分布累积 skew，搜索路径变长
- **SPFresh P99.9 全程平稳 ~4ms**——LIRE 持续维持 well-partitioned 性质

> **2.41× 平均优势** = SPFresh 在 100 天累积 P99.9 比 DiskANN 平均低 2.41×

### Insert Throughput

| | DiskANN | SPANN+ | SPFresh |
|---|---|---|---|
| Per-thread insert | ~400 QPS | ~700 QPS | **~700 QPS** |

DiskANN 单 insert 慢因为 graph traversal in-memory 重；SPANN+/SPFresh 都是单 posting append。

### Search Accuracy（Recall 10@10）

| | DiskANN | SPANN+ | SPFresh |
|---|---|---|---|
| Day 0 | ~0.78 | ~0.78 | ~0.78 |
| Day 50 | ~0.55-0.6（数据漂移退化） | ~0.85（缓涨） | ~0.85 |
| Day 100 | ~0.5-0.55 | ~0.88 | **~0.88** |

[xu-2023-spfresh Fig 7]：
- DiskANN accuracy 随 deletion 累积**下降**——pruned graph edges 不重建会失去 reachability
- **SPANN+ 与 SPFresh accuracy 都缓涨**——新插入向量集中到 subset of postings，新查询碰这些 hot postings 时 recall 高
- 但论文承认 SPANN+ 的 P99.9 latency 在涨——"高 recall + 高 latency" trade-off 暴露

### Memory Usage

| | DiskANN | SPANN+ | SPFresh |
|---|---|---|---|
| Day 0 | ~80 GB | ~20 GB | ~10 GB |
| Day 100 | ~110 GB（streamingMerge 缓冲） | ~30 GB | **<20 GB** |

**SPFresh 5.30× lower memory**。
- DiskANN 60 GB 给 streamingMerge 背景 + 15 GB in-memory delta index
- SPANN+ block-mapping entries 多（posting 长度增长）
- SPFresh metadata 仅随 split 缓涨（分裂创新 posting）

## Billion-Scale Stress Test（Workload C，[Fig 9]）

### 设置

- 同硬件，1B 数据
- 8 search threads + 4 update threads + 3 background = 15 cores
- max QPS 由 NVMe IOPS 上限决定（Azure lsv3 max guaranteed 400K）
- Search threads 在 8 时饱和（[Fig 8]：QPS / IOPS plateau at 8 threads）

### 结果（20 天）

| Workload | P99.9 latency | Search throughput | Insert throughput | NVMe IOPS | Memory | CPU |
|---|---|---|---|---|---|---|
| Uniform (SIFT) | ~5 ms | ~3000 QPS | ~2000 QPS | **饱和 400K**（≥ guaranteed limit） | ~74 GB | 1300% (= 13 cores) |
| Skew (SPACEV) | ~6.5 ms | ~3000 QPS | ~2000 QPS | 饱和 | ~74 GB | 1300% |

**关键**：SPFresh 在 1B 上**已成 IOPS-bound**——CPU + memory 都没饱和，IOPS 是真瓶颈。这意味着多 NVMe / 多机能立刻 scale。

### Recall 稳定性（[Fig 9]）

- Uniform：accuracy >= **0.862** during 20-day stress（搜 nearest 64 postings）
- Skew：accuracy >= **0.807** during 20-day stress

## Data Distribution Shifting Micro-Benchmark（[§5.4 Fig 10]）

实验：从 Static index 出发，逐步加 LIRE sub-component：

| Setting | Recall vs Latency 曲线位置 |
|---|---|
| **Static** (target) | 最佳 (NW corner) |
| **In-place + Split + Reassign**（完整 SPFresh） | **几乎贴 Static** |
| In-place + Split only（无 reassign） | 略差 Static |
| In-place 无 split 无 reassign（≈ SPANN+） | 显著差（recall ↓ + latency ↑） |

→ Reassign 与 split 都不可缺，**Reassign 是 LIRE 主要价值**。

## Parameter Studies（§5.5）

### Reassign Range（[Fig 11]）

| Reassign top-K nearby postings | Recall 提升 | 备注 |
|---|---|---|
| 0（仅 split，no reassign） | baseline | LIRE 缺失 |
| 8 | +moderate | |
| **64** | **+大** | **SPFresh 默认** |
| 128 | +marginal | 性价比下降 |

**结论**：64 nearby postings 检查足够 maintain index 质量。

### Foreground/Background Thread Ratio（[Fig 12]）

- 8-threaded foreground Updater 需要 **≥4 threads** background Local Rebuilder
- **2:1 ratio**（前台:后台）平衡最优 → SPFresh 默认配置

## 与其他 wiki 大规模部署对比

| 部署 | 数据 | 索引 | 介质 | Update 模式 | Recall | 单查询延迟 |
|---|---|---|---|---|---|---|
| Faiss-GPU SIFT1B | 1B | IVFPQ + GPU | HBM | freeze | 高 | <0.1 ms |
| NSG @ Taobao | 2B | NSG 分布式 | DRAM × 32 机 | static | ~98% | ~5 ms |
| Faiss trillion @ Meta | 1.5T | SQ6 + HNSW coarse | mmap | freeze | — | ~1 s |
| DiskANN @ z840 | 1B | Vamana + PQ + SSD | DRAM + NVMe | streamingMerge | ~98% | ~3-5 ms |
| SPANN @ Bing | 1B+ / 千亿+ | HBC + closure + SSD | DRAM + NVMe | static + 周期 rebuild | >90% | ~1 ms |
| Milvus 1.x SIFT1B | 1B | IVF_FLAT + HNSW etc | DRAM 1.5 TB | LSM | — | — |
| **SPFresh @ Azure lsv3** | **1B** | **SPANN + LIRE** | **DRAM + NVMe** | **In-place 持续** | **>0.86** | **~5 ms** |

SPFresh 在 wiki 已有 6 个 1B+ 部署中**唯一**：(1) 持续 in-place update + (2) ~10 GB memory + (3) 单机 + (4) IOPS 饱和（不是 CPU/memory bound）的组合。

## 可信度评估

- **实验设计**：作者作为 SPFresh 团队，但 baselines（DiskANN, SPANN+）使用的是 published / open-source 代码 + 论文默认参数。SPANN+ 是 SPFresh 团队修改 SPANN 自实现的——可能 tilted 不利于 SPANN+
- **潜在偏向**：
  1. **DiskANN 1B rebuild 的资源数字 (1100 GB / 32 cores)** 来自 DiskANN 论文 [subramanya-2019-diskann]——但 SPFresh 100-day workload 用 100M scale 而非 1B（VM 装不下 DiskANN 1B），所以 1B vs 100M 的延迟数据不直接可比
  2. **Workload A/B 用 1% daily update**——非常温和；burst 写场景（10% / 50% daily）下 SPFresh LIRE 触发率与 cascading 上限未量化
  3. SPACEV1B "skew" 是 Microsoft Bing 自家数据集——SPFresh 在此优势最大有"主场"嫌疑（与 [SPANN](../systems/spann.md) benchmark 同样问题）
  4. **不与 [Milvus](../systems/milvus.md) LSM 对比**——尽管 Milvus 的 LSM 也是 dynamic data 解；可能因为 Milvus 是 DBMS 整体而 SPFresh 是算法系统，对比点不直接
  5. **Recall 数字 0.862 / 0.807** 是 stress test 期间——比 SPANN 论文 (Bing 实测 >0.9) 略低，可能因为 search candidate L=40 较小
- **复现难度**：中。SPFresh 代码基于 [Microsoft SPTAG](https://github.com/microsoft/SPTAG) 衍生，应公开（论文未明确给 GitHub link）；BIGANN 数据公开
- **场景局限**：
  - L2-NN（不是 [MIPS](../topics/mips-vs-l2-nn.md)）；MIPS 任务下 LIRE 未验证
  - 维度 ≤ 128；高维 deep embedding（768-d / 1024-d）未测
  - 单机 / 单 NVMe；多 SSD / 分布式未测（论文 §6 明示 future）
  - 同步 update（每天发起）；burst / async event-driven update 未测
  - 100 days / 20 days 实验时长——年级 update 累积下 LIRE 行为未知

Cited by: [queries/giga-scale-sharding.md](../queries/giga-scale-sharding.md)
