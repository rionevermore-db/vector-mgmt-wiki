---
title: GPU vs CPU ANN（实现差异与算法设计偏好）
type: topic
sources: [johnson-2017-faiss-gpu, malkov-2016-hnsw, fu-2017-nsg, jegou-2011-pq, guo-2019-scann, ootomo-2023-cagra]
related: [../concepts/warpselect.md, ../concepts/product-quantization.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/scann.md, ../concepts/cagra-graph.md, ../concepts/vamana.md, ../systems/cagra.md, ../systems/faiss.md, ../systems/milvus.md, ../benchmarks/faiss-gpu-sift1b-deep1b.md, ../benchmarks/cagra-vs-hnsw-ggnn-ganns.md]
created: 2026-05-07
updated: 2026-05-09 (CAGRA — graph-on-GPU now solved)
---

# GPU vs CPU ANN

**TL;DR**: ANN 算法的最优形态在 CPU 与 GPU 上不一样。**CPU 偏好图遍历**（HNSW / NSG / Vamana 的低 distance compute / 高随机访问模式）+ **量化压缩**（PQ 减内存）；**GPU 偏好高并行计算**（brute-force GEMM / fused kernel / WarpSelect k-selection）。**关键状态变更（2026-05-09）**：之前 wiki 视 graph-on-GPU 为开放问题（"Graph methods 能否 GPU 化？" 是 Open Q）；[CAGRA](../systems/cagra.md) [ootomo-2023-cagra] **解决这个 gap**——首个 GPU-native graph-based ANN，build 比 HNSW (CPU 64-core) 2.2-27× 快，large-batch search 33-77× 快，single-query 3.4-53× 快。**CAGRA 设计哲学**：fixed out-degree + non-hierarchical + rank-based reordering——algorithm 与 hardware 同步演进。**GPU ANN landscape 现在双路径**：IVF + PQ + WarpSelect（Faiss-GPU 2017）vs graph-based CAGRA（NVIDIA 2024）。

## 问题陈述

CPU 与 GPU 的硬件特性差异：

| | CPU | GPU |
|---|---|---|
| 算力 | ~1 TFLOPS（AVX-512） | 10+ TFLOPS（FP32），300+ TFLOPS（FP16/Tensor Core） |
| 内存带宽 | ~50 GB/s | ~500 GB/s（HBM）, A100 ~1.5 TB/s |
| 内存容量 | 数百 GB 单机 | 单卡 16–80 GB |
| 随机访问 cost | 中等（缓存友好时低） | 高（warp divergence / coalescing） |
| 分支预测 | 强 | **弱**（warp 内 32 thread 必须同分支） |
| 适合的算法模式 | irregular pointer chasing | **regular dense compute + fixed-pattern access** |

ANN 算法在两侧的最优形态因此不同——但**CAGRA 证明 graph 算法可以重新设计为 GPU-native**，不再绑死 CPU 偏好。

## 算法家族在两侧的偏好（updated）

| 算法家族 | CPU 上 | GPU 上 |
|---|---|---|
| **Variable-degree Graph ([HNSW](../concepts/hnsw.md) / [NSG](../concepts/nsg.md) / [Vamana](../concepts/vamana.md))** | **首选** —— 距离计算少（log N 跳）、随机访问被 prefetch 缓解 | **不友好** —— variable degree → warp divergence + load imbalance |
| **Fixed-degree Graph ([CAGRA](../concepts/cagra-graph.md))** | 可行但单线程慢（更多 distance comp） | **首选** —— uniform parallelism + 无 warp divergence + non-hierarchical |
| **Brute force** | 不可行（O(Nd) per query） | **可行** —— GEMM 友好，cuBLAS 85% 峰值带宽 |
| **[PQ](../concepts/product-quantization.md) / IVFADC** | **首选** —— 内存敏感、扫描友好 | 也好 —— lookup-add 是 bandwidth-bound，可 fuse 到 [WarpSelect](../concepts/warpselect.md) |
| **LSH** | 中等 | 一般 —— hash table 随机访问对 GPU 不友好 |

[johnson-2017-faiss-gpu §6.4 vs malkov-2016-hnsw §5; fu-2017-nsg §4.2; ootomo-2023-cagra §V]

## GPU graph ANN 的设计演化

[per concepts/cagra-graph.md + ootomo-2023-cagra §II-C]

GPU graph-based ANN 经历四代：

