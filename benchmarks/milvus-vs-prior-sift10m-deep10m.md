---
title: Milvus vs SPTAG / Vearch / 3 商业系统 (SIFT10M / Deep10M, SIFT1B)
type: benchmark
sources: [wang-2021-milvus]
related: [../systems/milvus.md, ../systems/faiss.md, ../concepts/hnsw.md, ../concepts/product-quantization.md, ../topics/attribute-filtering.md, ../topics/multi-vector-queries.md]
created: 2026-05-07
updated: 2026-05-07
---

# Milvus vs SPTAG / Vearch / 3 商业系统

**TL;DR**: Milvus 论文 §7 全套 system-level evaluation。在 SIFT10M / Deep10M（IVF_FLAT 与 HNSW）下 Milvus 比 Vearch 快 6.4×–73.9×、比商业 System B/C 快 1.3×–153.7×、比 SPTAG（树）快 1.3×–2.1×；HNSW 变体最快达 73.9×。SIFT1B 单节点（1.5 TB RAM）+ 12 节点近线性扩展。Cache-aware 2.7× / SIMD AVX512 1.5× / SQ8H 在 GPU 装不下场景下系统胜 pure CPU/GPU。

## 实验设置

- **平台**：Alibaba Cloud
  - CPU 节点：ecs.g6e.13xlarge / ecs.re6.26xlarge
    - 默认：ecs.g6e.6xlarge（Xeon Platinum 8269 Cascade 2.5GHz, 16 vCPUs, 35.75 MB L3, AVX512, 64 GB RAM）
    - SIFT1B single-node：ecs.re6.26xlarge（**104 vCPU, 1.5 TB RAM**）
    - SIFT1B 12-node：ecs.g6e.13xlarge（52 vCPU, 192 GB RAM）
  - GPU 节点：ecs.gn6i-c16g1.4xlarge（NVIDIA Tesla T4, **64 KB private mem, 16 GB global**, PCIe 3.0 16x）
- **数据集**：
  - SIFT10M / Deep10M（前 10M 向量；prior systems 太慢扛不住 1B）
  - SIFT1B / Deep1B（用于 scalability，IVF_FLAT only）
  - Recipe1M（multi-vector 评估）
