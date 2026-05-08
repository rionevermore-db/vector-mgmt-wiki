---
title: AnalyticDB-V vs Two-step Solution + VGPQ vs IVFPQ
type: benchmark
sources: [wei-2020-analyticdb-v]
related: [../systems/analyticdb-v.md, ../concepts/vgpq.md, ../concepts/product-quantization.md, ../topics/attribute-filtering.md]
created: 2026-05-08
updated: 2026-05-08
---

# AnalyticDB-V vs Two-step Solution + VGPQ vs IVFPQ

**TL;DR**: ADBV 论文 §6 的核心实验。**(a)** ADBV-V 比 "AnalyticDB + 独立 ANN engine" 两步式方案 **3-13× 快**——证明 native hybrid query engine 比 two-system 拼接的工程价值。**(b)** [VGPQ](../concepts/vgpq.md) 比 IVFPQ recall-vs-response-time 全程更优（SIFT1B / Deep1B / AliCommodity 三 dataset），同时 construction -10%、index size 同。**(c)** 4 plan CBO 在 selectivity 变化下能选最优执行计划，与单一 strategy 比可达 100× throughput。**(d)** 16 节点线性扩展 + mixed read/write 8:2/6:4 仍稳 4400 QPS。[wei-2020-analyticdb-v §6]

## 实验设置

[wei-2020-analyticdb-v §6.1]

- **平台**：Alibaba Cloud 16-node cluster
  - 每 node：Intel Xeon Platinum 8163 (2.5 GHz, 32 logical cores), 150 GB DRAM, 1 TB SSD
  - 10 Gbps Ethernet
- **数据集**：
  - **SIFT1B**（128-d byte, 1B base, 公开）
  - **Deep1B**（96-d float, 1B base, 公开）
  - **AliCommodity**（**自家 in-house**，830M × 512-d float，21 个结构属性如 color/sleeve_type/style/create_time）
- **查询模式**：
  - **Q1**：纯 vector top-k
  - **Q2**：hybrid `WHERE c >= p1 AND c <= p2 ORDER BY DISTANCE(...) LIMIT k`
- **超参**：top-k 默认 50；selectivity 通过调 p1/p2 范围控制（uniformly 假设）

## 结果 1：VGPQ vs IVFPQ（§6.2）

### Construction time + index size

[wei-2020-analyticdb-v Table 2，AliCommodity]

| Method | Time (min) | Size (GB) |
|---|---|---|
| IVFPQ(4096) | 155 | 112 |
| IVFPQ(8192) | 199 | 112 |
| **VGPQ(4096, 64)** | **144** | 112 |
| VGPQ(8192, 64) | 178 | 112 |
| VGPQ(8192, 128) | 182 | 112 |

→ 同 n_clusters，VGPQ **-10% build time**；index size 完全相同。

### Recall vs response time

[wei-2020-analyticdb-v Fig 10]：

- SIFT1B：VGPQ(1024, 24/32) > IVFPQ(1024)；VGPQ(2048, 48/32) > IVFPQ(2048)
- Deep1B：同样 VGPQ 全程更优
- AliCommodity：VGPQ(4096, 32/48/64) > IVFPQ(4096)

→ **三 dataset 三种维度 (128/96/512) VGPQ 都系统优于 IVFPQ**——证明几何 subcell 剪枝的普适性。

## 结果 2：Clustering-based partitioning（§6.3）

[wei-2020-analyticdb-v Fig 11]

SIFT1B_512p（512 partitions，cluster-based partitioning）：

| Search partitions | Recall |
|---|---|
| 1 | ~0.55 |
| 3 | **~0.95** (top-50) |
| 5 | ~0.97 |
| 10 | ~0.99 |
| 100 | ~1.0 |

→ 仅扫 3 个最相关 partition（512 中的）即达 95% recall——**partition pruning 100×+ throughput 提升**。

## 结果 3：4-plan CBO（§6.4）

[wei-2020-analyticdb-v Fig 12]

测三种代表场景：
- (a) k=50, recall ≥ 0.95, s 0.2-0.9999
- (c) k=250, recall ≥ 0.9
- (e) k=500, recall ≥ 0.85

每场景测 Plan A/B/C/D 在不同 selectivity α 下的 recall + response time。

**关键观察**：
- 单一 plan 不能覆盖全 selectivity 区间（每 plan 在某些 α 区间最优，其他区间退化）
- ADBV CBO **始终选最优 plan**（Fig 12 右列：CBO 曲线 = min(Plan A, B, C, D)）
- 在小 k + 高 recall 场景 CBO 优势最大

## 结果 4：vs Two-step Solution（§6.5）

[wei-2020-analyticdb-v Table 3，AliCommodity, k=50]

Two-step solution = AnalyticDB（关系 DB）+ 独立 ANN engine（IVFPQ 或 VGPQ）—— 标准工业实践（in 2020 era）。**Note**: 论文忽略数据传输时间——已经向 two-step 方倾斜结果。

