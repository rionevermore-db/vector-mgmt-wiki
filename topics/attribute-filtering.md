---
title: Attribute Filtering（向量+属性混合查询）
type: topic
sources: [wang-2021-milvus, douze-2024-faiss-library, gollapudi-2023-filtered-diskann, patel-2024-acorn, zhang-2023-vbase, qdrant-docs]
related: [../systems/milvus.md, ../systems/faiss.md, ../systems/diskann.md, ../systems/pinecone.md, ../systems/analyticdb-v.md, ../systems/pase.md, ../systems/vbase.md, ../systems/qdrant.md, ../concepts/product-quantization.md, ../concepts/vgpq.md, ../concepts/filtered-vamana.md, ../concepts/acorn.md, ../concepts/hnsw.md, ../concepts/relaxed-monotonicity.md, ./index-selection.md, ./topk-vs-iterator-model.md, ./vector-range-query.md, ../benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md, ../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md, ../benchmarks/vbase-8queries-recipe1m.md, ../benchmarks/vectordbbench.md, ../benchmarks/big-ann-benchmarks.md, ./ann-benchmarking-methodology.md, ../concepts/range-filter-ann-2024.md]
created: 2026-05-07
updated: 2026-05-21 (benchmark-trio: VDBBench filtering case + big-ann filter track 作公平横测锚点)
---

# Attribute Filtering

**TL;DR**: "找与 query 最相似的 top-k 向量，**且满足属性条件 C_A**"——是工业向量检索最常见的混合查询场景（电商按价格、推荐按地域、监控按时间）。两条主流路线：**vector-first**（先 ANN 找候选再过滤）和 **attribute-first**（先按属性筛再扫向量）。[Faiss](../systems/faiss.md) 用 IDSelector callback、AnalyticDB-V 用 cost-based、Milvus 提出 partition-based（按高频 filter 属性预分区，比 cost-based 快 13.7×）。**Filtered-DiskANN** 是后继研究方向。

> **公平横测 filter 性能怎么做**：本 page 给的是策略 taxonomy;**跨厂商 / 标准化硬件的 benchmark 锚点**见 [benchmarks/vectordbbench.md](../benchmarks/vectordbbench.md)（Filtering case：int-based + label-based，30+ vendor 横测）与 [benchmarks/big-ann-benchmarks.md](../benchmarks/big-ann-benchmarks.md)（NeurIPS 2023 **Filter track**：YFCC 10M + tag 过滤，FAISS baseline 3,200 QPS，标准化 Azure 硬件）。方法论(必须固定 embedding + 标明 A/B/C strategy + selectivity sweep + 防 vendor 调参不对称)见 [topics/ann-benchmarking-methodology.md](./ann-benchmarking-methodology.md)。

## 问题陈述

形式化：top-k 查询 + 属性约束 `C_A`（如 `price >= 100 AND price <= 500`）+ 向量约束 `C_V`（如 `cosine(q, x) > 0.7` 或 top-k）。两个 selectivity 维度：
- **C_A selectivity**：满足属性条件的 entity 占比
- **C_V selectivity**：top-k 在原 entity 总集合的占比

不同 selectivity 组合下最优策略不同。

## 工业方案对比

[wang-2021-milvus §4.1, Fig 4]：

