---
title: DiskANN on SIFT1B（vs IVFOADC+G+P, vs HNSW/NSG in-memory）
type: benchmark
sources: [subramanya-2019-diskann]
related: [../concepts/vamana.md, ../systems/diskann.md, ../concepts/product-quantization.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../topics/disk-vs-memory-ann.md, ./spann-vs-diskann-billion.md]
created: 2026-05-07
updated: 2026-05-07
---

# DiskANN on SIFT1B

**TL;DR**: DiskANN 论文 §4 的全套实验。在 million-scale 内存任务上 [Vamana](../concepts/vamana.md) 击败 [HNSW](../concepts/hnsw.md) / [NSG](../concepts/nsg.md)；在 SIFT1B（10⁹ 点）上 single-shot 索引 1-recall@1 = 98.68% @ <5 ms；merged 版（64 GB RAM 即可构建）仍达 ~97%；同等内存预算下 IVFOADC+G+P-32 plateau 在 62.74%。[subramanya-2019-diskann §4]

## 实验设置

- **硬件**：
  - **z840 工作站**：Xeon E5-2620v4 × 2 (16 核)，64 GB DDR4 RAM，2× Samsung 960 EVO 1 TB SSD（RAID-0）
  - **M64-32ms 虚拟机**：Xeon E7-8890v3 × 2 (32 vCPU)，**1792 GB** DDR3 RAM（仅用于 single-shot 1B 索引构建）
- **数据集**：
  - SIFT1M / GIST1M / DEEP1M（in-memory comparisons）
  - SIFT1B（[BIGANN](http://corpus-texmex.irisa.fr/)，1B × 128-d uint8）
  - DEEP1B（1B × 96-d float CNN feature）
- **对手**：
  - In-memory：[HNSW](../concepts/hnsw.md) [malkov-2016-hnsw], [NSG](../concepts/nsg.md) [fu-2017-nsg]
  - SSD / 量化：FAISS [johnson-2017-faiss-gpu], IVFOADC+G+P [Baranchuk 2018]

## 结果

### In-memory，million-scale（[Fig 3]）

100-recall@100 vs latency 在 SIFT1M / GIST1M / DEEP1M 上：

> **Vamana 曲线全程在 HNSW 与 NSG 之上**（recall 越高，差距越明显）。

DEEP1M 索引构建时间：

- Vamana：**149 s**
- HNSW：219 s
- NSG：480 s（含 EFANNA kNN graph 构造时间）

[subramanya-2019-diskann §4.1]

### Hop 数 vs Max degree（[Fig 2c]，SIFT1M, recall 98% 5@5）

> Vamana 的平均 hop 数随 max degree **单调下降**；HNSW / NSG 平稳。

意义：Vamana 能"用密度换 hop 数"，对 SSD 部署是关键（每 hop = SSD 读）。

### SIFT1B Single vs Merged（[Fig 2a]）

| 索引 | 1-recall@1 | Mean latency | 索引大小 | 构建资源 |
|---|---|---|---|---|
| **DiskANN single (R128)** | **98.68%** | **<5 ms** | ~300 GB | M64-32ms (1100 GB peak), ~2 days |
| DiskANN merged (R128) | ~97% | ~6 ms（20% 慢） | 348 GB | z840 (64 GB peak), ~5 days |
| IVFOADC+G+P-32 | **plateau at 62.74%** | <5 ms | 同 DiskANN merged | n/a |
| IVFOADC+G+P-16 | plateau at 37.04% | <5 ms | 一半 DiskANN | n/a |

观察：

- **DiskANN single 在 SIFT1B 上达成 1-recall@1 接近 100%**（98.68%）
- **Merged 比 single 慢 ~20%** 但内存可控（64 GB peak vs 1100 GB peak）
- **IVFOADC+G+P 量化失真天花板**：再增 nprobe 也只能扫到更多 PQ codes，recall plateau
- 同内存下 DiskANN > IVFOADC+G+P **35+ 个百分点**

[subramanya-2019-diskann §4.3-4.4]

### DEEP1B 单查询延迟（[Fig 2b]）

DiskANN merged on DEEP1B：1-recall@1 vs latency 曲线，>95% recall @ <8 ms。

### 16-thread 吞吐

> >5000 QPS @ <3 ms mean latency @ 95%+ 1-recall@1 on SIFT1B（z840 16 cores）

## 内存预算分析（[§3.1]）

SIFT1B m=8 byte/vec 的 IVFOADC+G+P-32 vs DiskANN merged：

- 两者**索引磁盘占用相当**（IVFOADC 把所有数据 codes 压到 RAM；DiskANN merged 把 PQ codes 放 RAM、graph + 全精度放 SSD）
- IVFOADC：所有 1B PQ codes 占 ~32 GB RAM
- DiskANN merged：1B PQ codes（32 byte/vec）= 32 GB RAM + ~316 GB SSD（graph + 全精度向量）

**关键**：DiskANN 的 64 GB 单机预算是 z840 的实际峰值，不是 PQ codes 本身那 32 GB。差额给 vertex cache、beam search state、OS。

## 可信度评估

- **实验设计**：作者作为 [DiskANN](../systems/diskann.md) 提出方，但对手算法（FAISS、IVFOADC+G+P）参数取自 Baranchuk 2018 报告值；硬件控制（同 z840）。
- **潜在偏向**：
  1. SIFT1B single 索引在 1100 GB RAM 服务器上构建（M64-32ms 是 Azure 虚拟机），不是论文标榜的"64 GB 单机"；single vs merged 的对比 tilted favorable
  2. Beam width W、PQ m 等参数对 DiskANN 有利的方向调；论文未消融这些选择
  3. 与 HNSW 的对比是 Vamana in-memory，没有 HNSW-on-SSD 实验（HNSW 设计本来不是 SSD-friendly，但缺少正面对照）
- **复现难度**：低-中。代码 [microsoft/DiskANN](https://github.com/microsoft/DiskANN) 全开源；SIFT1B 数据公开；硬件需求（64 GB RAM + 2× NVMe SSD）属于工作站级。
- **场景局限**：
  - L2-NN（不是 [MIPS](../topics/mips-vs-l2-nn.md)）；MIPS 任务下 PQ 应换成 [ScaNN](../concepts/scann.md) anisotropic
  - SIFT 描述符（128-d, uint8 编码）；现代 deep embedding（768-d, float32）的 SSD 扇区 layout 不同
  - 未测 batched search（beam search 重在单 query 延迟，不是吞吐）

## 与其他大规模部署对比

| 部署 | 规模 | 索引类型 | 介质 | source |
|---|---|---|---|---|
| Faiss-GPU SIFT1B | 1B | IVFPQ + GPU | HBM | [johnson-2017-faiss-gpu] |
| NSG @ Taobao | 2B | NSG 分布式 | DRAM × 32 机 | [fu-2017-nsg] |
| Faiss trillion @ Meta | 1.5T | SQ6 + HNSW coarse | mmap | [douze-2024-faiss-library §7.1] |
| **DiskANN @ z840** | **1B** | **Vamana + PQ + SSD** | **DRAM + NVMe** | **本论文** |

DiskANN 的差异化：**单机 + 64 GB RAM + 高 recall** 三者同时满足，是 wiki 已有 4 个部署里独一份的组合。
