---
title: Filtered-DiskANN vs Milvus / Faiss-IVF / NHQ + Microsoft 广告 A/B 实验
type: benchmark
sources: [gollapudi-2023-filtered-diskann]
related: [../concepts/filtered-vamana.md, ../concepts/vamana.md, ../systems/diskann.md, ../systems/milvus.md, ../systems/faiss.md, ../topics/attribute-filtering.md]
created: 2026-05-08
updated: 2026-05-08
---

# Filtered-DiskANN vs Milvus / Faiss-IVF / NHQ + Microsoft 广告 A/B 实验

**TL;DR**: Filtered-DiskANN 论文 §5-6 的核心实验。**(a)** vs Milvus / NHQ / Faiss IVF (post-processing + inline-processing) on Microsoft 真实数据集（Turing 2.6M, Prep 1M, DANN 3.3M）：**FilteredVamana / StitchedVamana 比所有 baseline 快一个数量级 QPS @ 90% recall** across 1%-100% specificity 全区间。**(b)** Microsoft 赞助广告搜索 2 周 A/B test：**+34.61% clicks (P=0.03), +48.95% revenue (P=0.009)**——小区域（<1% share）gain **+70.67%/+79.77%**。**(c)** SSD 模式 28M DANN：thousands QPS @ 90%+ recall on inexpensive SSDs。**(d)** Build time：FilteredVamana ~3× faster than StitchedVamana。[gollapudi-2023-filtered-diskann §5-6]

## 实验设置

[gollapudi-2023-filtered-diskann §5.1]

- **平台**：Azure E64dsv4 VM（Intel Xeon Platinum 8272CL @ 2.60 GHz, **64 vCPU, 500 GB RAM**）
- **Threads**：48 threads for QPS measurement
- **Datasets**：

| Dataset | Dim | # Pts | # Queries | Source | Filters | Filters/Pt | Unique Filters | Specificity（1pc） |
|---|---|---|---|---|---|---|---|---|
| **Turing**（Microsoft 内部） | 100 | **2.6M** | 996 | Text | Natural | 1.09 | **3070** | 7.7×10⁻⁶ |
| **Prep**（Microsoft 内部） | 64 | 1M | 10000 | Text | Natural | **8.84** | 47 | 0.09 |
| **DANN**（Microsoft 内部） | 64 | 3.3M | 32926 | Text | Natural | 3.91 | 47 | 0.150 |
| SIFT | 128 | 1M | 10000 | Image | **Random** | 1 | 12 | 0.082 |
| GIST | 960 | 1M | 1000 | Image | Random | 1 | 12 | 0.083 |
| msong | 420 | 992K | 200 | Audio | Random | 1 | 12 | 0.083 |
| audio | 192 | 53K | 200 | Audio | Random | 1 | 12 | 0.083 |
| paper | 200 | 2M | 10000 | Text | Random | 1 | 12 | 0.083 |

→ 真实数据集前 3 行（Microsoft 内部 Bing/广告流量）；后 5 行半合成（用 [42] 方法生成 random labels）。

- **Algorithms / Baselines**：
  1. **FilteredVamana**（L=90, R=96, parameter sweep R∈{32,64,96}, L=50-330）
  2. **StitchedVamana**（R_small=32, L_small=100, R_stitched=64, α=1.2）
  3. **IVF Inline-Processing** [GRANN ANNS Library]（4096 clusters）
  4. IVF Post-processing with Faiss IVF
  5. **NHQ-NPG** [Wang 2022, KGraph version]（appendix）
  6. **Milvus HNSW + IVF FLAT/SQ8/PQ**（appendix——QPS 太低单独 plot）
  7. HNSW Post-processing with Faiss HNSW
  8. Vamana Post-processing

## 主结果 1：Filtered Queries on Turing（[Fig 1]）

[gollapudi-2023-filtered-diskann §5.3.1]

Turing 2.6M, 3070 unique filters。**5 个 specificity 区间**（100pc / 75pc / 50pc / 25pc / 1pc）测 QPS-vs-recall@10：

