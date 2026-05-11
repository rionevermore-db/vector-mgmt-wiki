---
title: ACORN（Predicate-Agnostic HNSW Filtered ANNS）
type: concept
sources: [patel-2024-acorn, qdrant-docs, weaviate-docs]
related: [./hnsw.md, ./filtered-vamana.md, ../topics/attribute-filtering.md, ../systems/qdrant.md, ../systems/weaviate.md, ../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md]
created: 2026-05-08
updated: 2026-05-11 (Weaviate as 2nd ACORN production case)
---

# ACORN

**TL;DR**: Stanford / DBOS / Berkeley 2024 SIGMOD 论文，**predicate-agnostic** filtered ANNS——构造时**不需要知道 query 会用什么 predicate**。建在 [HNSW](./hnsw.md) 上，通过**predicate subgraph traversal** 在 search 时 emulate "oracle partition index" 的 sublinear 性能。两变体：**ACORN-γ**（高性能版，construction 时每 node 收 M·γ candidate）+ **ACORN-1**（低 construction 开销版，靠 search-time 2-hop neighbor expansion 近似 ACORN-γ）。论文核心 critique：[FilteredVamana / StitchedVamana](./filtered-vamana.md) 限 ≤1000 equality filters，**HCPS 高基数 predicate set（10^8-10^11 unique predicates）**完全失败；ACORN 在 LCPS 上 2-10× faster than Filtered-DiskANN，在 HCPS 上 30-50× faster than HNSW post-filtering，25M LAION dataset >1000× higher QPS。开源 [stanford-futuredata/ACORN](https://github.com/stanford-futuredata/ACORN)。[patel-2024-acorn §1, §5-7]

## 提出背景

[patel-2024-acorn §1, §3]

ACORN 论文 attack 三条现有路线：

1. **Pre-filtering**（Milvus / AnalyticDB-V partial）：linearly 扫 X_p，**O(sn + K)** 复杂度——大数据 / 中等 selectivity 都退化
2. **Post-filtering**（FAISS-IVF, Pinecone hybrid, Weaviate, AnalyticDB-V partial, Milvus[62]）：标准 ANN → filter 后处理；论文论证 **negative query correlation** 时退化到 O(n)
3. **Specialized indices**（[FilteredDiskANN](./filtered-vamana.md), HQANN, NHQ）：限制 predicate set ≤1000 + equality only + single structured attribute——**真实应用 predicate 通常 unbounded + 复杂 operator**（regex / contains / between / OR）

**ACORN 论点**：predicate set 在 indexing 时**不知道 / 不应该限定**——所谓 "predicate-agnostic indexing"。

## Query Correlation 形式化（§3.1）

[patel-2024-acorn §3.1, Fig 2]

定义 query correlation `C(D, Q)`：

```
C(D, Q) = E_{(x_i, p_i) ∈ Q} [E_{R_i} [g(x_i, R_i)] − g(x_i, X_{p_i})]
```

其中 `g(x, S) = min_{y ∈ S} dist(x, y)`，R_i 是从 X 均匀采样的 |X_{p_i}| 大小集合。

```
positive correlation: query 倾向接近 target subset → post-filtering OK
no correlation: query 与 target 独立 → post-filtering 部分 OK
negative correlation: query 与 target anti-cluster → post-filtering 退化为 O(n)
```

→ **post-filtering 在 negative correlation 下完全失败**——这是论文 attack post-filtering 的核心 angle。

## LCPS vs HCPS 二分（§7.1）

[patel-2024-acorn §7]

**LCPS**（Low Cardinality Predicate Set）：predicate set 小（论文实验取 12）；FilteredVamana / NHQ / HQANN 适用：
- SIFT1M (random int label, cardinality 12)
- Paper (random int label, 12)

**HCPS**（High Cardinality Predicate Set）：predicate 数量爆炸；现有 specialized indices **完全无法 build**：
- TripClick: clinical area (~28 unique values) × publication year (~120 years) × **contains operator** → **>10^8 unique predicates**
- LAION 1M / 25M: **>10^11** predicates（regex match 30 token + contains keyword list）

> "**FilteredDiskANN** ... fail because they assume are unable to handle the high cardinality query predicate sets and non-equality predicate operators"——论文 §7.3 直接论证。

## Predicate Subgraph Traversal（§5 核心）

[patel-2024-acorn §5, Fig 3]

**理论 ideal**：oracle partition index = 每 predicate 一个 HNSW，search 复杂度 O_s(log(sn) + K)——sublinear in n。但 unbounded predicate 不可能 build 全。

**ACORN 实际**：把 HNSW 修改为 **denser graph**，让 search 时**沿 predicate-induced subgraph G(X_p) traverse** 能 approximate oracle partition behavior。

```
                    ┌─────┐
        ●●●         │ ●   │  predicate subgraph G(X_p)
       ●●●●         │  ● ● │  绿色节点 = 满足 predicate 的点
        ●●          │ ● ●  │  ACORN search 在这个子图上 traverse
                    └─────┘
                          ↑
                   ACORN 索引
                   构造时让每节点
                   有 M·γ 候选邻居
```

## ACORN-γ 算法（§5.1-5.2）

### Construction modifications（vs HNSW）

[patel-2024-acorn §5.2]

```
HNSW 构造:
  每 node v: 收集 M 个 candidate neighbors → RNG-based pruning → 留 M

ACORN-γ 构造:
  每 node v: 收集 **M·γ** candidate neighbors （γ 是 neighbor expansion factor）
            → predicate-agnostic pruning（保留 M_β + 部分 truncate）
            → 总边数 ~M·γ（vs HNSW 的 M）
  
  γ 选择基于预期 minimum selectivity s_min:
    γ = 1 / s_min  (论文经验值)
    s_min 用 30-80 selectivity 的预期 lower bound
```

### Predicate-Agnostic Pruning

[patel-2024-acorn §5.2 + Fig 5]

为何不用 RNG-based pruning（HNSW / Vamana / FilteredVamana 都用）？

```
HNSW pruning rule（metadata-blind）:
  对 v 的候选 a, b, c：如 dist(a, b) < dist(v, b) 则 prune b
  → 假设单一 predicate set；任意 predicate 子图上不一定 hold

[FilteredVamana](./filtered-vamana.md) pruning（metadata-aware）:
  prune p' iff F_p* ⊃ F_p'
  → 假设 predicate set 在 ≤1000 范围内已知

ACORN-γ pruning（predicate-agnostic + compression）:
  保留前 M_β candidates uncompressed
  剩余 M·γ - M_β 候选 truncate
  → 不假设 predicate；compression 节省内存
```

### Search modifications

[patel-2024-acorn §5.1, Algorithm 2]

```
ACORN-SEARCH-LAYER:
  while |C| > 0:
    c ← argmin in candidate set
    if dist(c, x_q) > dist(f, x_q) and |W| ≥ ef·c:
      break
    neighborhood ← GET-NEIGHBORS(c, l, p_q)
    
    # ACORN-γ: simply filter + truncate
    # ACORN-1: filter + 2-hop expansion if neighbor list 不够
    
    for v in neighborhood[1:M]:
      ...
```

### 适应 query correlation 与 selectivity 的能力

ACORN-γ 在所有三种 query correlation（positive/no/negative）+ 大范围 selectivity（1pc-100pc）下**性能稳定**——这是 post-filtering 不能做到的。

## ACORN-1 算法（§5.3）

[patel-2024-acorn §5.3]

**目标**：approximate ACORN-γ search 但 construction 极简。

```
Construction:
  与 HNSW 同（γ=1, M_β=M）——TTI ~HNSW

Search:
  每 visited node v:
    - 1-hop neighbors of v: filter by predicate
    - 不够 M 个 → 2-hop neighbor expansion
    - 把 v 的邻居的邻居加入候选 + 再 filter
    
  → 等同于 search 时**动态构造 ACORN-γ 的 dense graph**
```

**Trade-off**：
- ✓ Construction TTI 9-53× lower than ACORN-γ
- ✓ Index size = HNSW × ~1
- ✗ Search QPS ~5× lower than ACORN-γ at fixed recall
- 适用：resource-constrained / 频繁 rebuild / 实时构造

## 关键性质 / 复杂度

[patel-2024-acorn §6]

| 维度 | HNSW oracle partition | **ACORN-γ** | post-filtering |
|---|---|---|---|
| Search complexity（per query） | O_s(log(sn) + K) | **O((d+γ)·log(sn) + log(1/s))** | O(log n + K/s)（best） / O(n)（neg corr） |
| Construction complexity | O(n·log n) | **O(n·γ·log(n)·log(γ))** | n/a |
| 复杂度 sublinear in n? | ✓ | **✓** | ✗（neg corr） |
| Unbounded predicate? | ✗（must build per p） | **✓** | ✓ |
| Construction time vs HNSW | n/a | **~11×** | 1× |
| Index size vs HNSW | n/a | **~1.3×**（vs StitchedVamana 1.6×） | 1× |

## 与 [HNSW](./hnsw.md) 的关系

ACORN 是 HNSW 的 **predicate-aware 扩展**——保持 HNSW 的 multilayer hierarchy + level assignment，仅修改：
1. Construction 时每 node 收 M·γ candidates 而非 M
2. Pruning 用 predicate-agnostic compression 而非 RNG
3. Search 时 GET-NEIGHBORS 加 predicate filter（ACORN-γ 简单 filter + truncate；ACORN-1 + 2-hop expansion）

→ ACORN 是 HNSW 库的 minor extension（不需重写 core）。

## 与 [FilteredVamana / StitchedVamana](./filtered-vamana.md) 的对比

| 维度 | FilteredVamana / StitchedVamana | **ACORN-γ / ACORN-1** |
|---|---|---|
| Base index | Vamana (DiskANN graph) | **HNSW** |
| Filter cardinality 上限 | **≤1000 equality filters** | **unbounded + 任意 operator** |
| Predicate operator | equality only | equality, regex, contains, between, OR... |
| LCPS 性能 | baseline | **2-10× faster than FilteredVamana** |
| HCPS 性能（10^8+） | **fail（无法 build）** | **support（核心论点）** |
| Construction 时知 predicate set? | ✓（FilteredVamana incremental + per-filter medoid） | **✗ predicate-agnostic** |
| Streaming insert | FilteredVamana ✓ / StitchedVamana ✗ | ACORN-γ partial / ACORN-1 ✓ |
| Construction TTI | FilteredVamana ~3× HNSW；StitchedVamana ~10× | ACORN-γ ~11× HNSW；ACORN-1 ~HNSW |

**关键差异哲学**：
- FilteredVamana：filter set 已知 + label baked-in graph build
- ACORN：**filter set 未知 + graph 是 dense superset，search 时 induce subgraph**

→ 两个方向**互补**：FilteredVamana 适合"少量已知 filter + production deployment"（如 Microsoft 47 region filter）；ACORN 适合"大量 ad-hoc query + 复杂 operator + 真实 web search workload"。

## 与 NHQ 的对比（论文 §2 + §7.2）

[patel-2024-acorn §2 + §7.2 末段] **NHQ** [Wang 2022, wiki 未 ingest]：
- 把 attribute 编码入 vector 拼接，"fusion distance" search
- 限制：单 structured attribute + equality only

→ ACORN 在 SIFT1M / Paper LCPS 上比 NHQ-NPG / NHQ-NPG_KGraph 2-10× faster（同 [Filtered-DiskANN A.1] 实证）。HCPS 上 NHQ 完全不 support。

## Open Questions

- **Selectivity 极低（s_min < 10^-4）**：论文 footnote 1 承认"highly selective queries where ACORN's predicate subgraph would be disconnected"——需 fall back to pre-filtering；ACORN 配置 minimum selectivity 阈值
- **多 predicate conjunction (AND/OR/NOT)**：论文 §1 承认 supports OR；AND/NOT 复杂表达通过 inverted index 等结构辅助；论文未深入
- **Distributed ACORN**：单机算法；多机扩展未涉及
- **ACORN + SSD**：HNSW 系列 SSD 集成困难（neighbor random access）；ACORN 没探讨 SSD 路径
- ~~**ACORN 与 [FilteredVamana](./filtered-vamana.md) 在 production 实测对比**：[FilteredVamana](./filtered-vamana.md) 有 Microsoft 广告 A/B test +35-49% revenue；ACORN 仅学术 benchmark——production deployment 比较 wiki 未覆盖~~ **2026-05-11 双 ingest 完整关闭 frontier**：
  - **Qdrant v1.16.0** [qdrant-docs] 集成 ACORN 作为 Filterable HNSW fallback——多 strict filter combination 或 soft-deleted points 多时启用
  - **Weaviate** [weaviate-docs llms.txt] 集成 ACORN + "**positive & negative correlation optimization**"——query planner 用 payload 字段相关性估计 selectivity
  - **两个独立 OSS vector DBMS 都集成 ACORN**——证明 [patel-2024-acorn] Stanford 学术工作已是工业 OSS DBMS 的 default-or-fallback 之一
  - 但 Qdrant / Weaviate docs 都未公开 production workload vs ACORN 论文 25M LAION 实证的 gap。FilteredVamana Microsoft 广告 A/B test +35-49% revenue 仍是 unique production metric
- **NHQ-NPG 实际表现**：论文 §A.1 实测 KGraph 版本仍 1 order of magnitude slower than FilteredVamana at 100 recall；ACORN 论文 §7.2 用 KGraph 版本作 baseline——但 ACORN vs Filtered-DiskANN 直接对比是关键
- **更新（delete / mutation）**：ACORN 论文未深入；HNSW 自身 delete 困难继承

Cited by: 待 query 引用