| 代 | 系统 | 论文 | 设计哲学 |
|---|---|---|---|
| 1 (CPU 移植) | SONG | [Zhao 2021] | NSW + open hash table + bounded queue + dynamic alloc → 10-180× over single-thread CPU |
| 2 (优化数据结构) | GGNN | [Groh 2022] | k-NN graph + GPU-friendly data structures + fast construction |
| 3 (扩展算法) | GANNS | [Yu 2022] | NSW + HNSW + k-NN tailored data structures |
| **4 (GPU-native)** | **CAGRA** | **[Ootomo 2023]** | **从 first principles 为 GPU 设计 graph 结构 + search algorithm** |

**1-3 代都是"adapt CPU graph to GPU"**——受限于 variable degree / hierarchy 的 CPU-friendly 假设。**CAGRA 是第 4 代——重新设计 graph 让 algorithm 与 hardware 同步演进**。

实测 [Fig 11]: CAGRA build vs GGNN/GANNS = **1.0-31× 快**——验证 first-principles design 的工程收益。

## 反直觉之一（已 outdated）：GPU 上 brute force 比 graph 更快

[NSG 论文 §4.2](../concepts/nsg.md) 在 DEEP100M 上 "NSG-16core 比 Faiss-GPU 快"——这是 **2017-2019 年代的真实但已 outdated**。

[johnson-2017-faiss-gpu §6.5] YFCC100M（95M 图，128-d）的 k-NN graph 构造，4 Titan X **brute-force** 35 分钟达 0.8+ accuracy；同等规模用 NN-Descent 在 128-CPU 集群跑 108.7 小时。

→ 当时论点："Graph 算法的优势在'少做距离计算'，但 GPU 的距离计算成本极低（cuBLAS GEMM），节省距离计算的算法收益很小；Graph 算法的随机访问劣势在 GPU 上被放大（warp divergence），抵消了'少跳'带来的 wall-clock 节省"。

**2024 年 CAGRA 改变 framing**：CAGRA fixed out-degree + warp splitting + forgettable hash table → graph 在 GPU 上**也可以**比 brute force 快。

| Workload | 2017 NSG/Faiss 时代 | **2024 CAGRA 时代** |
|---|---|---|
| GPU + 100M build | brute force 快 | **CAGRA 快**（rank-based reordering + GPU-native graph） |
| GPU + large-batch search | Faiss-GPU IVFPQ 主导 | **CAGRA 互补**（graph search 高 recall + Faiss-GPU 内存效率） |
| GPU + single-query | Faiss-GPU 不友好 | **CAGRA multi-CTA mode 友好** |

## 反直觉之二：CPU 上 PQ 不一定输给 graph

[malkov-2016-hnsw §5.4] 在 200M SIFT 上 [HNSW](../concepts/hnsw.md) 比 Faiss-CPU 的 IVFPQ 快但内存大 2–3 倍；但同实验在 1B SIFT 上 HNSW 直接 OOM，PQ 仍是唯一选项。

CPU 内存虽大但有上限；十亿级单机 = 必须量化压缩 = [PQ](../concepts/product-quantization.md) 路径不可绕开。

## 工业方案对比（updated 2026-05-09）

| 方案 | 硬件 | 算法 | 适合规模 |
|---|---|---|---|
| HNSW + nmslib（CPU） | 多核 CPU | graph + heuristic (variable degree) | 1M–100M |
| [NSG](../concepts/nsg.md) + ZJULearning | 多核 CPU | graph + MRNG | 1M–100M（内存内） |
| [Vamana](../concepts/vamana.md) / DiskANN | CPU + SSD | graph + α-controlled prune | 1B (with PQ + SSD rerank) |
| Faiss-CPU IVFPQ | 多核 CPU | quantization + IVF | 1B+（内存敏感） |
| **Faiss-GPU IVFPQ** [johnson-2017] | 单/多 GPU | quantization + WarpSelect | 1B 单机 / 多 GPU |
| [ScaNN](../concepts/scann.md) | 多核 CPU + SIMD | anisotropic PQ | MIPS 高 recall |
| **[CAGRA](../systems/cagra.md) (NEW 2024)** | **NVIDIA GPU** | **graph (fixed degree + non-hierarchical)** | **100M 单 A100 / multi-GPU 1B+** |
| [SONG] / GGNN / GANNS | NVIDIA GPU | graph (CPU 移植) | 100M（被 CAGRA 取代） |

## 关键 GPU 优化技术对照