- **对手**：
  - 开源：[Jingdong Vearch](https://github.com/vearch/vearch)（v3.2.0，2020-07）+ [Microsoft SPTAG](https://github.com/microsoft/SPTAG)（[wang-2021-milvus ref 14]）
  - 商业：System A / B / C（匿名，最新 2020-07 版本）
- **指标**：
  - **Recall**：top-50 默认；ground truth top-k 求交集比例
  - **Throughput**：10000 random queries
  - 单节点对比：Milvus / SPTAG / 商业 A/C 用 2 节点（64 GB/节点）；System B 用 4 节点（128 GB/节点）

[wang-2021-milvus §7.1]

## 主结果

### IVF index（SIFT10M / Deep10M, Fig 8）

定性顺位（相同 recall 下 throughput）：

| 排名 | 系统 | 相对 Milvus_IVF_FLAT 倍速 |
|---|---|---|
| 1 | **Milvus_GPU_SQ8H** | 最快（GPU acceleration） |
| 2 | Milvus_IVF_FLAT | 1× baseline |
| 3 | Milvus_IVF_SQ8 | 略慢 baseline（量化损失） |
| 4 | Milvus_IVF_PQ | 中速 |
| 5 | SPTAG | 1.3×–2.1× slower |
| 6 | Vearch | **6.4×–27× slower** |
| 7 | System B (4 nodes) | **153.7× slower than Milvus** |
| 8 | System C | 4.7×–11.5× slower |

注意：System B 的 brute-force 数据点是参数没调（n_probe / nlist），团队在 2020-08 提交 bug report；论文承认调参后 System B 可能更接近，留作 future work。

[wang-2021-milvus Fig 8, footnote 11-13]

### HNSW index（SIFT10M / Deep10M, Fig 9）

| | 相对优势 |
|---|---|
| Milvus_HNSW vs Vearch | **15.1×–60.4× faster** |
| Milvus_HNSW vs System A | 8.0×–17.1× faster |
| Milvus_HNSW vs System C | **7.3×–73.9× faster** |

Deep10M 上 System A 不参评（不支持 inner product）；System C 在 Deep10M 上构建超 100 小时未完成。

### Scalability（SIFT1B IVF_FLAT, Fig 10）

| 配置 | 数据 | 行为 |
|---|---|---|
| Single ecs.re6.26xlarge（104 vCPU, 1.5 TB RAM） | SIFT1B 全装内存 | throughput 随 data size 增加而 graceful 下降（线性） |
| 12 节点 ecs.g6e.13xlarge（52 vCPU, 192 GB/节点） | 分片 | 4-12 节点 throughput **near-linear scaling** |

> 论文 §7.3 注：g6e.13xlarge 单节点 throughput 高于 re6.26xlarge **因为多核共享 L3 与内存带宽竞争**。这一观察反向支持 Milvus 的 cache-aware 设计——内存带宽是瓶颈，不是 vCPU 数。

## Optimization-level 实验

### Cache-aware design（Fig 11）

| L3 size | Cache-aware speedup vs original |
|---|---|
| 12 MB（i7-8700） | **up to 2.7×** |
| 35.75 MB（Xeon Platinum 8269） | 1.5× |

更大 L3 的速比更小——因为 original 设计在大 L3 上本就少 miss。Cache-aware 在 commodity CPU（小 L3）上收益最大。

### SIMD（Fig 12）

| | 相对 |
|---|---|
| AVX512 vs AVX2 | **1.5× faster** |

支持 AVX512 仅在最近 Xeon Cascade Lake / 部分桌面 CPU 上有；老 Skylake / 桌面 i7 都只到 AVX2。Milvus runtime hooking 让同 binary 跨 CPU 都最优。

### GPU SQ8H（Fig 13）

SIFT1B（GPU 16 GB 装不下全数据）下：

| | 相对 SQ8H |
|---|---|
| Pure CPU SQ8 | 系统性 slower |
| Pure GPU SQ8 | 系统性 slower（PCIe transfer 瓶颈） |
| **SQ8H（hybrid）** | **始终最快**，且随 query batch size 增长差距扩大 |

关键洞见：query batch ≥ 1000 时，**纯 GPU 慢于纯 CPU** —— PCIe demand-loading 把 GPU 计算优势抹平。SQ8H 是绕开此 trap 的工程解。

### Attribute Filtering 五策略（Fig 14-15）

[详见 topics/attribute-filtering.md](../topics/attribute-filtering.md)

| query selectivity | 最优策略 |
|---|---|
| 高 selectivity（C_A 极严，候选少） | A 最优 |
| 中 selectivity | E（Milvus partition-based）总最优 |
| 低 selectivity（C_A 不严） | E 仍然最优 |

| | 速度 |
|---|---|
| Milvus E vs D（cost-based AnalyticDB-V） | **up to 13.7× faster** |
| Milvus E vs Vearch / 商业 ABC | **48.5×–41299.5× faster** |

48× 至 41,000× 的范围—系统差异巨大。最大那档商业系统几乎没做 vector + attr 联合优化。

### Multi-Vector Query（Fig 16, Recipe1M）

[详见 topics/multi-vector-queries.md](../topics/multi-vector-queries.md)

| 相似度 | 最优算法 | 备注 |
|---|---|---|
| Inner product | **Vector fusion** 比 iterative merging 3.4×–5.8× faster | 单次 ANN 即可 |
| Euclidean | **Iterative merging k'=4096** 比 NRA-2048 15× faster | NRA 退化场景 |

## 可信度评估

- **实验设计**：Milvus 团队为论文作者，但对手系统全部用 latest 2020-07 公开版本；硬件统一（Alibaba Cloud）；recall 用 ground truth 实测
- **潜在偏向**：
  1. **System B 没调参**——论文 footnote 11 自己承认 "single data point because parameter tuning was disabled"。157× 那档可能调参后大幅缩小
  2. **System A 不支持 inner product** → Deep10M 上对比缺；商业系统能力不全
  3. **不与 Faiss 直接对比** —— 论文 §7.4 隐含用"Milvus = Faiss + 优化"做内部对照（cache-aware / SIMD / SQ8H ablation），但没在 Fig 8-9 把 vanilla Faiss 列为对手。读者无从知道 "Milvus vs Faiss" 净增多少
  4. **未与 [DiskANN](../systems/diskann.md) / [SPANN](../systems/spann.md) 对比** —— 同年 NeurIPS 论文 SPANN 不在对手列表；DiskANN 也不在。SSD 路线 vs DRAM 路线没正面比
  5. **数据集偏 SIFT 系列** —— 128-d / 96-d；现代 768-d / 1024-d 未测
  6. **最大数据集 SIFT1B** —— 与 Faiss trillion-scale (1.5T) 差三个数量级
- **复现难度**：中。Milvus 完全开源（[milvus-io/milvus](https://github.com/milvus-io/milvus)）；BIGANN / Recipe1M 数据公开；商业 ABC 匿名无法复现
- **场景局限**：
  - L2-NN 与 inner product 都测；MIPS-specific 优化（[ScaNN](../concepts/scann.md) anisotropic）未覆盖
  - 单 query latency 与 batch throughput 都给但侧重 throughput；P99 latency 未拆数字
  - 多租户 / 隔离 / 安全场景未涉及（DBMS 该有的全套）

## 与其他大规模部署对比

| 部署 | 规模 | 系统类型 | 介质 | source |
|---|---|---|---|---|
| Faiss-GPU SIFT1B | 1B | Library + GPU | HBM | [johnson-2017-faiss-gpu] |
| NSG @ Taobao | 2B | NSG 多机分片 | DRAM × 32 机 | [fu-2017-nsg] |
| Faiss trillion @ Meta | 1.5T | Library + 分布式 mmap | mmap | [douze-2024-faiss-library §7.1] |
| DiskANN @ z840 | 1B | 算法系统 | DRAM + NVMe | [subramanya-2019-diskann] |
| SPANN @ Bing | 1B+ / 千亿+ | 算法系统 | DRAM + NVMe | [chen-2021-spann] |
| **Milvus @ 数百组织** | **production scale 未公开** | **DBMS** | DRAM 或 + S3/HDFS | **本论文** |

Milvus 是 wiki 已有 6 个工业部署里**唯一的 DBMS 形态**——其他都是 library / 算法实现 / 单租户应用。
