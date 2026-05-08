---
title: Relaxed Monotonicity（向量索引与关系索引的统一遍历性质）
type: concept
sources: [zhang-2023-vbase]
related: [hnsw.md, nsg.md, product-quantization.md, vamana.md, ../systems/vbase.md, ../systems/milvus.md, ../systems/pase.md, ../systems/analyticdb-v.md, ../systems/spann.md, ../topics/topk-vs-iterator-model.md, ../topics/attribute-filtering.md, ../topics/multi-vector-queries.md, ../topics/vector-range-query.md, ../benchmarks/vbase-8queries-recipe1m.md]
created: 2026-05-08
updated: 2026-05-08
---

# Relaxed Monotonicity

**TL;DR**: VBASE [zhang-2023-vbase §4] 形式化的统一性质——**主流向量索引（HNSW / IVFFlat / SPANN）与传统关系索引（B-tree / hash）共享**的索引遍历性质：存在一个时刻 s，从 s 之后所遍历向量到 query 的距离的**移动中位数**只会越来越大（不再向 query 接近）。这让 vector index 与 relational index 可以**用同一套 Volcano / Iterator query engine** 处理——这是 VBASE 不再依赖 TopK speculation 的理论基础。论文称 "the **first** to identify a common foundation between vector & relational indices"。

## 提出背景

VBASE 论文 [zhang-2023-vbase §1, §3] 的核心动机：

之前的 vector DBMS（[Milvus](../systems/milvus.md) / [AnalyticDB-V](../systems/analyticdb-v.md) / [PASE](../systems/pase.md) / Elasticsearch）都基于 **TopK 接口**集成 vector index——TopK 把 vector 当 black-box 取 K 个最近邻 → 关系 query engine 在 K 个候选上做 filter / join / aggregation。这条路径必然要求 **K' = K / filter_selectivity** 的预测，而 selectivity 在 query 时未知 → 静态猜值在动态 selectivity 下系统性失败。详见 [topics/topk-vs-iterator-model.md](../topics/topk-vs-iterator-model.md)。

**VBASE 的根本观察**：vector index 的"接近 query 然后离开"的遍历模式，与关系索引（B-tree / hash）的 ordered scan 模式**结构相似**——都满足"过了某一刻就单调离开 target"。如果可以形式化这个共同性质，就可以用统一的 iterator interface 处理两类索引，**完全绕开 TopK speculation**。

## 形式化定义

[zhang-2023-vbase §4 Eq 4-6]

设 query 是 q，distance metric 是 D。索引按内部顺序遍历向量集合 X={x_1, x_2, ...}。定义在第 t 步的"窗口距离" M^t_q 为最近 W 步遍历向量到 q 的距离的中位数（W = window size，VBASE 默认 W = 10）：

```
M^t_q = median{ D(q, x_{t-W+1}), ..., D(q, x_t) }
```

定义查询的"参考距离" R_q 为：当遍历过的向量足够多时（至少 W 步），R_q = max(D(q, x_i)) for i ∈ candidates so far that satisfy filter。

**Relaxed Monotonicity (RM)** 性质：

```
∃ s ∈ ℕ such that ∀ t ≥ s:  M^t_q ≥ R_q
```

直观解释：**两阶段遍历**——
- **Phase 1（接近阶段）**：t < s，索引正在向 q 趋近，M^t_q < R_q（看到的向量越来越近）
- **Phase 2（离开阶段）**：t ≥ s 后，索引开始系统离开 q（窗口距离不会再小于已找到的 top-K 距离 → 可以安全停止）

## 三大类索引都满足 RM

[zhang-2023-vbase §4.2-4.3]

| 索引 | Phase 1 行为 | Phase 2 行为 | 自然的 stop 时机 |
|---|---|---|---|
| **[HNSW](./hnsw.md)** | 从 entry node 沿 graph greedy 下降到接近 q 的 cluster | 在该 cluster 邻域里反复回探，距离逐步抬升 | 到达"再回探都不更近"时停 |
| **[IVFFlat / IVF](../concepts/product-quantization.md)** | 排序 nprobe 个 cluster 后**首先**扫最近 cluster | 后扫的 cluster 距离 centroid 更远，平均距离抬升 | 跨过当前 top-K 时停 |
| **[SPANN](../systems/spann.md)** | 排序 K 个 posting list 后扫最近的 list | 后扫的 posting list 距离更远 | 同上 |
| **B-tree (numeric `popularity`)** | 从 root 找到 query 区间起点 | 沿叶子顺序扫，单向移动远离起点 | 区间扫完 |
| **B-tree + relaxed `LIKE`** | 同上 | 同上 | 同上 |

**关键洞察**：所有这些索引都**自带"我已经离 query 越来越远了"的信号**——只是之前没有 query engine 把这个信号统一表达出来。VBASE 把它形式化为 RM，并 expose 为 iterator 的 `Next()` 单步语义。

## 与 TopK 接口的根本差异

| | TopK 接口 | RM-based Iterator 接口 |
|---|---|---|
| 单次返回 | K 个最近邻（K 提前指定） | 当前最近的 1 个（增量） |
| 控制权 | 调用方提前给 K | 调用方决定何时停止 `Next()` |
| 处理 filter 的方式 | 预测 K' = K / selectivity（不准） | 不断 `Next()` 直到累积 K 个满足 filter 的 |
| 处理 range query | 不自然（要 ∞ 大 K） | 自然——`Next()` 直到距离超过 r |
| 处理 multi-column / Join | 不自然（外层不知道何时停） | 自然——RM 让外层知道每条 source 的 progress |
| 复杂度（动态 selectivity） | K' 设小 → recall 损失；K' 设大 → 浪费 | 自适应到真正最优 K̃ |