| 策略 | 描述 | 适用 | 缺点 |
|---|---|---|---|
| **A: attr-first-vector-full-scan** | 先按 C_A 用 B-tree 索引 / 跳表 / 二分查 → 全扫候选向量 | C_A **极选择性**（候选数 << N） | 候选多时退化为线扫 |
| **B: attr-first-vector-search-bitmap** | 先按 C_A 拿候选 entity ID → bitmap → 跑 ANN 时 bitmap 检查每个候选向量是否满足 | C_A 或 C_V 都 **moderate selectivity** | bitmap 上的 ANN 扫描可能跳过 partial match |
| **C: vector-first-attr-full-scan** | 先跑 ANN 拿 θ·k 个候选（θ>1）→ 全扫验证 C_A | C_V **极选择性** | C_A 不严时退化为大量无效 vector search |
| **D: cost-based** | 基于代价估计在 A/B/C 中选——[AnalyticDB-V](../systems/analyticdb-v.md) 方案 [per wei-2020-analyticdb-v §5] | 通用 | 最优策略选对，但**单 query 内仍受 A/B/C 自身限制** |
| **E: partition-based**（Milvus 新提） | 按高频被 filter 的属性预分区；query 时只扫 range 重叠的分区，**range 完全 cover 时跳过 C_A check** | 高频固定属性、能预知 filter 维度 | 需预分区设计；新加属性需重组 |

[wang-2021-milvus Fig 14-15]：strategy E 比 strategy D 快 **up to 13.7×**；selectivity 越细，partition-based 优势越大。

## Milvus partition-based 的具体实现

[wang-2021-milvus §4.1] 例：filter 属性 = 'price'，dataset 切到 5 个 partition：

```
P₀ [1,    100]
P₁ [101,  200]
P₂ [201,  300]
P₃ [301,  400]
P₄ [401,  500]
```

Query `C_A = [50, 250]`：
- P₀: range overlap → 仍需 check C_A
- P₁: **range fully covered** → **skip C_A check**，直接 ANN
- P₂: overlap → check C_A
- P₃ / P₄: 不 overlap → skip 整个 partition

ρ（partition 数）经验取 ~1M vec/partition；1B 数据 ~1000 partition。论文 §4.1 末尾承认这是"interesting future work"用 ML 自动学习最优 partition。

> **wiki 解读**：partition-based 把"filter selectivity"前置到**数据布局**，而不是 query 时用代价模型。本质上是把过滤约束的语义作为 storage layout 的一部分——和经典 OLAP 的 Z-order / partition pruning 同源。

## 现有 wiki 系统的属性过滤现状

| 系统 | 支持方式 | 强度 |
|---|---|---|
| **[Faiss](../systems/faiss.md)** | `IDSelector` callback / `bow_id_selector` bit-signature [per systems/faiss.md] | **基础**——library 级别，需用户手写 selector |
| **[Milvus](../systems/milvus.md)** | 5 策略 + cost-based 自动选 + partition-based [wang-2021-milvus] | **DBMS 级别原生支持** |
| **[Pinecone](../systems/pinecone.md)** | metadata filtering + namespace 隔离 + filterable schema fields [per pinecone-docs] | **SaaS 级别原生支持**；具体实现算法不公开 |
| **[AnalyticDB-V](../systems/analyticdb-v.md)** | 4 plan CBO（Plan A brute-force / B PQ Knn Bitmap Scan / C VGPQ Knn Bitmap Scan / D VGPQ Knn Scan + filter）+ accuracy-aware 超参 grid search [per wei-2020-analyticdb-v §5] | **OLAP DB 级别**——SQL 接口；与 Milvus 5 plan / Pinecone metadata 同代但路径不同（OLAP-extended vs vector-first）|
| **[PASE](../systems/pase.md)** | iterative search via PG `amgettuple` interface——增量 fetch top-K from vector index + check WHERE 子句，自然 short-circuit [per yang-2020-pase §2.6] | **OLTP RDBMS 级别**——PG 内核扩展；与 ADBV 4-plan CBO 是 compound query 的两种工程哲学（前置 cost model vs 增量 iterative）|
| **[DiskANN](../systems/diskann.md)** | **Filtered-DiskANN** = [FilteredVamana / StitchedVamana](../concepts/filtered-vamana.md)（WWW 2023）已正式 ingest——filter-aware **build** + filter-aware search；Microsoft sponsored ads production +35-49% gain | **首个 filter-aware build 工业系统** |
| **[SPANN](../systems/spann.md)** | 论文未涉及 | 弱 |
| Faiss-IDSelector vs Milvus | Faiss IDSelector 走 strategy B (bitmap)；Milvus 把它泛化为 5 策略 + 自动选择 | Milvus 完整覆盖 Faiss 思路 |
| **[VBASE](../systems/vbase.md)** | **Iterator + [Relaxed Monotonicity](../concepts/relaxed-monotonicity.md)**——绕开 K' 预测，单 column iterator + filter check + RM Phase 2 自动停 [zhang-2023-vbase §4.4] | **新范式**——所有上述策略都基于 TopK 接口（A-E 都是"先取候选再 filter"的不同变体）；VBASE 直接换接口让 K' 不再需要预测。详见 [topics/topk-vs-iterator-model.md](./topk-vs-iterator-model.md) |
| **[Qdrant](../systems/qdrant.md)** | **Filterable HNSW**——HNSW graph + payload-aware extra edges (per indexed field)；v1.16.0 集成 [ACORN](../concepts/acorn.md) algorithm 作 fallback（多 strict filter combination 或大量 soft-deleted points）[per sources/docs/qdrant/manage-data/indexing.md] | **HNSW 路径 filter-aware build**——与 [FilteredVamana](../concepts/filtered-vamana.md) Vamana base 平行；**首个 ACORN production 落地**。Trade-off: extra edges 占额外存储；多 payload index 组合不能 cover 所有 filter combination，故 ACORN fallback |

