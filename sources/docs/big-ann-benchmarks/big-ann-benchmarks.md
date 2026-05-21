# Big ANN Benchmarks — captured methodology (WebFetch summary, 2026-05-21)

> 来源：`github.com/harsha-simhadri/big-ann-benchmarks` + `big-ann-benchmarks.com`。两届 NeurIPS competition。

## NeurIPS 2021 — Billion-Scale ANN Challenge

**3 tracks**：

| Track | Baseline | Search 机器 | Build 机器 | 目标吞吐 |
|---|---|---|---|---|
| **T1 in-memory（标准硬件）** | FAISS | Azure Standard_F32s_v2（32 vCPU / 64 GB） | F64s_v2（64 vCPU / 128 GB / 4 TB SSD） | 10,000 QPS |
| **T2 out-of-core / SSD** | DiskANN | Azure Standard_L8s_v2（8 vCPU / 64 GB / 1 TB local SSD） | 同 T1 | 1,500 QPS |
| **T3 custom hardware** | — | GPU / FPGA / 加速器（自带或远程） | — | 2,000 QPS + power + cost/query |

**6 个 billion-scale 数据集（均 1B）**：

| Dataset | Dim | dtype | Metric |
|---|---|---|---|
| BIGANN | 128 | uint8 | L2 |
| Yandex DEEP-1B | 96 | float32 | L2 |
| Yandex Text-to-Image-1B | 200 | float32 | inner-product |
| Microsoft Turing-ANNS-1B | 100 | float32 | L2 |
| Microsoft SPACEV-1B | 100 | int8 | L2 |
| Facebook SimSearchNet++ (FB-SSNPP) | 256 | uint8 | L2 |

**指标**：private query set 上的 **recall@10 at fixed query throughput**;排名 = "sum of recall improvements over baseline at target QPS"，跨全部数据集求和。

**Winners（2021-12）**：
- T1：**kst_ann_t1**（Kuaishou + Tsinghua）
- T2：**BBANN**（**Zilliz** + Southern University of Science and Technology）
- T3：**OptaNNe**（Intel / Intel Labs / UC Davis）

## NeurIPS 2023 — Practical Vector Search Challenge

**4 tracks**：

| Track | Dataset | Points | Dim | dtype | Metric | Baseline 数字 |
|---|---|---|---|---|---|---|
| **Filter** | YFCC | 10M | 192 | uint8 (ℓ₂) | QPS @ recall，tag 过滤（1-2 tags） | FAISS 3,200 QPS |
| **Sparse** | MSMARCO / SPLADE | 8.8M | <100K（avg ~120 nonzeros） | float32 (IP) | QPS @ 90% recall | Linear scan 101 QPS |
| **OOD（cross-modal）** | Yandex Text-to-Image | 10M | 200 | float32 (IP) | QPS @ recall，query/base 分布不同 | DiskANN 4,882 QPS |
| **Streaming** | MS-Turing 30M slice | — | 100 | float32 (ℓ₂) | runbook checkpoint 平均 recall | DiskANN 0.883 recall@10（45 min） |

- 标准化硬件：**Azure Standard D8lds v5（8 vCPU / 16 GiB）**，Intel Xeon Platinum 8370C @ 2.80GHz。
- Build time 限制：non-streaming track **12 小时**;streaming 限 **1 小时 + 8 GB DRAM**。

## 区别 / 定位

- **标准化硬件 + 私有 query set** —— competition 级严谨,防过拟合。
- billion-scale（2021）+ practical 变体（2023：filter / sparse / OOD / streaming）。
- 与 ann-benchmarks（million-scale 算法级）互补,与 VectorDBBench（system-level 厂商级）正交。
