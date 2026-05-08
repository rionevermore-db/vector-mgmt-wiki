---
title: FilteredVamana / StitchedVamana（Filter-aware Graph ANNS）
type: concept
sources: [gollapudi-2023-filtered-diskann]
related: [./vamana.md, ./hnsw.md, ./proximity-graph.md, ./acorn.md, ../systems/diskann.md, ../topics/attribute-filtering.md, ../benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md, ../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md]
created: 2026-05-08
updated: 2026-05-08
---

# FilteredVamana / StitchedVamana

**TL;DR**: Microsoft Research 2023 WWW 论文的两个 filter-aware graph ANNS 算法，**首次把 label 信息 baked-in 到 graph index 构造本身**——不只在 search 步骤过滤。建在 [Vamana](./vamana.md) 之上：FilteredVamana 是**incremental** 版本（streaming-friendly）；StitchedVamana 是**batch** 版本（per-label Vamana + 拼接）。在 Microsoft 真实数据 + 半合成数据上**比 Milvus / NHQ / Faiss IVF 高一个数量级 QPS@90%recall** across 1%-100% filter specificity。Microsoft 赞助广告搜索 A/B test +34.61% clicks / +48.95% revenue（小区域 +70%/+80%）。是 [DiskANN](../systems/diskann.md) 框架下的 Filtered-DiskANN 核心算法。[gollapudi-2023-filtered-diskann §3-5]

## 提出背景

[gollapudi-2023-filtered-diskann §1.2 "Drawback of Existing Methods"]

之前 filtered ANNS 方法**只改 search 步骤，不改 index build**。论文逐项 critique：

| 现有方法 | 问题 |
|---|---|
| **Post-processing**：标准 ANN → filter | 低 specificity (filter 严) 时退化——召回数大量候选才碰一个匹配 |
| **Pre-processing per filter**：每 filter 一个 index | 大量 filter 时不可行（论文 Turing 数据集 3070 unique filters） |
| **Inline-processing**（FAISS-IVF, Pinecone hybrid）| 在 IVF / LSH 上有效；在 graph 上仍是 search-time 改动，**不改 build** |
| **Weaviate / Milvus inverted index over labels** | 改 search 但 graph 本身仍 label-blind |
| **NHQ** [Wang 2022] | 把 labels 拼到 vector 末尾——单 label/point 假设；多 label 完全 disjoint label set 时退化 |
| **AnalyticDB-V** [wei-2020-analyticdb-v] | 4-plan CBO 但每 plan 仍是 search-time |

**Filtered-DiskANN 的根本差异**：**graph 边的连接同时基于 geometric proximity + label 共享**——build 时 baked-in。

## 5 个核心算法

[gollapudi-2023-filtered-diskann §3]

### Alg 1: FilteredGreedySearch（query 时）

```
Input: graph G, start node S, query x_q, list size L, filter set F_q
Output: top-L 候选 + visited set V

while L \ V ≠ ∅:
    p* = arg min over p ∈ L\V  ||x_p − x_q||
    V = V ∪ {p*}
    N'_out(p*) = {p' ∈ N_out(p*) : F_p' ∩ F_q ≠ ∅, p' ∉ V}  ← 仅 label 相交
    L = L ∪ N'_out(p*)
    if |L| > L_max:
        L = closest L_max nodes
return top k from L
```

**关键**：只有 `F_p' ∩ F_q ≠ ∅` 的 out-neighbor 才被加候选。**与 query filter 不相交的邻居直接跳过**——graph traversal 自动避开无关区域。

### Alg 2: FindMedoid（per-filter start nodes）

每 filter f 选独立 start node st(f)：
1. 对每 filter f 在 P_f（满足 f 的点集）中**随机采样 τ 个候选**
2. 每候选 p 维护 counter T[p]
3. 选 counter 最小的 p* 作 st(f) — load balance across filters

**为什么**：单一 medoid 作所有 filter start point → 起点附近邻居被多 query 路径反复经过 → 局部 hot spot。Per-filter start point + load-balanced 选择避免该问题。

