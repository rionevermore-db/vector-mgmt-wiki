---
title: GPU vs CPU ANN（实现差异与算法设计偏好）
type: topic
sources: [johnson-2017-faiss-gpu, malkov-2016-hnsw, fu-2017-nsg, jegou-2011-pq, guo-2019-scann]
related: [../concepts/warpselect.md, ../concepts/product-quantization.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/scann.md]
created: 2026-05-07
updated: 2026-05-07
---

# GPU vs CPU ANN

**TL;DR**: ANN 算法的最优形态在 CPU 与 GPU 上不一样：CPU 偏好图遍历（HNSW / NSG）的低 distance compute / 高随机访问模式；GPU 偏好 brute-force + fused k-selection 的高 throughput / 顺序 access 模式。Faiss 的 GPU 论文证明 1B 向量在单台 4-GPU 机器上 12 小时构图，挑战了"必须用 graph 才能 scale"的隐含假设。[johnson-2017-faiss-gpu §6]

## 问题陈述

CPU 与 GPU 的硬件特性差异：

| | CPU | GPU |
|---|---|---|
| 算力 | ~1 TFLOPS（AVX-512） | 10+ TFLOPS（FP32） |
| 内存带宽 | ~50 GB/s | ~500 GB/s（HBM） |
| 内存容量 | 数百 GB 单机 | 单卡 16–80 GB |
| 随机访问 cost | 中等（缓存友好时低） | 高（warp divergence / coalescing） |
| 分支预测 | 强 | **弱**（warp 内 32 thread 必须同分支） |
| 适合的算法模式 | irregular pointer chasing | **regular dense compute** |

ANN 算法在两侧的最优形态因此不同。

## 算法在两侧的偏好

| 算法家族 | CPU 上 | GPU 上 |
|---|---|---|
| **Graph ([HNSW](../concepts/hnsw.md) / [NSG](../concepts/nsg.md))** | **首选** —— 距离计算少（log N 跳）、随机访问被 prefetch 缓解 | 不利 —— pointer chasing + warp divergence |
| **Brute force** | 不可行（O(Nd) per query） | **可行** —— GEMM 友好，cuBLAS 跑 85% 峰值带宽 |
| **[PQ](../concepts/product-quantization.md) / IVFADC** | **首选** —— 内存敏感、扫描友好 | 也好 —— lookup-add 是 bandwidth-bound，可 fuse 到 [WarpSelect](../concepts/warpselect.md) |
| **LSH** | 中等 | 一般 —— hash table 随机访问对 GPU 不友好 |

[johnson-2017-faiss-gpu §6.4 vs malkov-2016-hnsw §5; fu-2017-nsg §4.2]

## 反直觉之一：GPU 上 brute force 比 graph 更快

[NSG 论文 §4.2](../concepts/nsg.md) 在 DEEP100M 上说"NSG-16core 比 Faiss-GPU 快"——这是**对的**，前提是数据完全装进 CPU 内存。

但 [johnson-2017-faiss-gpu §6.5] 给出反例：YFCC100M（95M 图，128-d）的 k-NN graph 构造，4 Titan X **brute-force** 35 分钟达 0.8+ accuracy；同等规模用 NN-Descent 在 128-CPU 集群跑 108.7 小时（DEEP1B 论文报告，36.5M × 384-d）。

为什么？

- Graph 算法的优势在"少做距离计算"，但 GPU 的距离计算成本极低（cuBLAS GEMM），节省距离计算的算法收益很小；
- Graph 算法的随机访问劣势在 GPU 上被放大（warp divergence），抵消了"少跳"带来的 wall-clock 节省。

## 反直觉之二：CPU 上 PQ 不一定输给 graph

[malkov-2016-hnsw §5.4] 在 200M SIFT 上 [HNSW](../concepts/hnsw.md) 比 Faiss-CPU 的 IVFPQ 快但内存大 2–3 倍；但同实验在 1B SIFT 上 HNSW 直接 OOM，PQ 仍是唯一选项。

CPU 内存虽大但有上限；十亿级单机 = 必须量化压缩 = [PQ](../concepts/product-quantization.md) 路径不可绕开。

## 工业方案对比

| 方案 | 硬件 | 算法 | 适合规模 |
|---|---|---|---|
| HNSW + nmslib（CPU） | 多核 CPU | graph + heuristic | 1M–100M |
| [NSG](../concepts/nsg.md) + ZJULearning | 多核 CPU | graph + MRNG | 1M–100M（内存内） |
| Faiss-CPU IVFPQ | 多核 CPU | quantization + IVF | 1B+（内存敏感） |
| **Faiss-GPU IVFPQ** [johnson-2017-faiss-gpu] | **单/多 GPU** | quantization + WarpSelect | **1B 单机 / 多 GPU** |
| [ScaNN](../concepts/scann.md) | 多核 CPU + SIMD | anisotropic PQ | MIPS 高 recall |

## 关键 GPU 优化技术（来自 Faiss-GPU）

1. **Kernel fusion**：距离计算 + k-selection 同 kernel，避免中间矩阵写回 → 节省 25% 时间。[johnson-2017-faiss-gpu §5.1]
2. **State in registers**：[WarpSelect](../concepts/warpselect.md) 的核心，绕开 shared memory 瓶颈。
3. **Lookup tables in shared memory**：PQ 的 256 个 centroid 距离表放 shared memory（每 query 算一次）。[johnson-2017-faiss-gpu §5.2]
4. **Replication for throughput / Sharding for capacity**：multi-GPU 策略。[johnson-2017-faiss-gpu §5.4]

## Open Questions

- **Graph methods 能否 GPU 化？** 论文未尝试。后续工作（GGNN、CAGRA、SONG）专门做 GPU graph ANN，但 wiki 尚未 ingest。
- **量化 codebook 训练 GPU 化的极限**：本论文用 GPU 做 k-means；OPQ / [ScaNN](../concepts/scann.md) 的更复杂 loss 在 GPU 上的优化未有系统研究。
- **AI 加速器（TPU、Trainium、NPU）上的 ANN**：与 GPU 的差异未在 wiki 任何 source 覆盖。
- **CPU SIMD（AVX-512、SVE2）的天花板**：单 socket CPU 用极致 SIMD 能否追上单 GPU？wiki 未覆盖。