定性观察：
- **FilteredVamana / StitchedVamana 在所有 5 个区间 90%+ recall 可达**，QPS 高
- **IVF post-processing / HNSW post-processing / Vamana post-processing**：**1pc / 25pc / 50pc specificity 失败**（recall 极低）——post-process 不能在 strict latency budget 下 cover 严苛 filter
- **IVF inline-processing**：100pc-50pc 还能 recall 90%，但 1pc / 25pc 退化（QPS 远低）
- **specificity 越低（filter 越严），FilteredVamana 优势越大**

> "10⁻¹ to 10⁻⁶ specificity，其他方法 fail to achieve any meaningful accuracy，**1000× lower QPS** for low specificity labels"

## 主结果 2：Filtered Queries on PREP（[Fig 2]）

[gollapudi-2023-filtered-diskann §5.3.2]

Prep 1M, 47 filters, 8.84 filters/point（每点多 label）。

90% recall @ 100pc specificity：
- FilteredVamana: **2.5× better than IVF inline (next-best)**
- StitchedVamana: **6× better than IVF inline**

→ 多 label/point 场景下 StitchedVamana 优势更大（每 vertex edge union 后候选邻居多）。

## 主结果 3：Filtered Queries on DANN（[Fig 3]）

[gollapudi-2023-filtered-diskann §5.3.3]

DANN 3.3M, 47 filters, 3.91 filters/point。

90% recall：
- FilteredVamana: **3× better than IVF inline**
- StitchedVamana: **7.5× better than IVF inline**

## 主结果 4：Build time（[Table 2]）

| Algorithm / Dataset | DANN (s) | Prep (s) | Turing (s) | Audio (s) | SIFT (s) |
|---|---|---|---|---|---|
| **FilteredVamana** | **159.8** | **66.6** | **103.4** | **1.3** | 44 |
| StitchedVamana | 469.9 | 222.6 | 295.9 | 1.6 | 24.4 |
| NHQ-NPG | NA | NA | NA | 1.1 | 24.4 |
| Milvus HNSW | 153.6 | 49.3 | NA | 5.5 | 72.0 |
| Faiss HNSW | 158.6 | 44.5 | 188.0 | 1.1 | 71.1 |

→ FilteredVamana build 通常 **~3× faster than StitchedVamana**；与 Milvus/Faiss HNSW 同量级。

## 主结果 5：Microsoft 广告 A/B 实验（[§6, Table 3-4]）

**核心 production deployment 数据**——A/B test 2 周 on live Microsoft sponsored ad search。

### 整体改善（47 region filters）

| Metric | Filter region 数 | % increase | P-value |
|---|---|---|---|
| **Clicks** | 47 | **+34.61%** | 0.03 |
| **Revenue** | 47 | **+48.95%** | 0.009 |

### 按 region 大小分组（[Table 4]）

| Region's share in index | Region 数 | Pct. incr. clicks | Pct. incr. revenue |
|---|---|---|---|
| 3-9% | 10 | 25.54% | 28.61% |
| 1-2% | 10 | 54.07% | 46.67% |
| **<1%** | **27** | **70.67%** | **79.77%** |

→ **小区域（<1% share）gain 最大**——证明 baseline post-filtering 在 selective filter 下严重失败。FilteredVamana 让小 region "fair representation"，retrieval complies 才严格符合 targeting-match。

## 主结果 6：SSD 模式（28M DANN, [Fig 6]）

[gollapudi-2023-filtered-diskann §5.5]

把 FilteredVamana 放进 DiskANN 框架（PQ codes 在 DRAM, graph + 全精度向量在 SSD）：
- 28M DANN dataset
- 24 threads, beam width 4, search L 40-100
- **Recall 80-95%（specificity 1pc-100pc）@ 100-150 SSD IOs/query**
- **Thousands QPS @ 90%+ recall on inexpensive SSDs**

→ 证明 SSD-resident filtered ANNS 可行——是 [SPANN](../systems/spann.md) / [SPFresh](../systems/spfresh.md) 在 filtered query 维度的同代延伸。

## 主结果 7：Robustness to Shuffled Labels（[Fig 4]）