[zhang-2023-vbase §1 + Table 4]：实测 PASE/Milvus/Elasticsearch（all TopK-based）在 Q4-Q6 multi-column TopK 上比 VBASE **慢 200-1000×**。

## 与已 ingest concept 的关系

### 与 [HNSW](./hnsw.md) 的两阶段搜索

HNSW 论文 [malkov-2016-hnsw §3-4] 描述的"zoom-in / zoom-out"实际上就是 RM 的特例——zoom-in 是 Phase 1，zoom-out 是 Phase 2。但 HNSW 论文未把这个性质 expose 为可被 query engine 利用的 iterator 接口；VBASE 第一次把它"提取"成可用语义。

### 与 [Vamana / DiskANN](./vamana.md) 的关系

[Vamana](./vamana.md) 的 GreedySearch 也满足 RM——α-controlled RobustPrune 保证 graph 的 navigability。VBASE §6.7 提到"未集成 DiskANN 但理论上同样适用"。

### 与 [IVF / Product Quantization](./product-quantization.md) 的关系

IVFFlat 满足 RM（按 cluster 距离排序遍历）；IVFPQ 也满足——但 PQ 距离是 lower bound（量化误差），所以是 **lossy RM**。VBASE 论文未深入 IVFPQ；只对 IVFFlat 实测。

### 与 [SPANN](../systems/spann.md) 的关系

SPANN 的 query-aware dynamic pruning [chen-2021-spann §3.2.3 Eq 3] 实质上是 RM 的近邻：根据 D(q, c_i1) 决定哪些 posting list 跳过——这与 RM 的"离 q 越来越远就停"语义同源。VBASE §5.4 实测把 SPANN 集成进 RM 框架仅需 <200 LOC，证明 RM 的 abstraction 适用于 partition-based + SSD 索引。

## 关键性质

### 1. 遍历无需提前知道 K（论文 §4.1）

不像 TopK 必须 specify K，RM-based iterator 由调用方动态决定停止时机。这是 [topics/topk-vs-iterator-model.md] 整章节的核心。

### 2. Result equivalence 证明（§4.4）

VBASE 论文证明：用 RM iterator + filter 的结果**等价于** TopK 的结果，**当 K' 足够大时**——即 RM iterator 的"自然 K̃"是 TopK 在最优 K' 下的 ground truth。所以 RM 路径不会损失正确性，只会"自动找到正确的 K̃"。

### 3. 启用 multi-column 自然合并（§4.5）

多个 vector index（不同 column）+ 多个 scalar index（B-tree）共同走 iterator interface → 用经典 NRA 算法（Fagin top-K aggregation）就可以 merge——不需要 vector-specific 的 fusion 算法。详见 [multi-vector-queries.md](../topics/multi-vector-queries.md)。

### 4. 启用 query planner cost estimation（§5）

每个 RM iterator 可以估计 progress（已扫多少 / Phase 1 还是 Phase 2）→ 让 cost-based planner 可以**实时**调整 plan（而不是 query 开始前 freeze）。

## 工程实现要点

[zhang-2023-vbase §4.2 + §6]

VBASE 在 PostgreSQL Volcano executor 中给 vector index 添加 4 个接口：

```c
// VBASE iterator API（每个 vector index 实现）
void   amopen(IndexScanDesc scan, query_vec, ef);  // open
Tuple* amgetnext(IndexScanDesc scan);              // next ANN candidate
bool   amisrm(IndexScanDesc scan);                 // RM phase 检测
void   amclose(IndexScanDesc scan);                // close
```

**关键**：`amisrm` 接口暴露当前 iterator 是 Phase 1 还是 Phase 2——query engine 据此决定继续扫还是停止。这是 RM 性质从"理论"→"工程接口"的桥梁。

VBASE 实测：每个新索引集成（HNSW / IVFFlat / SPANN）只需 **<200 LOC**——足见 RM 抽象的简洁。

## 与 [topics/topk-vs-iterator-model.md] 的关系

RM 是"为什么 iterator 接口可以替代 TopK"的**理论基础**；topics/topk-vs-iterator-model.md 是这个理论应用到工程哲学层面的展开。两者必读。

## Open Questions

- **PQ-based 索引（IVFPQ / [VGPQ](./vgpq.md)）的 RM 行为**：lossy distance 在 RM 下的精度损失？论文未深入
- **Filter-aware 索引（[FilteredVamana](./filtered-vamana.md) / [ACORN](./acorn.md)）的 RM 适用性**：filter-aware build 后 RM 是否仍 hold？理论上 yes，但 ACORN 的 2-hop expansion 与 RM 窗口语义可能复杂化
- **GPU 索引（CAGRA）的 RM**：GPU greedy traversal 的 phase 分界是否可识别？wiki 未覆盖
- **Window size W 的最优值**：VBASE 默认 W=10 [§4.2]，理论上 W → ∞ 严格但实测 W=10 已足够准确——但论文未给出 W 的 closed-form 选择
- **RM 在 streaming insertion 下的稳定性**：[ADBV](../systems/analyticdb-v.md) 的 lambda 流式索引 / Milvus 的 growing segment 中，RM 的 Phase 边界如何漂移？
- **结合 [LIRE](./lire.md) 的 in-place update**：SPFresh 在 update 时 cluster 边界变化，RM 假设是否被破坏？
- **跨 model embedding 升级时 RM 的 transferability**：是否需要重 evaluate Phase 边界？wiki 未覆盖

Cited by: 待 query 引用
