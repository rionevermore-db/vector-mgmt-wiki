---
title: HNSW vs Faiss PQ on 200M SIFT
type: benchmark
sources: [malkov-2016-hnsw]
related: [../concepts/hnsw.md, ../concepts/product-quantization.md]
created: 2026-04-30
updated: 2026-05-07
---

# HNSW vs Faiss PQ on 200M SIFT

**TL;DR**: HNSW 论文 §5.4 的对比实验。HNSW 在召回-延迟曲线上完胜 Faiss 的两套 [PQ](../concepts/product-quantization.md) 配置，但内存占用是 Faiss 的 2–3 倍。[malkov-2016-hnsw §5.4 + Fig 15 + Table 3]

## 实验设置

- **数据集**：200M SIFT（来自 1B SIFT [13 in malkov-2016-hnsw]），128 维 L2。
- **查询任务**：1-NN。
- **硬件**：4×Xeon E5-4650 v2，128 GB RAM，OpenBLAS。
- **参与者**：
  - HNSW：专用精简构建（非 nmslib），无向量化、整数距离函数、支持增量。
  - Faiss：2017 年 5 月 build —— 当时刚发布的 PQ 实现 [12, 15 in malkov-2016-hnsw]。
- **HNSW 参数**：
  - HNSW 1：M=16, efConstruction=500
  - HNSW 2：M=16, efConstruction=40
- **Faiss 参数**：
  - Faiss 1：OPQ64,IMI2x14,PQ64
  - Faiss 2：OPQ32,IMI2x14,PQ32

## 结果

| | Build time | Peak memory | 召回-速度曲线 |
|---|---|---|---|
| HNSW 1 | 5.6 h | 64 GB | 最优 |
| HNSW 2 | 42 min | 64 GB | 仍优于两个 Faiss 配置 |
| Faiss 1 | 12 h | 30 GB | 比 HNSW 慢 |
| Faiss 2 | 11 h | 23.5 GB | 最慢 |

[malkov-2016-hnsw Table 3]

Fig 15 内嵌子图：HNSW 查询时间随数据集大小（10M → 200M）增长，**接近但偏离纯对数**——作者归因于 SIFT 的实际维度较高。

## 结论

- **速度**：HNSW 在所有召回点上都更快。
- **内存**：HNSW 64 GB vs Faiss 23–30 GB —— PQ 的核心价值（量化压缩）依然成立。
- **构建时间**：HNSW efConstruction=40 即可在 42 min 内构建完，远快于 Faiss 的 11–12 h。
- **核心 trade-off**：内存换速度。十亿级 SIFT 直接全放内存（HNSW 路径）需要数百 GB；PQ 路径在常规服务器上仍是必要选择。

## 可信度评估

- **实验设计**：作者作为 [HNSW](../concepts/hnsw.md) 提出方，对手算法（Faiss）参数取自 Faiss 官方 wiki，相对公允。
- **潜在偏向**：HNSW 用了"特殊精简构建"，未走标准 nmslib 路径——意味着部分加速来自工程优化（手写整数距离、避免 BLAS 开销）而非算法本身。Faiss 未做对应级别的定制。
- **复现难度**：中。Faiss 参数公开，HNSW 的"special build"在论文写作时未直接开源，需自行实现整数距离 + 非向量化代码。
- **场景局限**：单线程、单机、1-NN。多线程吞吐 + 分布式场景（HNSW 弱项 [malkov-2016-hnsw §6]）未参评；K>1 的召回曲线在论文其他章节有，但 §5.4 这套实验只测 1-NN。