### Alg 3: FilteredRobustPrune（build 时）

[Vamana RobustPrune α 系数](./vamana.md) 加 filter 检查：

```
N_out(p) = {}
while V ≠ ∅:
    p* = closest in V
    N_out(p) ∪= {p*}
    if |N_out(p)| = R: break
    for p' in V:
        if F_p' ∩ F_p* ⊄ F_p:    # filter intersection 不被覆盖 → keep
            continue
        if α · d(p*, p') ≤ d(p, p'):
            remove p' from V
```

**关键**：α 控制 prune 严格度（同 Vamana），但加一**filter 维度的 keep 规则**——`F_p* 不能完全覆盖 F_p'` 的 neighbor 即使 α-far 也保留。

### Alg 4: FilteredVamana Indexing（incremental，streaming-friendly）

```
G ← empty
s ← medoid of P
for each filter f: st(f) per Alg 2
σ = random permutation of [n]
for i in [n]:
    候选起点 = {st(f) : f ∈ F_{σ(i)}}
    [_; V_F] = FilteredGreedySearch(候选起点, x_{σ(i)}, 0, L, F_{σ(i)})
    FilteredRobustPrune(σ(i), V_F, α, R) ← 设新邻居
    backward edges: for each p' in N_out(σ(i)): add edge p' → σ(i)
    if N_out(p') 超 R: FilteredRobustPrune on p'
```

→ 与原 Vamana 同框架，仅把 search/prune 替换为 filter-aware 版本。**Incremental**：随 σ 顺序加点；可 streaming。

### Alg 5: StitchedVamana Indexing（batch）

```
对每 filter f：
    G_f = Vamana(P_f, α, R_small, L_small)  ← per-filter 独立 Vamana
对每 vertex v:
    FilteredRobustPrune(v, N_out(v), R_stitched)  ← 把所有 G_f 的边 union 后再 prune
```

→ **batch only**——必须 known label set 提前。Edge set = union(per-filter Vamana edges)。
**Trade-off**: build 慢（per-filter Vamana 总和）；查询通常更快（每 vertex 累积更多 useful 候选邻居）。

## 关键性质 / 复杂度

| 维度 | FilteredVamana | StitchedVamana |
|---|---|---|
| Build 模式 | **incremental** | **batch only** |
| Build 时间（Microsoft DANN 3.3M） | 159.8 s | 469.9 s（**~3× slower**） |
| Streaming-friendly | ✓ | ✗（needs known label set） |
| QPS @ 90% recall | baseline | 通常略高 2-7× |
| Filter set 数支持 | 千级（论文实测 3070 filters） | 同上（更少更佳） |
| 每 vertex 边集大小 | 较小 | union 后较大 |
| 增量加点 | ✓ direct | ✗（需 rebuild）|

→ **paper 推荐 FilteredVamana** as more practical for production，因 streaming 友好 + build 快。

## 与 wiki 已有 graph ANNS 的关系

[per concepts/proximity-graph.md, concepts/vamana.md]

| | [HNSW](./hnsw.md) | [NSG](./nsg.md) | [Vamana](./vamana.md) | **FilteredVamana** |
|---|---|---|---|---|
| 层数 | 多层 | 单层 | 单层 | 单层 |
| α 参数 | 隐式 α=1 | 隐式 α=1 | **可调（核心创新）** | 同 Vamana + filter-aware |
| 长程边 | 顶层 hierarchy | medoid + RNG | α>1 + 平面 | 同 Vamana |
| 边构造维度 | geometric only | geometric only | geometric only | **geometric + label** |
| 增量 add | ✓ | ✗ | ✗ | **✓**（Filtered**Vamana**） |
| Filtered ANNS native | ✗ | ✗ | ✗ | **✓** |
| SSD 友好 | ✗ | ✗ | ✓（small diameter） | ✓ |

→ FilteredVamana **首次把 label 加进图构造本身**——是 graph ANNS 演化新维度。

## 与其他 filtered ANNS 方法对比

[gollapudi-2023-filtered-diskann §1.2, §A]

