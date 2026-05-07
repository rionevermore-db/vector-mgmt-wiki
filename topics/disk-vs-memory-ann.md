---
title: Disk vs Memory ANN（SSD 与 DRAM 的 ANN 路线）
type: topic
sources: [subramanya-2019-diskann, jegou-2011-pq, malkov-2016-hnsw, fu-2017-nsg, douze-2024-faiss-library]
related: [../concepts/vamana.md, ../concepts/product-quantization.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../systems/diskann.md, ../systems/faiss.md, ../benchmarks/diskann-sift1b.md, ../benchmarks/faiss-trillion-scale.md]
created: 2026-05-07
updated: 2026-05-07
---

# Disk vs Memory ANN

**TL;DR**: 十亿+ 向量索引能否单机部署？传统答案是"必须 PQ 压缩到全内存"（[Faiss IVFPQ](../systems/faiss.md) 路径），代价是召回卡在 60-70%。[DiskANN](../systems/diskann.md) 给出新答案：**graph + SSD 全精度 re-rank**，64 GB RAM + SSD 即可达 95%+ 召回。两条路线对硬件、数据规模、recall 上限的取舍不同。[subramanya-2019-diskann §1 + §4.4]

## 问题陈述

ANN 索引在不同存储层级的代价差异巨大：

| 层级 | 容量（典型单机） | 随机访问延迟 | 带宽 |
|---|---|---|---|
| L1 / L2 cache | KB–MB | 1–10 ns | 1+ TB/s |
| DRAM | 数百 GB | 100 ns | 50–500 GB/s |
| **NVMe SSD** | **数 TB** | **几百 μs** | 3–7 GB/s |
| HDD | 10 TB+ | 10 ms | 100 MB/s |
| Network | ∞ | 1+ ms | 10–100 GB/s |

ANN 的搜索过程涉及大量随机访问（图节点跳转 / 倒排表扫描）。**算法是否对随机访问延迟敏感**决定了它能否下放到 SSD。

## 工业方案对比

| 路线 | 代表 | 数据驻留 | 1B SIFT recall @ <5ms | 单机内存预算 |
|---|---|---|---|---|
| **全内存 graph** | [HNSW](../concepts/hnsw.md), [NSG](../concepts/nsg.md) | DRAM | 1B 直接 OOM | TB 级 |
| **量化压缩 + 全内存** | [Faiss IVFPQ](../systems/faiss.md) | DRAM | ~62%（IVFOADC+G+P-32 plateau） | 数十 GB |
| **多机分片** | NSG @ Taobao（32 partition × 1/32 数据） | 分布式 DRAM | ~98% | N 台机器 × 数十 GB |
| **GPU brute force** | [Faiss-GPU](../systems/faiss.md) | HBM | 高 | 单卡 ~32 GB |
| **磁盘 + 量化导航 + SSD re-rank** | [DiskANN](../systems/diskann.md) | DRAM (PQ) + SSD (graph + full vec) | **98.68%** | 64 GB |

[subramanya-2019-diskann §1, §4.4]; [douze-2024-faiss-library §5.5 Fig 8]

## 关键洞见 1：算法对随机访问延迟的敏感度

**Graph 算法**的搜索成本 ≈ hops × per-hop time

- per-hop time ∈ DRAM = ~100 ns
- per-hop time ∈ SSD = ~几百 μs（**3-4 个数量级慢**）

朴素把 [HNSW](../concepts/hnsw.md) / [NSG](../concepts/nsg.md) 搬到 SSD 会让 latency 从毫秒级炸到秒级。所以算法必须做两件事之一：

1. **减少 hops**：[Vamana](../concepts/vamana.md) 通过 α>1 + 长程边把 hop 数降 2-3×
2. **批量化 hops**：[DiskANN](../systems/diskann.md) beam search W=4-8 让一次 SSD I/O 拿多个邻居 → round-trips 减半到 1/8

[subramanya-2019-diskann §3.3]

## 关键洞见 2：全精度 re-rank 改变 recall 上限

[Faiss IVFPQ](../systems/faiss.md) 路径的最终距离用 PQ 估计 → recall plateau 在 60-70%（量化失真无法挽回）。

[DiskANN](../systems/diskann.md) 把 PQ 仅用于导航，**最终 ranking 用 SSD 取回的全精度向量**。这把 recall 上限从"PQ 失真极限"提升到 100%。代价：每次邻居读出 4 KB 扇区里**顺手包含**全精度坐标，所以是"免费"的（[subramanya-2019-diskann §3.5]）。

这一模式（DRAM-PQ + SSD-FullPrecision）是 [PQ](../concepts/product-quantization.md) 范式之后的新混合模式。

## 关键洞见 3：内存层级与算法选择的强耦合

| 层级 | 算法偏好 |
|---|---|
| L1/L2 cache | 任何算法都吃缓存友好的数据布局；HNSW prefetch 显式优化 [malkov-2016-hnsw §5] |
| DRAM only | 全图遍历 (HNSW/NSG/Vamana) 或量化驻留 (PQ) |
| DRAM + SSD | **必须 batch I/O + 减少 hop** → DiskANN |
| HBM (GPU) | brute force + fused k-selection ([WarpSelect](../concepts/warpselect.md)) |
| 网络存储 | 几乎无 ANN 方案能正常工作（除非完全 batch） |

## 工业方案适用边界

| N | DRAM 预算 | 推荐 |
|---|---|---|
| < 10M | 任何 | 全内存 graph（HNSW / NSG） |
| 10M–1B | 内存富余（>500 GB） | Faiss `IVF_HNSW,Flat` |
| 10M–1B | 内存吃紧（<200 GB） | Faiss `IVFPQ`（接受 ~70% recall）/ Faiss + GPU |
| **1B+** | **64 GB + SSD** | **DiskANN** |
| 1B+ | DRAM 极便宜 / 多机 | 多机 graph 分片（Taobao 模式） |
| 100B+（trillion） | 任意 | 分布式 mmap + 极端压缩（[Faiss trillion-scale](../benchmarks/faiss-trillion-scale.md)） |

## Open Questions

- **GPU + SSD 混合**：当前 [Faiss-GPU](../systems/faiss.md) 是全 HBM，[DiskANN](../systems/diskann.md) 是 CPU + SSD。两者结合（PQ in HBM + graph on NVMe）未在文献覆盖。
- **网络存储 ANN**：所有"disk-resident"分析假设本地 NVMe；远程块设备 / 对象存储下 latency 完全不同。
- **持久内存（CXL、Optane）作为中间层**：DRAM 与 SSD 之间出现新的存储层；ANN 算法适配未在 wiki 任何 source 覆盖。
- **SSD 寿命与 wear leveling**：高 QPS ANN 服务对 SSD 是持续随机读负载；写入压力低但寿命经济性需要量化。
- **SPANN（Chen 2021）等后继路线**：用 inverted file + SSD-resident posting list 路线（与 DiskANN graph 路线对立），wiki 尚未 ingest。
