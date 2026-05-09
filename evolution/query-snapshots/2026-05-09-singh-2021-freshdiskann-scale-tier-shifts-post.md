---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-09
phase: post
ingest-context: singh-2021-freshdiskann
wiki-pages-total: 61
cited-pages: [systems/freshdiskann.md, concepts/freshvamana.md, topics/in-place-vs-out-of-place-updates.md]
cited-count: 3
---

# Post-snapshot (singh-2021-freshdiskann): scale-tier-shifts

## TL;DR (delta from wang-2024-starling post)

**FreshDiskANN 把第 5 质变维度（out-of-place rebuild → in-place）从 cluster-only 升级为双路径**——之前 wiki 视 in-place 为 cluster path 唯一（SPFresh / LIRE）；FreshDiskANN 揭示 graph path 也有等价工业方案（FreshVamana + StreamingMerge），但**内存预算 30× 差距**（SPFresh 4 GB vs FreshDiskANN 128 GB for 1B）。**第 5 维度细化为 5a (cluster path) vs 5b (graph path)**，类似第 4 维度的 4a/4b 细分。**核心洞察**：α 参数从"性能调节钮"升级为 graph fresh-ANNS 的**必要条件**——这是 graph ANN 文献的根本 conceptual shift（α=1 必失败，α>1 必成功）。

## Answer

### 十一个质变点 + sub-dimensions（updated）

| # | 质变点 | 维度 | 触发 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | ~10M |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | ~1B |
| 3 | 单机 SSD 触顶 | 规模 + 存储 | ~百亿+ |
| 4 | library → DBMS（cloud-native） | 工程形态 | dynamic + 分布式 + filter |
| 4a (sub) | single-server DBMS（DiskANN/SPANN） | 部署假设 | single-server 大磁盘 |
| 4b (sub) | segment-level DBMS（Starling/Milvus） | 部署假设 | vector DBMS 工程主流 |
| **5** | **out-of-place rebuild → in-place** | update strategy | 持续 update + 资源约束 |
| **5a (NEW sub)** | **in-place via cluster path（SPFresh/LIRE）** | 内存预算 ≤ ~10 GB for 1B | cost-sensitive streaming |
| **5b (NEW sub)** | **in-place via graph path（FreshDiskANN/FreshVamana）** | 内存预算 ~128 GB for 1B | recall-sensitive streaming |
| 6 | static index_type → adaptive across lifecycle | 索引设计哲学 | SaaS / vendor 自动调优 |
| 7 | strong/eventual binary → tunable delta τ | consistency 模型 | vector DB 应用多样化 |
| 8 | vector-first DBMS → DB-extended | 架构起点 | 8a OLAP / 8b OLTP |
| 9 | search-time filter → build-time filter-aware | filter selectivity | <25% selectivity 触发 |
| 10 | filter-aware build → predicate-agnostic build | filter cardinality | >1000 filters / 任意 operator |
| 11 | TopK interface → Iterator + RM | query 复杂度 | multi-column / range / Join |
| 11b | Biased PQ → Unbiased + sharp error bound | quantizer 自身的理论保证 | 同样攻击 K' 问题但 quantizer 层 |

### 第 5 维度的两条路径（NEW sub-dimensions）

[per topics/in-place-vs-out-of-place-updates.md "In-place 的两条路径"]

| 决策因素 | **5a Cluster path (SPFresh)** | **5b Graph path (FreshDiskANN)** |
|---|---|---|
| 论文年份 | 2023 SOSP | **2021 arXiv** (preprint，早 2 年) |
| 团队 | Microsoft Research（Yuming Xu 等） | Microsoft Research India + CMU（Singh + Subramanya 等） |
| 索引基础 | SPANN inverted file | Vamana graph |
| 必要条件 | 2 NPA conditions | **α-RNG property (α > 1) — 算法层 conceptual shift** |
| 增量机制 | 5 operations | Insert + Lazy Delete + Batch Consolidate + StreamingMerge |
| 内存预算（1B） | **~4 GB** | ~128 GB |
| Update cost | 极低 (per-cluster locality) | 中 (StreamingMerge 4.4h on 800M / 30M change) |
| Recall 上限 | ~95% (cluster centroids 约束) | **~98%+ (graph 高 recall ceiling)** |
| 实证 | 100 days × 1% daily | 50 cycles × 5%/10%/50% |

→ 两条路径 **memory budget vs recall ceiling** trade-off——cluster path 30× 经济，graph path recall 上限更高。

### α 参数的"再发现"（NEW conceptual shift）

[per concepts/freshvamana.md "α-RNG Property"]

之前 wiki 内对 Vamana α 的理解：α > 1 是 build-time 性能 trade-off（更稠密 graph → 更短 search path）。

**FreshDiskANN 的 conceptual shift**：α > 1 不仅是性能优化，**更是 graph fresh-ANNS 的必要条件**。

```
Static ANNS:
  α=1 (HNSW/NSG default): OK, 性能 best
  α=1.2 (Vamana default): OK, 性能 slight cost

Streaming ANNS (50 cycles 5% delete+insert):
  α=1: FAIL (recall 95% → 89%)
  α=1.2: PASS (recall stable 95%+)
```

→ 这暗示 **HNSW / NSG 的 fresh-ANNS 失败不是 graph 算法本征问题**——而是 RobustPrune 哲学（α=1 too aggressive）问题。理论上 HNSW / NSG 加 α-augmented patch 都可以 streaming-ready，但工业实现没人做。

### 与之前 ingest 的累积演进

| | wang-2024-starling post | **singh-2021-freshdiskann post (NEW)** |
|---|---|---|
| 质变维度数 | 11 (with 11b sub + 4a/4b sub) | **11 (with 11b + 4a/4b + 5a/5b sub)** |
| 第 5 维度 | 单一 in-place | **细化为 5a cluster vs 5b graph** |
| α 参数地位 | build-time trade-off | **streaming 必要条件（fresh-ANNS conceptual shift）** |
| Wiki 内 graph-path streaming 实证 | n/a | **首次明确（FreshDiskANN）** |
| Streaming workload 选择 | 仅 cluster path | **+ graph path（高 recall budget）** |

### 不算质变（参数微调）

- FreshVamana α=1.2 vs α=1.3
- StreamingMerge β=8 iterations
- TempIndex 大小 5M vs 30M points threshold

### 已知盲区

- **维度 5a vs 5b head-to-head**：FreshDiskANN 2021 / SPFresh 2023 时间错位，论文相互不直接对比；wiki 内 head-to-head 实证空白
- **HNSW / NSG α-augmented patch (5b 扩展)**：理论可行未做
- **Streaming + filter heavy (维度 5 + 9 + 10)**：完全空白
- **Streaming + iterator (维度 5 + 11)**：理论可行未实证
- **Streaming + RaBitQ (维度 5 + 11b)**：理论可行未实证

## Cited Pages

- [systems/freshdiskann.md](../../systems/freshdiskann.md)
- [concepts/freshvamana.md](../../concepts/freshvamana.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