| 方法 | 路径 | filter-aware build | wiki 内系统 |
|---|---|---|---|
| Post-processing（标准 ANN + filter）| Search-time | ✗ | 朴素 |
| Pre-processing per filter | Build-time per filter | partial | 不可扩展 |
| Inline-processing（FAISS-IVF）| Search-time | ✗ | Faiss IDSelector |
| Pinecone hybrid filter | Search-time | ✗ | [Pinecone](../systems/pinecone.md) |
| Milvus 5-strategy CBO | Search-time | ✗ | [Milvus](../systems/milvus.md) |
| ADBV 4-plan CBO | Search-time | ✗ | [AnalyticDB-V](../systems/analyticdb-v.md) |
| NHQ [Wang 2022] | Embed labels as vector dims | partial（仅 single label/point）| wiki 未 ingest |
| **FilteredVamana** | **Build-time** | **✓** | **本 page** |

→ FilteredVamana 是首个在 build 步骤就 label-aware 的实用 graph 方法。

## 在 [DiskANN](../systems/diskann.md) 框架下：Filtered-DiskANN

[gollapudi-2023-filtered-diskann §5.5]

把 FilteredVamana / StitchedVamana 直接放进 DiskANN 框架（DRAM PQ codes + SSD graph）：
- 同 Vamana on SSD 路径
- 28M DANN dataset 实测：**thousands QPS @ 90%+ recall on inexpensive SSDs**
- 24 threads, beam width 4, search list L 40-100

→ "Filtered-DiskANN" 名字来源。

## 与 [ACORN](./acorn.md) 的对比（NEW）

[patel-2024-acorn] 2024 SIGMOD 论文 attack FilteredVamana 的 **filter cardinality 限制**：
- FilteredVamana / StitchedVamana 限 ≤1000 equality filters + single equality operator
- ACORN 支持 **>10^11 predicates + 任意 operator (regex/contains/between/OR)**

| | FilteredVamana / StitchedVamana | ACORN-γ / ACORN-1 |
|---|---|---|
| Base index | Vamana | HNSW |
| Filter cardinality | **≤1000 equality** | **unbounded** |
| Predicate operator | equality only | equality, regex, contains, between, OR... |
| Filter set 已知? | construction 时已知 | **predicate-agnostic** |
| LCPS 性能 | baseline | **2-10× faster than FilteredVamana** |
| HCPS 性能（10^8+） | **fail** | **support**（30-1000× over baseline） |
| Production A/B 实证 | **Microsoft 广告 +35-49%** | 学术 only |

**互补关系**：
- FilteredVamana 适合 **少量已知 filter + production deployment**（如 Microsoft 47 region filter）
- ACORN 适合 **大量 ad-hoc query + 复杂 operator + 真实 web search**

→ 两者**不是替代**，而是**针对不同 workload pattern**。详见 [benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md](../benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md)。

## Open Questions

- **Filter conjunctions (AND of multiple filters)**：论文 §2 明确说"limit to simpler filters - exact match with one filter"。复杂 boolean filter（AND/OR/NOT）"challenging avenue for future work"
- **超过几千 filters**：论文实测 |F| ≈ 1000 quasi-OK；更大 |F| 性能未量化
- **完全 dynamic（删除）**：FilteredVamana incremental insert 友好；deletion 论文承认"future work"——与 [SPFresh LIRE](./lire.md) 对应物（in-place delete 在 graph 上）仍开放
- **多 label/point 的 graph 拓扑变化**：StitchedVamana edge union 在每 vertex 多 label 时膨胀；论文 §3.2 末尾承认"could be as large as building separate index"
- **Filter selectivity 极低（<1e-6）下行为**：Microsoft Turing 实测 1pc=10^-2；更低未实测
- **与 NHQ-NPG 的实测细节**：appendix §A.1 在 5 NHQ datasets 上 Filtered-DiskANN 一个数量级 better at 100 recall，但其余区间 NHQ 也 OK——比较窄场景

Cited by: 待 query 引用
