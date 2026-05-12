---
title: Range-Filtering ANN 三条路线 (SeRF / iRangeGraph / UNIFY)
type: concept
sources: [zuo-2024-serf, xu-2024-irangegraph, liang-2024-unify]
related: [
  ../topics/attribute-filtering.md,
  ../concepts/acorn.md,
  ../concepts/filtered-vamana.md,
  ../concepts/frontier-2025-distributed-vector-search.md,
  ../concepts/hnsw.md,
]
created: 2026-05-12
updated: 2026-05-12
---

# Range-Filtering ANN 三条路线 (SeRF / iRangeGraph / UNIFY)

**TL;DR**: 3 篇 2024 SIGMOD / PVLDB paper 解 **range-filter ANN (RF-ANN)** 同一问题——给定 vector + numeric range `[l, h]` (e.g. price, date, timestamp), 找 attribute 在 range 内的近邻——3 种不同 attack:
**(1) SeRF** (Zuo et al. SIGMOD 2024 SJTU + Microsoft + Alibaba): **compression-first** segment graph, n ANN indexes lossless 压缩到 1 个 segment graph (**Ω(n) 索引节省**), 2D segment graph 处理任意 range, O(n log n) avg index size, SOTA 在 0.3%-50% range;
**(2) iRangeGraph** (Xu et al. PVLDB 2024): **dynamic assembly** per-query, 预建 moderate 数量 elemental graphs, query 时 segment tree intersect target range → 动态组合 composite subgraph + greedy search;
**(3) UNIFY** (Liang et al. PVLDB 2024 SJTU + Tencent): **unified hybrid** — SIG/HSIG (Segmented Inclusive Graph + HNSW-inspired hierarchical) + skip list connections + global edge masking + **range-aware strategy selection** (3 strategy A/B/C 按 query range 自动选), **0.1%-100% range, 支持 incremental update**, 2.29× over SOTA.
本 bundle 填 wiki **numeric range filter ANN 完全空白** (之前 wiki 仅 [ACORN](./acorn.md) categorical filter / [Filtered-Vamana](./filtered-vamana.md) label filter, PathFinder 提及多属性但 range 维 zero source-backed).

## 提出背景

[per sources/papers/liang-2024-unify.pdf §1]

**RF-ANN 问题定义**: dataset of n attributed vectors (vector + numeric attribute), query = (q, [l, h], k), 返 attribute ∈ [l, h] 的 top-k nearest neighbors.

**应用场景**:
- Amazon 同价格区间相似商品
- Google Image Search 同时段同主题
- 日期 / 价格 / 数量 range filter 是产品 search 默认

**Strategy A (Pre-Filtering)**: B-tree 过滤 attribute → linear scan filtered set or PQ codes
- **vendor 例**: Alibaba ADBV / Milvus 分区
- **trade-off**: 小 range 高效, 大 range 退化 (scan cost 线性)

**Strategy B (Post-Filtering)**: ANNS 找 k' candidates → filter by attribute → top-k 真结果
- **vendor 例**: Vearch / NGT + HNSW / IVF-PQ
- **trade-off**: 大 range 高效, 小 range 候选不足 → 必须放大 k' → 性能差

**Strategy C (Hybrid Filtering)**: 单 data structure 同时 index vector + attribute
- **paper 例**: SeRF (Strategy C 第一个 PG-based 实现)
- **trade-off**: 中等 range 高效, 但 SeRF 在 < 0.3% 和 > 50% range 性能不如 A/B

## 三论文 axis 三角

### 路线 1: Compression-First — **SeRF (Zuo et al. SIGMOD 2024)**

[per sources/papers/zuo-2024-serf.pdf + 公知]

- **核心创新**: 把 n 个 ANN index (per range) 压缩到 **1 个 segment graph**, index time + size 与 single ANN index 同——**Ω(n) 节省**
- **关键技巧**: 
  - **Range-aware edges**: 数据按 attribute 升序 sort, 增量 insert 到 graph
  - **Range 隐式 encode by insertion order**——相邻 attribute value 的 vector 在 graph 中相连
- **2D segment graph**: 处理一般 range query (不仅 prefix / suffix), avg index size O(n log n), 打破 quadratic barrier
- **结果**: SOTA 在 0.3% ~ 50% range
- **局限** (UNIFY §1 cite): "lags behind Strategy A and B for smaller and larger ranges; lacks support for incremental data insertion"
- **本质**: 把"建 n 个 dedicated index"压成"建 1 个 + clever encoding"