[per systems/faiss.md §Open Questions] Faiss 论文 §5.2 明确把 OOD-DiskANN、**Filtered-DiskANN** 列为 frontier。**Filtered-DiskANN [gollapudi-2023-filtered-diskann] 已 ingest**——是 Faiss frontier flag 的具体答案：filter-aware graph build + per-filter medoid + filter-aware RobustPrune。Milvus 的 partition-based 是 search-time 工业 solution；Filtered-DiskANN 是 build-time 学术领先方案；两者哲学不同（详见 [concepts/filtered-vamana.md "与其他 filtered ANNS 方法对比"](../concepts/filtered-vamana.md)）。

## Open Questions

- **Selectivity 阈值的 cutoff**：Faiss 用经验阈值 ~3×10⁻⁴ 切 vector-first vs attr-first；理论上是否可学习？[per topics/index-selection.md Open Q] 已 flag
- **Schema migration**：partition-based strategy E 在新加 filter 属性时需重组数据。工业系统（Milvus / Pinecone / Weaviate）的 schema migration workflow wiki 未覆盖
- **Multi-attribute filtering**：单属性 partition 容易，多属性（price + region + category）联合 partition 是 NP-hard 问题——Milvus 论文未解决
- **Dynamic data 下的 filtering**：LSM segment 持续 merge 时 partition 边界如何漂移？[wang-2021-milvus §2.3, §4.1] 没把两者完全连接
- **Embedding 与属性的语义关联**：vector 与 attribute 是否独立分布的假设——例如商品图片 embedding 与 price 不独立时，partition-based 假设失效
- **Top-k aggregation 在 attribute filtering 上的扩展**：[multi-vector queries](./multi-vector-queries.md) 的 vector fusion / iterative merging 未与 filter 联合讨论
- **A-E 五策略与 Iterator 范式的关系**：[VBASE](../systems/vbase.md) 显示 Iterator + RM 直接绕开五策略选择问题；A-E 可视为 TopK 框架内的 best engineering，但不及范式转换。Milvus / ADBV / PASE 是否能在保持现架构的前提下增量集成 RM iterator？[per topics/topk-vs-iterator-model.md] 是开放

## Cited Pages

- [systems/milvus.md](../systems/milvus.md)
- [systems/faiss.md](../systems/faiss.md)
- [systems/diskann.md](../systems/diskann.md)
- [topics/index-selection.md](./index-selection.md)
