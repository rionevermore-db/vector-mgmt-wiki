---
title: Faiss-GPU on SIFT1B / DEEP1B / YFCC100M
type: benchmark
sources: [johnson-2017-faiss-gpu]
related: [../concepts/warpselect.md, ../concepts/product-quantization.md, ../topics/gpu-vs-cpu-ann.md]
created: 2026-05-07
updated: 2026-05-07
---

# Faiss-GPU on SIFT1B / DEEP1B / YFCC100M

**TL;DR**: Faiss-GPU 论文 §6 的标志性结果。SIFT1B 上单 Titan X 比前作 8.5× 快；DEEP1B 上 4 GPU 替代了原论文 128 CPU 服务器的部署；YFCC100M 95M 图 k-NN graph 构造 35 分钟达 0.8 准确度，比 NN-Descent 集群快两个数量级。[johnson-2017-faiss-gpu §6]

## 实验设置

- **硬件**：4× Maxwell Titan X，2× 2.8 GHz Intel Xeon E5-2680v2，CUDA 8.0
- **k-selection 对手**：fgknn (Tang 2015), TBiS (Sismanis 2012)
- **k-means 对手**：BIDMach
- **ANN 对手**：Wieschollek 2016（同硬件用 binary codes）

## 结果

### k-selection（Fig 3, Titan X）

输入：n_q=10000 query，array length ℓ ∈ [1024, 65536]。

| | k=100 | k=1000 |
|---|---|---|
| WarpSelect vs fgknn | **1.62× faster** | **2.01× faster** |
| WarpSelect vs TBiS | TBiS 大 ℓ 更慢 | TBiS 限制 ℓ ≤ 48000 |
| WarpSelect 峰值带宽 | **55%** | 16% |

观察：WarpSelect 在大 k 下与 peak 差距扩大（因 register pressure 与 warp queue 操作开销）。

### k-means on MNIST8m（Table 1）

8.1M 28×28 图像 → 784-d 向量，20 次迭代：

| 方法 | # GPUs | 256 centroids | 4096 centroids |
|---|---|---|---|
| BIDMach | 1 | 320 s | 735 s |
| **Faiss-GPU** | 1 | **140 s** | 316 s |
| **Faiss-GPU** | 4 | **84 s** | **100 s** |

观察：单卡比 BIDMach 2× 快；4 卡接近线性 speedup（256 centroids 1.66× from 1→4 卡，4096 centroids 3.16×；problem size 越大越线性）。

### 精确 k-NN search on SIFT1M（§6.3）

- 距离矩阵 GEMM 跑 1.28 TFLOP，<1s 完成
- WarpSelect kernel 达 **85% 峰值带宽**（在距离矩阵 D' 上）
- 没有 fused L2 / k-selection kernel 的话慢 25%

### ANN on SIFT1B（§6.4，**最标志性**）

- 单 Titan X，PQ m=8 byte/vector，n_q=10⁴
- **WarpSelect: R@10 = 0.376 @ 17.7 μs/query**
- vs Wieschollek 2016（同硬件，binary codes）：R@10 = 0.35 @ 150 μs/query
- → **8.5× 加速且更准**

### ANN on DEEP1B（§6.4）

- 1B vectors, ℓ=10⁹，PQ m=20 + OPQ → d=80
- 4 GPUs (S=2, R=2)：**R@1 = 0.4517 @ 13.3 ms/vector**
- 对比：原 DEEP1B 论文用 128 CPU 服务器、108.7 小时构图
- 4 Titan X 单机 ≈ 128 服务器集群同等吞吐 → **数量级硬件压缩**

### k-NN graph 构造（§6.5）

| 数据集 | 规模 | 硬件 | 时间 | 质量 |
|---|---|---|---|---|
| YFCC100M | 95M × 128-d | 4 Titan X (S=1, R=4) | **35 min** | 10-intersection > 0.8 |
| YFCC100M | 95M × 128-d | 8 M40 (m=20, S=2, R=4) | ~24 hours | 0.6+ |
| DEEP1B（lower quality） | 1B × 80-d | 4 Titan X | **6 hours** | ~0.5 |
| DEEP1B（higher quality） | 1B × 80-d | 4 Titan X | **~12 hours** | ~0.7 |

参照：

- 同等 YFCC 规模用 NN-Descent on 128-CPU server cluster 报告 108.7 小时（且只 36.5M × 384-d，规模小得多）；
- DEEP1B 1B 的 brute-force k-NN graph 用 32 Tesla C2050 + GEMM 估算需 200 天 —— Faiss 用 IVFADC + WarpSelect 把它压到 12 小时。

## 可信度评估

- **实验设计**：作者作为 Faiss 提出方，但对手算法（fgknn, TBiS, Wieschollek, BIDMach）均开源、参数取自原作；硬件统一控制。
- **潜在偏向**：
  1. 仅与 GPU baseline 对比，未对比同期 CPU SoTA（如 [HNSW](../concepts/hnsw.md) + nmslib）。CPU 在 SIFT1B 量级也有解（Faiss-CPU IVFPQ 即可）。
  2. SIFT1B m=8 byte/vector 的极小内存配置选择对 [PQ](../concepts/product-quantization.md) 友好，graph 算法在该配置下 OOM。
  3. 当时 GPU 是 Maxwell（Titan X 2014 / M40 2015），与现代 H100 / B200 的 register file 大小、HBM 带宽差距巨大；推断到现代硬件需谨慎。
- **复现难度**：低。Faiss 完全开源（[facebookresearch/faiss](https://github.com/facebookresearch/faiss)）；BIGANN / SIFT / DEEP1B 数据集公开。
- **场景局限**：
  - L2-NN（不是 MIPS）；MIPS 任务的 GPU 化未在本论文涵盖（[ScaNN](../concepts/scann.md) 后续工作）。
  - 距离矩阵需装进 GPU 内存；query batch 大于 GPU memory 时需 tiling，§5.1 简述但未细测。