[per concepts/warpselect.md, concepts/cagra-graph.md "四个 GPU-specific 优化"]

| 优化 | Faiss-GPU IVFADC (2017) | **CAGRA (2024)** |
|---|---|---|
| **Kernel fusion** | 距离计算 + k-selection 同 kernel → -25% time | distance + bitonic sort + graph traversal 同 kernel |
| **State in registers** | [WarpSelect](../concepts/warpselect.md) k-selection 全在 registers | bitonic sort partly in registers |
| **Shared memory hash table** | PQ centroid 距离表（256 entries × 1 KB） | **forgettable hash table for visited nodes (≤4 KB)** |
| **Multi-GPU strategy** | replication (throughput) / sharding (memory) | logical 但论文 §IV-C 仅 1 段 |
| **Warp splitting (NEW from CAGRA)** | n/a | **team size 4-32 自适应 dimensionality** |
| **1-bit flag in node index (NEW from CAGRA)** | n/a | **MSB of node index = parent flag** |
| **Single-CTA + Multi-CTA dual mode (NEW)** | n/a | **runtime dispatch by batch size** |

→ CAGRA 在 Faiss-GPU 优化基础上**显著扩展 GPU graph search 的优化层级**。

## CPU graph + GPU graph 的关系（NEW）

[per concepts/cagra-graph.md "vs HNSW / NSG / Vamana"]

CPU graph 与 GPU graph 不是**取代关系**，是**目标 hardware 不同的双路径**：

| 维度 | CPU Graph (HNSW/NSG/Vamana) | **GPU Graph (CAGRA)** |
|---|---|---|
| Out-degree | variable | **fixed** |
| Hierarchy | multi-layer (HNSW) / single-layer (NSG/Vamana) | **non-hierarchical** |
| Initial entry | top-layer descent / fixed entry | **random sampling p × d** |
| 适合 dataset size | 100M-1B | 1M-100M (单 GPU) |
| 适合 single-query latency | best (CPU) | **CAGRA multi-CTA 也快** |
| 适合 large-batch throughput | 中（CPU 64-core HNSW） | **GPU CAGRA 33-77× 快** |
| Build time on equivalent dataset | reference | **2.2-27× faster (A100 vs 64-core)** |
| Streaming insert/delete | ✓ via [FreshVamana](../concepts/freshvamana.md) | **✗ (open work)** |
| Filter-aware build | ✓ via [FilteredVamana](../concepts/filtered-vamana.md) / [ACORN](../concepts/acorn.md) | **✗ (open work)** |

→ **不同 workload 选不同路径**：CPU graph 适合 streaming + filter-heavy + 极致内存 budget；GPU graph 适合 high-throughput + low-latency + 静态索引 + dataset 装得进 GPU memory。

## 关键 Open Questions（updated 2026-05-09）

- ~~**Graph methods 能否 GPU 化？** 论文未尝试。后续工作（GGNN、CAGRA、SONG）专门做 GPU graph ANN，但 wiki 尚未 ingest。~~ **2026-05-09 ingest [ootomo-2023-cagra] 已解** — CAGRA 是 GPU-native graph 完整工业方案
- **量化 codebook 训练 GPU 化的极限**：Faiss-GPU 用 GPU k-means；OPQ / [ScaNN](../concepts/scann.md) / [RaBitQ](../concepts/rabitq.md) 的更复杂 codebook 在 GPU 上的优化未有系统研究
- **CAGRA + [RaBitQ](../concepts/rabitq.md) on GPU**：理论可叠加，wiki 内 zero coverage（两个论文不同年代不同公司）
- **CAGRA + DiskANN-style hybrid**：超 GPU memory 时 fallback 到 SSD；论文 §V-E 提为 future
- **CAGRA + streaming (FreshCAGRA)**：fixed-degree graph 的 streaming 比 variable-degree 难——open
- **CAGRA + filter / multi-vector / range / Join**：完全 open
- **CAGRA + Faiss 集成**：Faiss 论文 §A.3 提为 emerging direction；当前 Faiss release 不集成 RAFT
- **AI 加速器（TPU、Trainium、NPU）上的 ANN**：与 GPU 的差异未在 wiki 任何 source 覆盖
- **CPU SIMD（AVX-512、SVE2）的天花板**：单 socket CPU 用极致 SIMD 能否追上单 GPU？wiki 未覆盖
- **GPU CAGRA vs CPU HNSW 的"经济阈值"**：何时 GPU 投资划算？workload-specific，论文未深入
