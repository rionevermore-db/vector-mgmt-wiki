---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是 proximity graph，工程上怎么选？"
date: 2026-05-08
phase: post
ingest-context: gollapudi-2023-filtered-diskann
wiki-pages-total: 46
cited-pages: [concepts/hnsw.md, concepts/vamana.md, concepts/filtered-vamana.md, systems/diskann.md]
cited-count: 4
---

# Post-snapshot (gollapudi-2023-filtered-diskann): hnsw-vs-nsg-selection

## TL;DR (delta from yang-2020-pase post)

**新增 graph-ANNS 选型维度：filter-aware build**。当 workload 含 attribute filter + 需要严格 latency budget，**FilteredVamana 是 wiki 内首个 filter-aware build graph 算法**——比 search-time HNSW + filter / NSG + filter 路径快 5-10× QPS @ 90% recall。**HNSW build 简单 + 增量 add ✓** 仍是无 filter 场景默认；FilteredVamana 把 Vamana 升级为 filter-aware + 同样支持 incremental add——成为 filter-heavy workload 的新默认选择。

## Answer

### Graph-ANNS 选型加新轴：filter-aware build（NEW）

[per concepts/filtered-vamana.md "与 wiki 已有 graph ANNS 的关系"]

| | HNSW | NSG | Vamana | **FilteredVamana** |
|---|---|---|---|---|
| 层数 | 多层 | 单层 | 单层 | 单层 |
| α 参数 | 隐式 1 | 隐式 1 | **可调** | 同 Vamana + filter-aware |
| 边构造维度 | geometric only | geometric only | geometric only | **geometric + label** |
| 增量 add | ✓ | ✗ | ✗ | **✓** |
| Filter-aware build | ✗ | ✗ | ✗ | **✓ 首个** |
| 推荐场景 | 通用、生态广 | 静态 + 高 recall | SSD-friendly | **filter-heavy + production** |

### Filtered-DiskANN A/B 实测的工程含义（NEW）

[per benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md "Microsoft 广告 A/B 实验"]

Microsoft 广告搜索 production：
- **+34.61% clicks, +48.95% revenue**（47 region filters）
- 小 region (<1% share)：**+70%/+80%**

→ 这意味着：**workload 含选择性 filter 时**，HNSW + filter post-processing 不只 latency 更高，**business metric 也显著差**——因 strict latency budget 下 post-processing 退化为 zero-result。

### 选型决策树（updated 加 filter 维度）

| 场景 | 推荐 | 备注 |
|---|---|---|
| 静态 + 高 recall | NSG | 论文实测最强 |
| 增量 add 无 filter | **HNSW** | 生态最广 |
| Filter-heavy + production latency 严 | **FilteredVamana / StitchedVamana** | filter-aware build + Microsoft A/B 实证 |
| Filter-heavy + streaming insert | **FilteredVamana** (StitchedVamana batch only) | incremental |
| Filter-heavy + max QPS | **StitchedVamana** | per-vertex 边集大但 QPS 高 2-7× |
| SSD-resident graph | **Vamana / Filtered-DiskANN** | small diameter |

### 与之前 ingest 的演进

| | yang-2020-pase post | **gollapudi-2023-filtered-diskann post (NEW)** |
|---|---|---|
| Graph 选型维度 | 增量 + 内存 + 性能 | **+ filter-aware build** |
| HNSW 在 filter 场景 | 仅 search-time post-processing | **post-processing 在 selectivity 1% 区间退化失败** |
| NSG / Vamana 演化 | NSG 不增量；Vamana SSD | **+ FilteredVamana 既增量又 filter-aware** |

### 与 Milvus / 其他 vector DBMS HNSW filter 处理对比

[per benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md "vs Milvus / Faiss / NHQ"]

Milvus HNSW + filter post-processing in [gollapudi-2023-filtered-diskann §A.2]：
- 实测 QPS < 300 across all datasets
- **比 Vamana 算法低 orders of magnitude**

→ 即使是 wiki 内最常用的"Milvus HNSW + filter"组合，在 filter-heavy + selective filter 场景下被 FilteredVamana 显著超越。但 Milvus HNSW 在**无 filter** 场景仍是默认推荐（Manu §5 实测 SIFT10M / Deep10M HNSW 最强）。

### Open / 未覆盖

- **HNSW + filter-aware build**：FilteredHNSW 是否可行？论文未尝试；推断可行（greedy search + 邻居选择都加 label intersection check）但 wiki zero coverage
- **NSG + filter-aware build**：同上
- **NHQ-NPG vs FilteredVamana 在 build 复杂度的差异**：[gollapudi-2023-filtered-diskann §A.1] 论文实测 KGraph 版本比 NHQ default 快 1 order of magnitude at 100 recall

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/vamana.md](../../concepts/vamana.md)
- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
- [systems/diskann.md](../../systems/diskann.md)
