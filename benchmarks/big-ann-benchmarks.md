---
title: Big ANN Benchmarks（NeurIPS 十亿级 + practical 竞赛标准）
type: benchmark
sources: [bigann-benchmarks-docs]
related: [vectordbbench.md, ann-benchmarks.md, ../topics/ann-benchmarking-methodology.md, ../systems/diskann.md, ../systems/faiss.md, ../systems/spann.md, ../systems/freshdiskann.md, ../concepts/splade-sparse-retrieval.md, ../topics/multimodal-embedding-retrieval.md, ../topics/disk-vs-memory-ann.md, ../concepts/manu-ssd-hierarchical-kmeans.md]
created: 2026-05-21
updated: 2026-05-21
---

# Big ANN Benchmarks

**TL;DR**: Harsha Vardhan Simhadri 等组织的 **NeurIPS competition** 系列,ANN 领域**标准化硬件 + 私有 query set** 的十亿级 / practical 评测权威。两届:**NeurIPS 2021 Billion-Scale**（3 track，6× 1B 数据集）和 **NeurIPS 2023 Practical Vector Search**（4 track：filter / sparse / OOD / streaming）。**与 wiki 多条线索直接相连**——[DiskANN](../systems/diskann.md) / [FAISS](../systems/faiss.md) 是官方 baseline、[SPLADE](../concepts/splade-sparse-retrieval.md) 是 sparse track 编码器、cross-modal OOD track 对接 [multimodal retrieval](../topics/multimodal-embedding-retrieval.md)、2021 T2（SSD）冠军 **BBANN = Zilliz**（[Milvus](../systems/milvus.md) 母公司,亦见 [Manu SSD-aware indexing](../concepts/manu-ssd-hierarchical-kmeans.md) 拿 BigANN Track 2 冠军）。[per sources/docs/big-ann-benchmarks/big-ann-benchmarks.md]

> **三方分工**见 [topics/ann-benchmarking-methodology.md](../topics/ann-benchmarking-methodology.md)：big-ann = **十亿级 + 标准化硬件 + 竞赛严谨**;[ann-benchmarks](./ann-benchmarks.md) = 算法级 million-scale;[VectorDBBench](./vectordbbench.md) = 系统级 + 成本。

## 实验设置

### NeurIPS 2021 — Billion-Scale ANN（3 tracks）

[per sources/docs/big-ann-benchmarks/big-ann-benchmarks.md]

| Track | Baseline | Search 机器 | 目标吞吐 |
|---|---|---|---|
| **T1 in-memory（标准硬件）** | [FAISS](../systems/faiss.md) | Azure F32s_v2（32 vCPU / 64 GB） | 10,000 QPS |
| **T2 out-of-core / SSD** | [DiskANN](../systems/diskann.md) | Azure L8s_v2（8 vCPU / 64 GB / 1 TB SSD） | 1,500 QPS |
| **T3 custom hardware** | — | GPU / FPGA / 加速器 | 2,000 QPS + power + cost |

Build 机器（T1/T2）：Azure F64s_v2（64 vCPU / 128 GB / 4 TB SSD）。

**6 个 billion-scale 数据集（均 1B）**：

| Dataset | Dim | dtype | Metric |
|---|---|---|---|
| BIGANN | 128 | uint8 | L2 |
| Yandex DEEP-1B | 96 | float32 | L2 |
| Yandex Text-to-Image-1B | 200 | float32 | inner-product |
| Microsoft Turing-ANNS-1B | 100 | float32 | L2 |
| Microsoft SPACEV-1B | 100 | int8 | L2 |
| Facebook SimSearchNet++ (FB-SSNPP) | 256 | uint8 | L2 |

### NeurIPS 2023 — Practical Vector Search（4 tracks）

| Track | Dataset | Points × Dim | Metric | Baseline |
|---|---|---|---|---|
| **Filter** | YFCC | 10M × 192 (uint8) | QPS @ recall，tag 过滤 | [FAISS](../systems/faiss.md) 3,200 QPS |
| **Sparse** | MSMARCO / [SPLADE](../concepts/splade-sparse-retrieval.md) | 8.8M × <100K（avg ~120 nonzeros） | QPS @ 90% recall | Linear scan 101 QPS |
| **OOD（cross-modal）** | Yandex Text-to-Image | 10M × 200 (float32, IP) | QPS @ recall | [DiskANN](../systems/diskann.md) 4,882 QPS |
| **Streaming** | MS-Turing 30M slice | × 100 (float32) | runbook checkpoint 平均 recall | DiskANN 0.883 recall@10（45 min） |

标准化硬件：**Azure D8lds v5（8 vCPU / 16 GiB）**，Xeon 8370C。Build 限制:non-streaming 12 h;streaming **1 h + 8 GB DRAM**。

## 结果

### 指标口径

- 2021：private query set 上 **recall@10 at fixed query throughput**;排名 = "sum of recall improvements over baseline at target QPS" 跨数据集求和——**固定吞吐看 recall 增益**,与 ann-benchmarks 的"画整条 Pareto"互补。
- 2023：各 track 自定义（filter/OOD/sparse 看 QPS@recall;streaming 看 runbook 平均 recall）。

### 冠军（2021-12）

| Track | Winner |
|---|---|
| T1 | kst_ann_t1（Kuaishou + Tsinghua） |
| **T2（SSD）** | **BBANN（Zilliz + SUSTech）** |
| T3 | OptaNNe（Intel + UC Davis） |

> **wiki 解读**：T2 冠军 BBANN 来自 **Zilliz**——同一团队的 SSD-aware 方案在 [Manu 论文 §4.4](../concepts/manu-ssd-hierarchical-kmeans.md) 记为"NeurIPS 2021 BigANN Track 2 winner"（hierarchical k-means + LSH 复制,比 baseline same QPS recall +60%）。本 benchmark 是那个冠军 claim 的**竞赛出处锚点**。streaming track 的 sustained insert/delete 评测范式与 [FreshDiskANN](../systems/freshdiskann.md) / [SPFresh](../systems/spfresh.md) 的 streaming 实验同源。

## 可信度评估

- **设计合理性**：**标准化硬件 + 私有 query set + 固定吞吐口径** —— competition 级严谨,防止参数过拟合与 cherry-pick,是三个 benchmark 里**最防作弊**的。
- **偏向**：竞赛中立(NeurIPS workshop 主办)。但参赛是 self-selected 团队,track 之外的真实运维(成本 / 持续插入退化)不覆盖。
- **复现难度**：高——billion-scale 需大机器 + 长 build(12h 限);硬件标准化反而降低跨论文比较难度。
- **边界**：纯检索性能,无成本(QP$)维度、无端到端 ingest duration 横测(那些由 [VectorDBBench](./vectordbbench.md) 补)。空间 / geo 检索两届均未设 track——呼应 wiki 内"空间 benchmark 空白"。

## Cited by

- [queries/milvus-laion-100m-ingest-rate.md](../queries/milvus-laion-100m-ingest-rate.md)
