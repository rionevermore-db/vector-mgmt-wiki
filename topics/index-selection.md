---
title: Index Selection（如何在 Faiss 索引家族里选）
type: topic
sources: [douze-2024-faiss-library, jegou-2011-pq, malkov-2016-hnsw, fu-2017-nsg, guo-2019-scann]
related: [../systems/faiss.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/product-quantization.md, ../concepts/scann.md, ../concepts/warpselect.md]
created: 2026-05-07
updated: 2026-05-07
---

# Index Selection

**TL;DR**: 在 Faiss 提供的几十种 index 类型里选哪个？决策的核心轴是 **N（向量数）+ memory budget + 是否需要精确**。论文 §A.5 给了决策树（Fig 10），可拆成 4 个量级 + 是否压缩 + 是否需要精确 reranking 的组合。[douze-2024-faiss-library §A.5 + Fig 10]

## 问题陈述

每次启动新项目就得重选 index 是 ANN 工程化的最大隐性成本：

- 算法 paper 的 benchmark 大多在 1M / 10M 量级；真实数据集可能 100M / 10B
- 内存预算约束（CPU RAM / GPU HBM / 磁盘）决定能否走 graph 路径
- 是否要 incremental add、是否要 delete、是否要 filtered search 都进一步收窄选择
- ann-benchmarks 的 leaderboard 不直接对应工程问题（数据集大小固定、不考虑 build time）

## 相关概念

- [HNSW](../concepts/hnsw.md) / [NSG](../concepts/nsg.md)：graph 类（中等规模、内存富余）
- [PQ / IVFADC](../concepts/product-quantization.md)：quantization 类（大规模 / 内存敏感）
- [WarpSelect](../concepts/warpselect.md) / [Faiss-GPU](../systems/faiss.md)：GPU 路径（数据装得进 GPU memory）
- [ScaNN](../concepts/scann.md)：MIPS 优先 + 极致 SIMD

## 决策流程（Faiss 论文 §A.5 Fig 10 改编）

### Step 1：是否需要精确？

- **是** → `IndexFlat`（brute force，CPU 或 GPU）
- 否，继续

### Step 2：N 多少？

| N | 推荐 |
|---|---|
| < 10k | `IndexFlat` 即可（数据少不值得索引） |
| 10k – 1M | `IndexNSGFlat`（最快 trade-off，但不支持增量） / `IndexHNSWFlat`（次快、支持增量） |
| 1M – 10M | `IndexIVF{16k,64k}_Flat`（IVF + 不压缩） |
| 10M – 100M | `IndexIVF64k_HNSW + Flat` 或 + 压缩 |
| 100M – 1B | `IndexIVF256k_HNSW + 压缩`（HNSW-as-coarse-quantizer） |
| > 1B | `IndexIVF1M_HNSW + PQ / SQ`（必须压缩） |

[douze-2024-faiss-library Fig 10]

### Step 3：内存预算？

设每向量预算 M 字节，d 是原维度：

| 预算 | 推荐编码 |
|---|---|
| M > 4d | `IVFx,Flat`（不压缩） |
| 2d < M ≤ 4d | `IVFx,SQfp16` |
| d ≤ M ≤ 2d | `IVFx,SQ8`（8-bit scalar quantizer） |
| M < d 但 build 慢可接受 | `IVFx,RQM`（residual quantizer）或 `OPQM/2,IVFx,PQM/2x4fs`（fast scan） |
| M < d 但 build 必须快 | `OPQM,IVFx,PQM` |

### Step 4：增量需求？

- 需要 add + delete + update：避开 NSG（不支持增量）；HNSW 支持 add 不支持 delete；考虑 `IndexIDMap2`（移除）+ HNSW
- 仅需要 add：HNSW 或 IVF 类都可
- 一次性构建：随便选

### Step 5：filtered search？

需要按 metadata 过滤：

- 简单情况：`IDSelector` callback（慢）
- 大量重复 metadata：`bow_id_selector` 用 bit-signature 预筛（fast）
- 高选择率：metadata-first（先按属性挑出小集合，brute force）

## 工业方案对比（同一问题的不同选择）

| 案例 | N | 选择 | 理由 |
|---|---|---|---|
| Glove1.2M MIPS | 1.2M | [ScaNN](../concepts/scann.md)（IVFPQ + anisotropic + SIMD FastScan） | MIPS 优先，high-recall 区最快 [guo-2019-scann] |
| Taobao 商品检索 | 2B | [NSG](../concepts/nsg.md) 分布式 32 分区 | 全程内存内 + 5ms 响应 [fu-2017-nsg] |
| Faiss 内部 trillion-scale | 1.5T | `PCAR72,SQ6` + HNSW10M coarse + mmap | 极端压缩 + 分布式磁盘 [详见 benchmark](../benchmarks/faiss-trillion-scale.md) |
| 学术 ann-benchmarks 1M | 1M | HNSW 默认 | 最简单 + 最稳 |

## Open Questions

- **决策树是 N 优先；现实里 query latency budget 优先**：100M 向量但要 P99 < 5ms 是另一组约束，论文决策树未直接覆盖
- **多目标场景**：既要 MIPS 又要 L2-NN（推荐 + 去重在同一 service），是否能共享 index？
- **磁盘 vs 内存边界正在移动**：DiskANN / SPANN 改变了 1B+ 必须 quantization 的旧定理；wiki 尚未 ingest
- **filtered search 的最优策略**：vector-first vs metadata-first 的 cutoff 是 selection rate；Faiss 用经验阈值（约 3×10⁻⁴），但理论上可学习
