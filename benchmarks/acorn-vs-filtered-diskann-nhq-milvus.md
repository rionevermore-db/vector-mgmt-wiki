---
title: ACORN vs Filtered-DiskANN / NHQ / Milvus / FAISS-HNSW（4 datasets + 25M LAION scale）
type: benchmark
sources: [patel-2024-acorn]
related: [../concepts/acorn.md, ../concepts/filtered-vamana.md, ../concepts/hnsw.md, ../topics/attribute-filtering.md]
created: 2026-05-08
updated: 2026-05-08
---

# ACORN vs Filtered-DiskANN / NHQ / Milvus / FAISS-HNSW

**TL;DR**: ACORN 论文 §7 的核心实验。**4 datasets × 多 baseline**：SIFT1M / Paper（LCPS）+ TripClick / LAION 1M, 25M（HCPS, 10^8-10^11 unique predicates）。**(a)** LCPS 上 ACORN-γ **2-10× higher QPS @ 0.9 recall** vs FilteredVamana / StitchedVamana / NHQ-NPG（即使 LCPS 是 specialized index 主场）；**(b)** HCPS 上 **FilteredVamana / NHQ / Milvus 完全 fail**（无法 build 或 high-cardinality predicate not supported）；ACORN 比 HNSW post-filter **30-50× higher QPS**；**(c)** **25M LAION**（the largest scale）ACORN-γ 比 next best baseline **>1000× higher QPS @ 0.9 recall**；**(d)** ACORN-1 approximates ACORN-γ search 在 5× lower QPS 但 9-53× lower TTI——resource-constrained 选项。[patel-2024-acorn §7]

## 实验设置

[patel-2024-acorn §7.1]

- **平台**：AWS m5d.24xlarge（**96 vCPU, 370 GB RAM, 196 threads**）
- **Datasets**:

| Dataset | Type | # Vectors | Vector Dim | Vector Source | Predicate Operators | Predicate Cardinality | Avg Selectivity |
|---|---|---|---|---|---|---|---|
| SIFT1M | LCPS | 1M | 128 | image (SIFT) | equals(y) | **12** | 0.083 |
| Paper | LCPS | 2M | 200 | text passage | equals(y) | **12** | 0.083 |
| **TripClick** | **HCPS** | 1.06M | 768 | medical query log | contains(y_1 OR y_2 OR ...) AND between(y_1, y_2) | **>10^8** | 0.17, 0.26 |
| **LAION 1M** | **HCPS** | 1M | 512 | image + caption | regex-match(y) AND contains(y_1 OR y_2 OR ...) | **>10^11** | 0.056-0.13 |
| **LAION 25M** | HCPS at scale | **24.65M** | 512 | same as above | same | **>10^11** | same |

- **Baselines**:
  1. **HNSW Post-filter** (FAISS, M=32 efc=40 default)
  2. **Pre-filter**（构建结构化属性 list → brute force similarity over filtered set）
  3. **FilteredVamana** [gollapudi-2023-filtered-diskann]：L=90, R=96
  4. **StitchedVamana**：R_small=32, L_small=100, R_stitched=64, α=1.2
  5. **NHQ** [Wang 2022, wiki 未 ingest]：NHQ-NPG_NSW + NHQ-NPG_KGraph
  6. **Milvus**: IVF-Flat / IVF-SQ8 / HNSW / IVF-PQ
  7. **Oracle Partition**（理论 ideal，仅 LCPS feasible）：每 predicate 一个 HNSW

## 主结果 1：LCPS Datasets（[Fig 7]，SIFT1M / Paper）

[patel-2024-acorn §7.3.1]

**SIFT1M @ 0.9 recall@10** 定性 QPS 顺位：

| 排名 | 方法 | 备注 |
|---|---|---|
| 1 | **Oracle Partition**（理论 ideal） | 每 predicate 一个 HNSW，不实用 |
| 2 | **ACORN-γ** | **2-10× over FilteredVamana / NHQ** |
| 3 | ACORN-1 | 5× lower QPS than ACORN-γ |
| 4 | StitchedVamana | LCPS 主场仍败给 ACORN |
| 5 | FilteredVamana | 同上 |
| 6 | NHQ-NPG_KGraph | 单 attribute + equality 限制 |
| 7 | HNSW Post-filter | post-processing 限制 |
| 8 | Pre-filter | 高 selectivity OK 但 throughput 低 |
| 9 | Milvus（IVF-Flat / SQ8 / HNSW / PQ） | 论文 §A.2 single-plot 因 QPS 太低 |

→ **即使 LCPS 是 FilteredVamana / NHQ 的主场**，ACORN-γ 仍胜出 2-10×。这是 ACORN 论文核心论据：predicate-agnostic 不仅在 HCPS 有用，**LCPS 上也比 specialized index 更优**。

### # Distance Computations to Achieve 0.8 Recall（[Table 3]）