[gollapudi-2023-filtered-diskann §5.4.2]

测试**当 label 与点位置不相关时**两算法行为：
- 用 disjoint discrete distribution 重新 sample label assignment
- → label 与 cluster 不相关

| Dataset | FilteredVamana 改变 | StitchedVamana 改变 |
|---|---|---|
| Prep | 几乎无变化 | QPS 显著降 |
| DANN | 两者都很小变化 | — |

→ FilteredVamana **更 robust** to uncorrelated labels；StitchedVamana 依赖 label-cluster correlation。

## 主结果 8：Performance on Unfiltered Queries（[Fig 5]）

[gollapudi-2023-filtered-diskann §5.4.3]

测试 filter-aware index 处理 **filter-free query**：
- 95% recall @ FilteredVamana ~0.8× QPS of original Vamana
- 95% recall @ StitchedVamana ~0.9× QPS of original Vamana

→ filter-aware build **几乎不损失 unfiltered 性能**——可作 unified 单 index 既 cover filtered 又 cover unfiltered query。

## 与 wiki 已有 attribute-filtering 系统对比

[per topics/attribute-filtering.md "工业方案对比"]

| 系统 | filter-aware build | filter-aware search | 实测 vs Filtered-DiskANN |
|---|---|---|---|
| Faiss IDSelector | ✗ | bitmap | post-processing 在 1pc 失败；inline 1000× 慢 |
| **Filtered-DiskANN** | **✓** | filter-aware greedy | baseline |
| Milvus 5-strategy | ✗ | partition-based | QPS < 300 across datasets, **orders of magnitude slower** [§A.2] |
| AnalyticDB-V 4-plan | ✗ | CBO | 论文论证 inferior（appendix） |
| NHQ-NPG | partial（label 拼 vector） | partial | KGraph version 仍**1 order of magnitude slower at 100 recall** |
| Pinecone hybrid | ✗ | inline-processing | 论文未直接 benchmark |

→ Filtered-DiskANN 是 **filter-aware build 的工业首例**，比所有 search-time 方法显著优。

## 可信度评估

- **实验设计**：作者 Microsoft Research 团队，Vamana 与 DiskANN 同公司同 author（Harsha Simhadri）
- **Production A/B test 是 wiki 内最强 production 实证**——P-value 0.009 / 0.03 in 2-week real ad serving with 47 regions
- **潜在偏向**：
  1. Turing / Prep / DANN 是 **Microsoft 内部数据**——不公开复现
  2. Milvus baseline 用默认参数——可能未充分调优；§A.2 承认"due to extremely low QPS"
  3. **NHQ 评估**：作者明示 "we have not found publicly available code to evaluate their results"——所以用 KGraph 版本的 NHQ；可能不是 NHQ 最优实现
  4. Vamana 同公司——主场优势可能存在
- **复现难度**：中。
  - SIFT/GIST/msong/audio/paper 公开
  - Microsoft 内部数据集**不公开**
  - GRANN ANNS Library / Filtered-DiskANN 代码 GitHub（commit hash 公开）
- **场景局限**：
  - **仅单 filter（exact match）**——AND/OR/NOT 复杂 filter 论文 §2 明确"future work"
  - 维度 64-960；高维 768/1024 未测
  - 单机；distributed 未测
  - 仅 incremental insert（FilteredVamana）；deletion 是 future work

## 与其他 wiki 大规模部署对比

| 部署 | 数据规模 | 索引 | filter scenario | source |
|---|---|---|---|---|
| Milvus @ Bing search | 不公开 | HNSW + IVF | partition strategy | [wang-2021-milvus] |
| ADBV @ Smart City | 13B records | VGPQ + HNSW | 4-plan CBO | [wei-2020-analyticdb-v] |
| **Filtered-DiskANN @ Microsoft sponsored ads** | **28M DANN benchmark + production scale 不公开** | **FilteredVamana** | **47 region filters A/B test +35-49% production gain** | **本论文** |

→ Filtered-DiskANN 是 wiki 内**第一次有 P-value-based production A/B test 的 vector ANNS 实证**——其他都是 latency / recall benchmark。
