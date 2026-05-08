---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-08
phase: post
ingest-context: gollapudi-2023-filtered-diskann
wiki-pages-total: 46
cited-pages: [topics/index-selection.md, topics/disk-vs-memory-ann.md, topics/attribute-filtering.md, concepts/filtered-vamana.md, systems/diskann.md, benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md]
cited-count: 6
---

# Post-snapshot (gollapudi-2023-filtered-diskann): scale-tier-shifts

## TL;DR (delta from yang-2020-pase post)

**新增第 9 质变维度：Filter selectivity → search-time vs build-time strategy**。当 filter 严（specificity 1%-25%），search-time post-processing 失败，必须切到 **filter-aware build**（FilteredVamana / StitchedVamana）。这是与 N 正交的"workload 触发型"质变——可在任何规模下发生。Microsoft 广告 A/B test 实证小区域（<1% share）gain 显著（+70-80% revenue）—— 证明 selectivity 极低时质变效应最大。

## Answer

### 第 9 质变维度的特征（NEW）

[per concepts/filtered-vamana.md "提出背景" + benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md]

**Filter selectivity 的质变阈值**：

| Selectivity | search-time post-processing | inline-processing | filter-aware build |
|---|---|---|---|
| 100pc（无 filter）| OK | OK | OK |
| 75pc-50pc | OK | OK | 显著更快 |
| **25pc** | **退化** | partial | **首选** |
| **1pc** | **失败**（recall < 40%）| partial | **唯一可行**（recall 90%+）|
| **<1%** | 完全失败 | 退化 | **唯一可行 + Microsoft A/B +70%/+80% gain** |

→ **selectivity ~25% 是质变阈值**——之上 search-time 路径 OK，之下必须 filter-aware build。

### 第 9 质变与其他维度的关系（NEW）

| # | 维度 | 触发 | PASE/ADBV 是否影响 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | n/a |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | n/a |
| 3 | 单机 SSD 触顶 | 规模 + 存储 | n/a |
| 4 | library → DBMS（cloud-native） | 工程形态 | n/a |
| 5 | out-of-place rebuild → in-place | update strategy | n/a |
| 6 | static index_type → adaptive across lifecycle | 索引设计哲学 | n/a |
| 7 | strong/eventual binary → tunable delta τ | consistency 模型 | n/a |
| 8 | vector-first DBMS → DB-extended | 架构起点 | 8a (OLAP) / 8b (OLTP) 子分化 |
| **9（NEW）** | **search-time filter → build-time filter-aware** | **filter selectivity 触发** | **正交于其他维度** |

### Filter-heavy production workload 的 8 + 9 双触发（NEW）

[per benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md "Microsoft 广告 A/B 实验"]

Microsoft 广告搜索 production 同时触发：
- **质变 4**（DBMS 形态）：动态广告 + 在线 ad serving
- **质变 9**（filter-aware build）：47 region filter + selective filter（<1% region 占 27 个）

→ **质变 9 上的 +70%/+80% revenue gain 数字**是 wiki 内首次"质变 = 显著业务收益"的实证。其他维度（5-8）都是技术 metric 提升；质变 9 直接转化为收入。

### 与之前 ingest 的演进

| | yang-2020-pase post | **gollapudi-2023-filtered-diskann post (NEW)** |
|---|---|---|
| 质变维度 | 8（含 8a/8b 子分化） | **9** |
| Filter selectivity 处理 | 隐含（5 plan / 4 plan / iterative） | **显式 search-time vs build-time 二分**（25pc 阈值）|
| Production 业务实证 | n/a | **+70%/+80% revenue 在 <1% selectivity 区**|
| Filter-aware build vs search-time | 隐含同等 | **search-time 在 1pc 失败** |

### 不算质变（参数微调）

- FilteredVamana α / R / L 调
- StitchedVamana R_small / R_stitched / L_small 调
- Beam width 4 vs 8 vs 16

### 已知盲区

- **多 filter conjunction（AND/OR/NOT）**：论文 §2 明确"future work"
- **HTAP DBMS（TiDB / SingleStore）+ filter-aware build**：仍未覆盖
- **2026 SIGMOD 跨 model 整合**：embedding lifecycle 是另一种质变
- **质变 10？**：filter set 演化（新增 filter 类型）触发何种质变 wiki 未覆盖

## Cited Pages

- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
- [systems/diskann.md](../../systems/diskann.md)
- [benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md](../../benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md)