| Method | SIFT1M | Paper |
|---|---|---|
| Oracle Partition | 398.0（最少）| 281.1 |
| **ACORN-γ** | **611.0 (+53.5%)** | **383.7 (+36.6%)** |
| ACORN-1 | 999.6 (+151%) | 567.8 (+102%) |
| HNSW Post-filter | 1837.8 (+362%) | 1425.5 (+406%) |

→ ACORN-γ 距离计算次数仅比 oracle ideal 多 36-54%——**接近 sublinear 性能上限**。HNSW post-filter 4-5× more distance computations（无 filter-aware build → search expand 大 candidate）。

## 主结果 2：HCPS Datasets（[Fig 8, 11]，TripClick / LAION）

[patel-2024-acorn §7.3.2]

```
TripClick (1.06M, 10^8 predicates):
   ACORN-γ:        ~10000 QPS @ 0.9 recall
   HNSW Post-filter: ~300 QPS @ 0.9 recall
   FilteredVamana:  fail （high-cardinality predicate set 无法 build）
   NHQ:            fail (single attribute + equality only)
   Milvus:         fail (regex / contains operator 不 support)
   Pre-filter:     low QPS
   
   → ACORN-γ 比 HNSW post-filter 30-50× faster
```

**LAION 1M (10^11 predicates) + LAION 25M**：
- **LAION 25M ACORN-γ 比 next best baseline >1000× higher QPS @ 0.9 recall**——三个数量级优势

## 主结果 3：Varied Predicate Selectivity（[Fig 9]，TripClick）

5 selectivity bin (1pc-99pc) 全部测：

| Selectivity | 主场方法 | ACORN-γ 表现 |
|---|---|---|
| 1pc (s=0.0127) | **Pre-filter** 主场 | ACORN-γ 略胜 pre-filter |
| 25pc (s=0.0485) | 接近 boundary | **ACORN-γ 胜** |
| 50pc (s=0.1215) | 中等 | **ACORN-γ 胜** |
| 75pc (s=0.2529) | post-filter / pre-filter 都 OK | **ACORN-γ 胜** |
| 99pc (s=0.6164) | 大 selectivity post-filter 主场 | **ACORN-γ 略胜** |

→ ACORN-γ **全 selectivity 区间稳定击败 baseline**——是论文 robustness 核心 claim。

## 主结果 4：Varied Query Correlation（[Fig 10]，LAION 1M）

[patel-2024-acorn §7.3.2]

| Correlation | 描述 | ACORN-γ 表现 |
|---|---|---|
| Negative correlation | query 与 target anti-cluster | **ACORN-γ 28-100× higher QPS than next best** |
| No correlation | query 与 target 独立 | ACORN-γ 仍胜 |
| Positive correlation | query 与 target 聚集 | ACORN-γ 略胜 |

→ ACORN-γ 在**最难的 negative correlation** 下优势最大——证明 predicate subgraph traversal 不依赖 correlation。

## 主结果 5：Construction Overhead（[Table 4-5]）

| Algorithm / Dataset | TripClick TTI (s) | LAION-1M TTI | LAION-25M TTI | Sift1M TTI | Paper TTI |
|---|---|---|---|---|---|
| **ACORN-γ** | **9902.9** | 835.8 | 38007.5 | 148.9 | 255.6 |
| ACORN-1 | **322.9** | **25.9** | **705.3** | **8.6** | **27.0** |
| HNSW | 891.0 | 32.9 | 1147.2 | 11.3 | 29.2 |
| FilteredVamana | NA | NA | NA | 18.3 | 51.9 |
| StitchedVamana | NA | NA | NA | 69.2 | 189.7 |

→ ACORN-γ TTI **~11× HNSW**；ACORN-1 TTI **几乎等同 HNSW + 9-53× faster than ACORN-γ**。

| Index Size (GB) | TripClick | LAION-1M | LAION-25M | Sift1M | Paper |
|---|---|---|---|---|---|
| ACORN-γ | 4.9 | 2.4 | 59 | 0.98 | 2.5 |
| ACORN-1 | 4.6 | 2.3 | 59 | 0.93 | 2.4 |
| HNSW | 4.1 | 2.2 | 54 | 0.75 | 2.1 |
| Flat | 3.1 | 1.9 | 47 | 0.51 | 1.6 |

→ ACORN-γ index size 仅比 HNSW **1.2× larger**——25% smaller than StitchedVamana。

## 主结果 6：ACORN-γ Pruning Effectiveness（[Fig 12]）

[patel-2024-acorn §7.4.2]

3 pruning 策略对比：
1. **ACORN-γ pruning**（predicate-agnostic compression with M_β）
2. **Metadata-aware RNG-based pruning**（FilteredVamana 用此）
3. **HNSW pruning**（metadata-blind）