### 路线 2: Dynamic Assembly — **iRangeGraph (Xu et al. PVLDB 2024)**

[per sources/papers/xu-2024-irangegraph.pdf + 公知]

- **核心创新**: **不预建** all possible range 的 graphs, 而是**预建 moderate 数量 elemental graphs** + query 时**动态拼接 composite subgraph**
- **关键技巧**:
  - **Segment tree on numeric attribute**: 每 tree node 一个 elemental graph (该 node attribute range 内的 PG)
  - **Query 时**: segment tree 找 intersecting tree nodes → retrieve elemental edges → assemble composite subgraph → greedy search
- **本质**: 复用 segment tree (经典 1D range data structure) + per-segment local PG, "拼接而非压缩"
- **优势**: query-time flexibility, 任意 range 都能合成对应 subgraph
- **代价**: elemental graphs 总 storage 大于 SeRF, query-time assembly cost (vs SeRF query-time 是单 graph traversal)

### 路线 3: Unified + Range-Aware Strategy Selection — **UNIFY (Liang et al. PVLDB 2024)**

[per sources/papers/liang-2024-unify.pdf §1]

- **核心创新**: 把 Strategy A/B/C **统一到单 PG-based 索引**, 同时**按 query range 自动选 strategy** + 支持 incremental update
- **4 关键技术**:
  1. **SIG (Segmented Inclusive Graph)**: novel graph family, dataset 按 attribute segment, **PG of 任意 segment 组合 都是 SIG 的 sub-graph**——保证 Strategy C 高效 hybrid filtering
  2. **HSIG (Hierarchical SIG)**: HNSW-inspired hierarchical 变体, **O(log(n')) RF-ANN time complexity** (n' = 涉及 segment 内 object 数)
  3. **Skip list fusion**: 经典 attribute index (skip list) connections **融入 HSIG 层次结构**——用 skip list navigate 实现 efficient pre-filtering (Strategy A)
  4. **Global edge masking**: bitmap-mark 全局 HNSW edges 在每 HSIG node 内——用 global HNSW navigate 实现 efficient post-filtering (Strategy B)
- **Range-aware strategy selection (O4)**: 启发式——
  - 小 range (Y ≤ τ_A) → Strategy A
  - 大 range (Y ≥ τ_B) → Strategy B
  - 中等 range (τ_A < Y < τ_B) → Strategy C
  - τ_A, τ_B 通过历史 query 统计学习
- **结果**: 0.1%-100% range 全部覆盖, up to **2.29× over SOTA**, **支持 incremental insertion**
- **本质**: 不是"选哪种 strategy 最好", 而是"**unified index + 按 range 自适应**"

## 三 axis 对比表

| Axis | SeRF (SIGMOD 2024) | iRangeGraph (PVLDB 2024) | UNIFY (PVLDB 2024) |
|---|---|---|---|
| 核心 philosophy | Compression: n indexes → 1 segment graph | Dynamic assembly per-query | Unified PG + strategy auto-select |
| Index 结构 | Segment graph (1D + 2D 变体) | Elemental graphs + segment tree | SIG / HSIG + skip list fusion + edge masking bitmap |
| Query time complexity | Single graph traversal | Segment tree lookup + assembly + greedy | O(log(n')) on HSIG |
| Index storage | O(n log n) avg | Moderate × n (elemental count tunable) | Comparable to HNSW + segment overhead |
| Range sweet spot | 0.3% ~ 50% | Wide (任意 range) | **0.1% ~ 100% 全覆盖** |
| Small range performance | 不如 Strategy A | Good (segment tree drops to small subtree) | **Best (Strategy A 路径)** |
| Large range performance | 不如 Strategy B | Good (segment tree covers most data) | **Best (Strategy B 路径)** |
| Incremental update | **不支持** | Doc 未明示 | **支持** |
| Best at | Compression, static workload | Flexibility | Production with mixed range distribution |

## 与同类 concept 对比

| 与本 bundle 对比 | 共同 | 差异 |
|---|---|---|
| [ACORN](./acorn.md) | filter-aware ANN | ACORN = **predicate-agnostic** for **categorical/high-cardinality** filter; 本 bundle = **numeric range** filter specialized——两 axis 互补 |
| [Filtered-Vamana](./filtered-vamana.md) | filter-aware graph | Filtered-Vamana = **label filter** (categorical equality), 本 bundle = **range filter** (numeric continuous); Filtered-Vamana 限 ≤1000 labels |
| [PathFinder (2025)](./frontier-2025-distributed-vector-search.md) | filter-aware ANNS optimizer | PathFinder = **多 attribute 多 filter type optimizer** (cost-based, RDBMS-style), 本 bundle = **单 numeric range filter algorithm core**; PathFinder 在更高 abstraction layer, 本 bundle 提供 specific filter type 的 algorithm primitive |
| [Attribute Filtering 五策略](../topics/attribute-filtering.md) | filter-vector hybrid | 五策略框架是 **filter execution strategy** 抽象, 本 bundle 是该框架在 numeric range filter 上的 algorithm-level 三 instantiation |

## Wiki 影响

### 关键贡献到 wiki 知识图

1. **Numeric range filter ANN 完全空白填补**——之前 wiki 仅 [ACORN](./acorn.md) (predicate-agnostic categorical) + [Filtered-Vamana](./filtered-vamana.md) (label filter), **numeric range filter zero source-backed**. 本 bundle 提供 3 paper 完整覆盖.

2. **3 类 range filter algorithm philosophy 显式化**:
   - **Compression-first** (SeRF)——n indexes → 1 segment graph
   - **Dynamic assembly** (iRangeGraph)——预建 elemental + query 时拼
   - **Unified + strategy auto-select** (UNIFY)——单 index + 按 range 选 A/B/C strategy
   是 wiki 内 **filter-ANN 设计空间 3-way taxonomy**, 与 [filter-aware ANN attribute-filtering 五策略](../topics/attribute-filtering.md) 框架在不同 axis.

3. **Strategy A/B/C 经典分类 source-backed**: 之前 wiki 五策略框架是综合, 现在 UNIFY paper §1 提供**经典 3-strategy taxonomy 直接 source citation**:
   - A = Pre-filtering (Alibaba ADBV / Milvus)
   - B = Post-filtering (Vearch / NGT)
   - C = Hybrid (SeRF / UNIFY)

4. **Range-aware strategy selection 是 PathFinder 的 algorithm-level 同源**: PathFinder ([Ingest #13](./frontier-2025-distributed-vector-search.md)) 在 query optimizer 层选 attribute-specific index, UNIFY 在 algorithm 层按 range size 选 A/B/C strategy——两者都是 "cost-based filter strategy selection", 不同 abstraction layer. wiki cross-link.

5. **Production vendor 对照**:
   - Snowflake Cortex `@gte` / `@lte` numeric filter (Ingest #16 deepen) → 实现可能基于 Strategy A pre-filter (黑盒)
   - Databricks filter on numeric column → 同样 Strategy A 路径 (大概率)
   - **没有 vendor 公开实现 SeRF / UNIFY-style hybrid C**——是 production "vendor implementation lag" 例.

6. **talk SIGMOD 2026 multimodal-bench-methodology query 直接相关**: 空间 + vector hybrid 中 **空间 = 2D range filter** (lat/lon range), 本 bundle 是 **空间 filter ANN 的 building block**.
   - 1D range (本 bundle) → 2D range (空间 box) 推广是 follow-up direction
   - SeRF 2D segment graph 已是 step towards 2D range

## Open Questions

- **多 attribute range 联合 filter**: 本 3 paper 都假设单 attribute numeric range. Multi-attribute range (e.g. price ∈ [10, 50] AND date ∈ [Jan, Mar]) 是 production 主流, 但 algorithm 怎么扩展? Liang et al. arxiv 2602.15488 "Multi-Attribute Range Filter" 是后续, 可未来 ingest.
- **GPU-accelerated range filter ANN**: arxiv 2604.20121 "GPU-Accelerated Multi-Attribute Range Filtered ANN" 暗示 GPU + range filter 是 frontier, 但 paper 在未来日期 (arxiv 2604), 推 future direction.
- **UNIFY incremental update 性能 vs static index**: 论文支持但 incremental cost 数据点缺. Production high-throughput 应用是否仍能 maintain SIG/HSIG invariant?
- **空间 (lat/lon 2D range) → 本 bundle 推广**: SeRF 2D segment graph 是 1D 推广, 但 lat/lon 是 2D continuous space, R-tree-style 适用; 与 PathFinder filter optimizer 联合是 talk 主题的 algorithm 层补足.
- **range filter recall 与 range size 的实测函数**: 3 paper 都说 range-size-dependent performance, 但 recall 函数 (recall vs range size curve) 缺标准 benchmark.

## Cited by

(将随未来 ingest 累积)