| 系统 | p2=1（高选择） | p2=3（中等） | p2=9（低选择） |
|---|---|---|---|
| AnalyticDB+IVFPQ（two-step） | 241 ms | 77 ms | 47 ms |
| AnalyticDB+VGPQ（two-step） | 181 ms | 55 ms | 33 ms |
| **AnalyticDB-V（IVFPQ）** | **19 ms** | **35 ms** | **47 ms** |
| **AnalyticDB-V（VGPQ）** | **19 ms** | **35 ms** | **33 ms** |

→ ADBV native hybrid 在**小 selectivity（高过滤）下 3-13× 快**；高 selectivity 下 ≈ two-step（因为 plan 自动 fallback 到 brute-force）。

> **关键解读**：two-step 必须先用 ANN 拿大量候选（amplified k）然后过滤——"小过滤率"场景下 ANN 浪费大；ADBV 直接 push down 过滤到 plan A/B/C/D 中——这是 native hybrid 的工程价值。**Milvus partition-based Strategy E 后来声称比 ADBV 快 13.7×**，但相对 two-step 的对比是 ADBV 论文最有力的结果。

## 结果 5：Scalability（§6.6）

[wei-2020-analyticdb-v Fig 13]

集群从 4 → 16 节点（SIFT1B / Deep1B）：
- QPS **线性扩展**——节点增加 4× → QPS 增加 4×

## 结果 6：Mixed read/write（§6.6）

[wei-2020-analyticdb-v Fig 14]

SIFT1B 上 8:2 与 6:4 read/write 比例：
- ADBV peak ~4400 QPS 持续
- 24h 长测**throughput 仅微降**——证明 lambda 框架对 high-rate write 的稳健性

## 结果 7：Production case study（§6.7）

[wei-2020-analyticdb-v Fig 15-16]

**Smart City 车辆违章检测**：
- 70 节点 cluster, 13B records, 30 TB
- Hybrid query：timestamp + location + camera + color + 视觉相似
- 实时查询 hundreds of seconds → ms
- 24h 实测：insert / select QPS 与 latency 均稳定

## 可信度评估

- **实验设计**：作者 Alibaba 团队，AliCommodity 是自家数据集（不公开 → 复现难）；公开 SIFT1B/Deep1B 上同样 VGPQ 全胜
- **潜在偏向**：
  1. **AliCommodity 数据集 + 自家 OLAP（AnalyticDB）+ 自家 storage（Pangu）+ 自家 scheduler（Fuxi）**——baseline 完全 Alibaba stack；可能"home turf" advantage
  2. **Two-step 实验忽略数据传输时间**——已对 two-step 友好；但 ADBV 仍 3-13× 快 → 真实场景 two-step 更慢
  3. **没有与 Faiss / Milvus / 其他 vector DBMS 直接比较** —— 仅与 OLAP+ANN two-step 比；没有"ADBV vs Milvus"数据
  4. **CBO 实验 selectivity α 是 user-controlled** —— 真实场景 α 估计有误差，CBO 可能选错 plan；论文未深入
- **复现难度**：高。
  - SIFT/Deep 公开
  - AliCommodity 不公开
  - ADBV 在 Alibaba Cloud 闭源 SaaS（PolarDB-V 类似产品）
  - Pangu 闭源 → 其他 cloud 用户无法直接复现
- **场景局限**：
  - 维度 96/128/512；现代 768/1024-d 未测
  - 单 region；多 region 未测
  - 万亿规模未测（论文测 1B；§6.7 production 13B 但未深 benchmark）

## 与其他 wiki 大规模部署对比

| 部署 | 数据 | 索引 | Update | source |
|---|---|---|---|---|
| Faiss-GPU SIFT1B | 1B | IVFPQ + GPU | freeze | [johnson-2017-faiss-gpu] |
| NSG @ Taobao | 2B | NSG 多机分片 | static | [fu-2017-nsg] |
| Faiss trillion @ Meta | 1.5T | SQ6 + HNSW coarse | freeze | [douze-2024-faiss-library §7.1] |
| DiskANN @ z840 | 1B | Vamana + PQ + SSD | streamingMerge | [subramanya-2019-diskann] |
| SPANN @ Bing | 1B+ / 千亿+ | HBC + closure + SSD | static | [chen-2021-spann] |
| Milvus 1.x SIFT1B | 1B | IVF_FLAT + HNSW etc | LSM | [wang-2021-milvus] |
| SPFresh @ Azure lsv3 | 1B | SPANN + LIRE | In-place | [xu-2023-spfresh] |
| Manu @ AWS m5.4xlarge | 100M（实测） | HNSW / IVF | delta τ + stream indexing | [guo-2022-manu] |
| **ADBV @ Smart City Alibaba Cloud** | **13B records / 30 TB**（生产）| **VGPQ + HNSW lambda** | **lambda streaming + batching** | **本论文** |
| **ADBV @ Freshippo** | **800M × 512-d, 4000 QPS, 80% hybrid** | **同上** | **同上** | **本论文** |

ADBV 的差异化：**唯一 13B records production scale + native SQL + 4-plan CBO**——是 wiki 内**OLAP-extended-with-vector** 路径唯一系统化论证。