| Pruning | TTI | Space | Pruned edges | Search Perf @ 20K QPS |
|---|---|---|---|---|
| **ACORN-γ pruning** | 中 | 中 | 中 | **最高 recall** |
| Metadata-aware RNG | 高 | 高 | 高 | 略低 |
| HNSW pruning | 低 | 低 | 低 | **显著低 recall** |

→ ACORN 自家 pruning 在 efficiency vs search performance 平衡最优。

## 与 wiki 现有 attribute-filtering benchmark 对比

[per topics/attribute-filtering.md]

| Benchmark | 系统 | Predicate 范围 | 最大 scale | 关键 take |
|---|---|---|---|---|
| AnalyticDB-V vs two-step | ADBV | 4-plan 范围 | 1B SIFT/Deep | 3-13× over two-step |
| Filtered-DiskANN vs Milvus | FilteredVamana | ≤1000 equality | 28M DANN | 5-10× over Milvus + Microsoft A/B +35% |
| **ACORN vs FilteredVamana / NHQ / Milvus** | **ACORN-γ** | **>10^11 predicates + regex / contains / between / OR** | **25M LAION** | **2-1000× over baselines + LCPS / HCPS 全覆盖** |

→ ACORN 是 wiki 内**首个 HCPS-supporting benchmark**——其他系统全限于 LCPS。

## 可信度评估

- **实验设计**：作者 Stanford / DBOS / Berkeley，**有 NHQ / FilteredVamana 完整 baseline 对比**——比 [FilteredVamana 论文](./filtered-diskann-vs-milvus-faiss-nhq.md) 比较更全面（FilteredVamana 论文未跟 ACORN 对比，因 ACORN 时间更晚）
- **潜在偏向**：
  1. **TripClick / LAION 是 ACORN 团队精心选择的 HCPS workload**——baseline (FilteredVamana, NHQ) 在这些 workload 上不是设计用途
  2. **ACORN-γ TTI ~11× HNSW** 但论文 framing 上没显著强调；FilteredVamana TTI 在论文 [gollapudi-2023] 中是亮点
  3. **没与 OOD-DiskANN / FreshDiskANN 等其他 specialized index 对比**——只比 FilteredDiskANN
  4. **Milvus baseline 用 single-thread + 默认参数**——产线场景下 Milvus 表现可能高很多（[FilteredVamana 论文 §A.2] 同样 caveat）
- **复现难度**：低-中。
  - SIFT1M / Paper / LAION 公开
  - TripClick 公开（health web search log）
  - **ACORN 代码开源** [stanford-futuredata/ACORN]
  - FilteredVamana / NHQ 代码也开源
  - 主要是 25M LAION TTI 38000s（10+ 小时）需大机器
- **场景局限**：
  - 单机；distributed 未测
  - 维度 128-768；现代 1024+ 未测
  - **没有 production A/B test**（FilteredVamana 有 Microsoft +35% revenue 实证；ACORN 是学术 benchmark）
  - L2 / cosine 距离；MIPS 优化未涉及

## 与其他 wiki 大规模部署对比

| 部署 | 数据规模 | 索引 | filter capability | source |
|---|---|---|---|---|
| Filtered-DiskANN @ Microsoft sponsored ads | 28M DANN | FilteredVamana | ≤1000 equality | [gollapudi-2023] |
| ADBV @ Smart City | 13B records | VGPQ + HNSW lambda | 4-plan CBO（partial） | [wei-2020-analyticdb-v] |
| **ACORN @ Stanford research** | **25M LAION (largest test)** | **HNSW + predicate-agnostic** | **>10^11 predicates + 任意 operator** | **本论文** |
| Milvus @ Bing search | 不公开 | HNSW + IVF | partition + bitmap | [wang-2021-milvus] |
| Pinecone hybrid | 不公开 | adaptive slab | metadata filter | [pinecone-docs] |

→ ACORN 是**filter cardinality 与 operator 维度的领先者**——但 production 部署 wiki 内**仅 GitHub 开源，无 production A/B test 实证**。

## ACORN 在 wiki 各对比维度的位置

[per topics/attribute-filtering.md]

| 维度 | ACORN 在哪 |
|---|---|
| Filter cardinality 上限 | **领先**（>10^11 vs FilteredVamana ≤1000） |
| Predicate operator 支持 | **领先**（regex, contains, between, OR vs FilteredVamana equality only） |
| LCPS 性能 | 2-10× over specialized | 
| HCPS 性能 | 30-1000× over generic baselines |
| Construction overhead | 中（~11× HNSW； ACORN-1 ~HNSW） |
| Index size | 中（1.25× HNSW） |
| Production deployment | **学术** vs FilteredVamana 有 Microsoft A/B 实证 |
| Streaming insert | partial（ACORN-1 OK； ACORN-γ 部分） |
| Distributed | ✗ |

Cited by: [queries/milvus-laion-100m-ingest-rate.md](../queries/milvus-laion-100m-ingest-rate.md)（LAION-25M HNSW build 是 100M 摄入速率外推的唯一 anchored 锚点）
