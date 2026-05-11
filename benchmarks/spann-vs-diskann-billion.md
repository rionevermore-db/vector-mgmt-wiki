---
title: SPANN vs DiskANN on Billion-scale (SIFT1B / SPACEV1B / DEEP1B)
type: benchmark
sources: [chen-2021-spann]
related: [../systems/spann.md, ../systems/diskann.md, ../topics/disk-vs-memory-ann.md, ../benchmarks/diskann-sift1b.md]
created: 2026-05-07
updated: 2026-05-07
---

# SPANN vs DiskANN on Billion-scale

**TL;DR**: SPANN 论文 §4 的核心实验。在三个 billion-scale 数据集上 SPANN 比 [DiskANN](../systems/diskann.md) 在 90% recall 时快 **2×**，同等内存预算下；同时在 SIFT1M 内存对决中，SPANN 的 VQ capacity（兼顾内存与 latency）系统性领先 NSG / HNSW / SCANN / NGT / N2。[chen-2021-spann §4.2 + Fig 6-8]

## 实验设置

- **硬件**：Ubuntu 16.04，2× Intel Xeon 8171M（2.6 GHz, 52 cores），128 GB RAM，**2.6 TB SSD RAID-0**
- **数据集**：
  - SIFT1B：1B × 128-d byte vectors
  - DEEP1B：1B × 96-d float vectors
  - SPACEV1B：1B × 100-d byte vectors（Microsoft Bing 真实生产数据，O-UDA license）
  - SIFT1M：1M × 128-d float vectors（in-memory baseline 用）
- **共同条件**：
  - SPANN 与 DiskANN 都用 ~32 GB（SIFT1B / SPACEV1B）/ ~60 GB（DEEP1B）内存
  - SPANN 参数：8 closure replicas，max posting 12 KB / 48 KB，ε₁=10.0，ε₂=6.0/7.0
  - DiskANN 用论文 [subramanya-2019-diskann] 默认参数

[chen-2021-spann §4.1]

## 结果

### SPANN vs DiskANN（Fig 6, 三个 billion 数据集）

| 数据集 | recall@1 to reach | DiskANN latency | SPANN latency | SPANN 优势 |
|---|---|---|---|---|
| SIFT1B | 95% | ~2 ms | <1 ms | **2× 更快** |
| SIFT1B | 95% recall@10 | ~3.5 ms | <1.5 ms | **2× 更快** |
| SPACEV1B | 90% | **DiskANN 在 <4 ms 内不达标** | ~1 ms | DiskANN 失败 |
| DEEP1B | 90% | ~3 ms | ~1.5 ms | **2× 更快** |

观察：

- SPANN 在低 latency budget（<4 ms）尤其领先
- 高 latency budget（>5 ms）两者收敛
- SPACEV1B（Microsoft 真实数据）SPANN 的优势最显著

### SPANN vs in-memory algorithms (SIFT1M, Fig 7-8)

对手：[NSG](../concepts/nsg.md), [HNSW (nmslib)](../concepts/hnsw.md), [SCANN](../concepts/scann.md), NGT-PANNG, NGT-ONNG, N2

- **Latency**：SPANN 高于全内存算法（因为 SSD 访问）
- **VQ capacity**：SPANN **全程领先**

VQ = vectors × QPS / memory cost，衡量"用同样内存 + 算力，能服务多少数据 + 多少查询"。

> SPANN 在 large scale 场景下虽延迟略高，但由于内存预算极小，**单位资源服务能力**最强。

[chen-2021-spann §4.2.2]

### Ablation: 中心数（Fig 9, SIFT1M）

| 中心数（% of N） | recall@1 latency 关系 |
|---|---|
| 4% | 慢 |
| 9% | 中 |
| **16%** | **接近最优** |
| 20% | 不再提升 |

结论：16% 是 sweet spot，更多中心不再提升。

### Ablation: 中心选择算法（Fig 10）

| 算法 | recall@1 / latency |
|---|---|
| Random | 最差 |
| Hierarchical KMeans (HC) | 中 |
| **Hierarchical Balanced Clustering (HBC)** | **最优** |

HBC 把 1M 点切成 160k 簇仅需 50 秒（64 线程），整 SPANN 索引 ~2 分钟。

### Ablation: closure replicas（Fig 11）

| Replicas | 结论 |
|---|---|
| 1 | 边界 recall 大幅下降 |
| 4–8 | recall 持续提升 |
| **>8** | **不再提升**，反而占空间 |

### Ablation: query-aware pruning（Fig 12）

带 dynamic pruning：在小 latency budget 下显著降低延迟，recall 几乎不损失。

## 分布式扩展（§4.3）

SPACEV1B 真实生产 query workload 测试（100k 历史 query），32-partition 场景：

| 方法 | 平均 dispatch 机器数 |
|---|---|
| Random partition | 32（全部） |
| Multi-constraint balanced clustering | 9 |
| + Closure assignment | 8 |
| **+ Query-aware pruning（完整 SPANN）** | **6.3** |

**省 80.3% IO/计算成本**。[chen-2021-spann §4.3 + Fig 13-14]

## 可信度评估

- **实验设计**：作者作为 [SPANN](../systems/spann.md) 提出方，但 [DiskANN](../systems/diskann.md) 用作者发布的预构建索引（DEEP1B）+ 论文默认参数（SIFT1B / SPACEV1B）。SPACEV1B 是 Microsoft 自家数据集 —— SPANN 在此数据上优势最大有"主场优势"嫌疑
- **潜在偏向**：
  1. 与 DiskANN 比时 SPANN 用了 closure replicas + query-aware pruning 全套优化，DiskANN 用默认参数；公平度有限
  2. 没有展示 latency budget > 6 ms 时的曲线（DiskANN 可能在更宽松约束下逆转）
  3. recall 曲线呈"楼梯状"（partial search 自然属性），不像 DiskANN 平滑 —— 可能在某些精度区间反而 DiskANN 更优
- **复现难度**：中。SPTAG 代码开源（[microsoft/SPTAG](https://github.com/microsoft/SPTAG)）；SIFT1B / DEEP1B / SPACEV1B 公开
- **场景局限**：
  - L2-NN（不是 [MIPS](../topics/mips-vs-l2-nn.md)）；MIPS 任务下未验证
  - 维度 ≤ 128；高维 deep embedding 未测
  - 单 query latency 优先；论文未测 batched search

## 与其他大规模部署对比

| 部署 | 规模 | 索引类型 | 介质 | 90% recall 延迟 | source |
|---|---|---|---|---|---|
| Faiss-GPU SIFT1B | 1B | IVFPQ + GPU | HBM | <0.1 ms | [johnson-2017-faiss-gpu] |
| NSG @ Taobao | 2B | NSG 分布式 | DRAM × 32 机 | ~5 ms | [fu-2017-nsg] |
| Faiss trillion @ Meta | 1.5T | SQ6 + HNSW coarse | mmap | ~1 s | [douze-2024-faiss-library §7.1] |
| DiskANN @ z840 | 1B | Vamana + PQ + SSD | DRAM + NVMe | ~3-5 ms | [subramanya-2019-diskann] |
| **SPANN @ Bing** | **1B+ (实测) / 千亿+ (生产)** | **HBC + closure + IVF + SSD** | **DRAM + NVMe** | **~1 ms** | **本论文** |

SPANN 在 SSD 路线上达到与 GPU 路线相近的延迟（数毫秒级），但内存预算只占 ~10%。

Cited by: [queries/giga-scale-sharding.md](../queries/giga-scale-sharding.md)
