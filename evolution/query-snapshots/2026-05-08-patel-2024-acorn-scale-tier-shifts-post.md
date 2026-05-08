---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-08
phase: post
ingest-context: patel-2024-acorn
wiki-pages-total: 48
cited-pages: [topics/index-selection.md, topics/attribute-filtering.md, concepts/acorn.md, concepts/filtered-vamana.md, benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md]
cited-count: 5
---

# Post-snapshot (patel-2024-acorn): scale-tier-shifts

## TL;DR (delta from gollapudi-2023-filtered-diskann post)

**第 9 质变（filter selectivity）扩展为第 10 质变（filter cardinality）**：之前 selectivity → search-time vs build-time 二分；ACORN 引入 **filter cardinality**（LCPS ≤1000 vs HCPS 10^8+）作新分裂点。LCPS 触发 filter-aware build（FilteredVamana 主场）；**HCPS 触发 predicate-agnostic indexing**（ACORN 唯一方案）。这两个质变维度 orthogonal——同 workload 下两者都可触发。

## Answer

### 十个质变点（updated）

| # | 质变点 | 维度 | 触发 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | ~10M |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | ~1B |
| 3 | 单机 SSD 触顶 | 规模 + 存储 | ~百亿+ |
| 4 | library → DBMS（cloud-native） | 工程形态 | dynamic + 分布式 + filter |
| 5 | out-of-place rebuild → in-place | update strategy | 持续 update + 资源约束 |
| 6 | static index_type → adaptive across lifecycle | 索引设计哲学 | SaaS 模式 / vendor 自动调优 |
| 7 | strong/eventual binary → tunable delta τ | consistency 模型 | vector DB 应用多样化容忍度 |
| 8 | vector-first DBMS → DB-extended | 架构起点 | 8a (OLAP) / 8b (OLTP) 子分化 |
| 9 | search-time filter → build-time filter-aware | filter selectivity | <25% selectivity 触发 |
| **10（NEW）** | **filter-aware build → predicate-agnostic build** | **filter cardinality** | **>1000 filters / 任意 operator** |

### 第 10 质变维度的特征（NEW）

[per concepts/acorn.md "LCPS vs HCPS 二分"]

| Filter cardinality | 触发条件 | 推荐方案 |
|---|---|---|
| ≤ 1000 equality（LCPS）| simple filter 场景 | **FilteredVamana / StitchedVamana** |
| 1000 - 10^6 mixed operators | 中等复杂度 web filter | ACORN-γ 或 FilteredVamana（lossy） |
| **10^8 - 10^11 + 任意 operator（HCPS）** | **真实 web search / regex / contains** | **ACORN-γ / ACORN-1（唯一方案）** |

→ 之前 wiki 隐含 ≤1000 filter 假设——ACORN 暴露 filter cardinality 维度的根本不同。

### 第 9 与第 10 质变的关系（NEW）

[per topics/attribute-filtering.md "LCPS vs HCPS"]

```
质变 9 (selectivity)  与  质变 10 (cardinality)  正交：

           filter cardinality
            ≤1000 (LCPS)     >10^8 (HCPS)
selectivity         
≥25%        post-filter OK    post-filter OK
            FilteredVamana    ACORN
            
<25%        FilteredVamana    ACORN
            (post-filter fail)  (post-filter fail)
```

→ 同 workload 下两者**都可触发**——例如真实 web search 同时是 HCPS（unbounded keyword）+ low selectivity（specific query）。post-filter 完全失败，必须 ACORN。

### Microsoft 广告场景跨两质变（NEW）

Microsoft 广告 47 region filter 实测（Filtered-DiskANN A/B test）：
- Cardinality: 47（**LCPS**——低 cardinality）
- Selectivity: 1pc-20pc（**low selectivity**——大量 filter 严的 query）
- 触发 9 + 10 中**仅质变 9**（filter selectivity 触发）；质变 10（cardinality）不触发

LAION 25M scenario：
- Cardinality: 10^11（**HCPS**）
- Selectivity: varies（0.056-0.13 avg, 1pc-99pc range）
- 触发 9 **AND** 10——FilteredVamana fail，ACORN 唯一可行

→ ACORN 价值最大场景是 **double 触发**（HCPS + selective filter）——是 LAION-style web search workload 特征。

### 与之前 ingest 的演进

| | gollapudi-2023 post | **patel-2024-acorn post (NEW)** |
|---|---|---|
| 质变维度数 | 9 | **10** |
| Filter dimension 子分化 | selectivity 唯一 | **+ cardinality 第二个**（与 selectivity 正交） |
| HCPS 概念出现 | 未涉及 | **首次出现**（10^8-10^11 unique predicates）|
| Production A/B 触发模式 | LCPS 主场 | LCPS 仍是 production 标准；HCPS 学术 only |

### 不算质变（参数微调）

- ACORN γ / M_β 调
- ACORN-1 vs ACORN-γ 选择
- LCPS 内 cardinality 100 vs 1000

### 已知盲区

- **HCPS + 千亿 scale**：完全空白
- **HCPS + SSD**：完全空白
- **质变 11+**：未来 ingest 是否暴露更多维度（如 multi-modal embedding 或 video temporal）
- **2026 SIGMOD 跨 model 整合**：embedding lifecycle 是另一种事件型质变

## Cited Pages

- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
- [concepts/acorn.md](../../concepts/acorn.md)
- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
- [benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md](../../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md)
