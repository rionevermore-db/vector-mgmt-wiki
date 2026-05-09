---
title: VBASE（PostgreSQL 上的统一 vector + relational 查询引擎）
type: system
sources: [zhang-2023-vbase]
related: [milvus.md, pase.md, analyticdb-v.md, spann.md, faiss.md, starling.md, ../concepts/relaxed-monotonicity.md, ../concepts/hnsw.md, ../concepts/product-quantization.md, ../concepts/vgpq.md, ../concepts/rabitq.md, ../concepts/block-shuffling.md, ../topics/topk-vs-iterator-model.md, ../topics/attribute-filtering.md, ../topics/multi-vector-queries.md, ../topics/vector-range-query.md, ../topics/index-selection.md, ../benchmarks/vbase-8queries-recipe1m.md, ../benchmarks/starling-vs-diskann-spann-on-segment.md]
created: 2026-05-08
updated: 2026-05-09 (Starling)
---

# VBASE

**TL;DR**: Microsoft Research Asia + ECNU + USTC + Renmin University 在 OSDI 2023 提出的研究系统：基于 [Relaxed Monotonicity](../concepts/relaxed-monotonicity.md) 性质，把 vector index 与 relational index 统一到 **PostgreSQL 的 Volcano / Iterator query engine** 之下。仅 **~2000 LOC** 加到 PostgreSQL，每个新集成的索引（HNSW / IVFFlat / SPANN）<200 LOC。8-query Recipe1M benchmark 上比 [Milvus](./milvus.md) Q4-Q6 multi-column TopK 快 **200-300×**，比 PostgreSQL Q8 vector Join 快 **7900×**——根本上是因为 VBASE **完全绕开 TopK speculation**（K' 预测问题），让 query engine 通过 RM 在线决定每个 vector iterator 的最优 K̃。GitHub: `microsoft/MSVBASE`。[zhang-2023-vbase §1, §6]

## 与 wiki 现有系统的定位差异

[per topics/topk-vs-iterator-model.md, zhang-2023-vbase §7]

| | [Milvus](./milvus.md) | [AnalyticDB-V](./analyticdb-v.md) | [PASE](./pase.md) | [Pinecone](./pinecone.md) | **VBASE** |
|---|---|---|---|---|---|
| 起点 | vector-first DBMS | OLAP RDBMS-extended | OLTP RDBMS-extended | vector-first SaaS | **关系 RDBMS-extended (PG)** |
| 核心 query interface | TopK + iterative merging | TopK + 4-plan CBO | TopK via amgettuple | TopK + filter | **Iterator (Open/Next/Close) + RM** |
| 处理动态 selectivity | iterative merging（多轮 TopK，慢） | static plan 选择 | iterative pop（per-query 不调） | metadata filter | **on-the-fly K̃，无需 K' 猜测** |
| Multi-column TopK | iterative merging 200-300× 慢 | 不支持 | 不支持 | 不支持 | **NRA-style merge + greedy/round-robin 自适应** |
| Vector range query | 不原生支持 | 不原生支持 | 需手动 LIMIT 拼凑 | 不原生支持 | **原生支持** [§5.2 Q7] |
| Vector Join | 不支持 | 不支持 | 不支持 | 不支持 | **原生支持，比 PG nested-loop 快 7900×** |
| 集成新 index 的工程代价 | 高（系统层修改） | 高（4-plan 适配） | 高（PG kernel 内核改） | n/a | **<200 LOC per index** |

**核心论点**：之前所有系统都把 vector index 当 **black-box TopK 提取器**——这导致 multi-column / range / Join 等复杂查询都需要外层"猜 K' 然后 merge"。VBASE 的不同之处是把 vector index **降级**成"和 B-tree 同级的 iterator"，用关系数据库 50 年来的 NRA / Fagin / Volcano 经典技术统一处理。

## 架构图

[zhang-2023-vbase Fig 4]

```
┌──────────────────────────────────────────────────┐
│  PostgreSQL SQL Parser / Planner                 │
│  + VBASE Selectivity Estimator (sampling-based)  │  ← §5
├──────────────────────────────────────────────────┤
│  Volcano Executor (Open / Next / Close)          │
│  ├─ Projection                                   │
│  ├─ Sort with Limit                              │
│  ├─ Filter                                       │
│  ├─ Index Scan                                   │
│  └─ Join                                         │
├──────────────────────────────────────────────────┤
│  Index Access Method Layer                       │
│  ┌──────────┬──────────┬──────────┬──────────┐  │
│  │ B-Tree   │ Hash     │ HNSW     │ IVFFlat  │  │
│  │ (PG)     │ (PG)     │ (VBASE)  │ (VBASE)  │  │
│  ├──────────┼──────────┼──────────┼──────────┤  │
│  │                                  │ SPANN  │  │
│  │   amopen / amgetnext /           │ (VBASE)│  │
│  │   amisrm  / amclose              │        │  │
│  └──────────────────────────────────┴────────┘  │
├──────────────────────────────────────────────────┤
│  PostgreSQL Storage (Heap / WAL / MVCC / ACID)   │
└──────────────────────────────────────────────────┘
```

## 数据流 / 控制流

### Iterator 接口（核心）

[zhang-2023-vbase §4 + §6]

每个 vector index 实现 4 个 method：

```c
amopen(scan, query_vec, ef);   // 初始化（HNSW: 设 entry node + ef）
Tuple* amgetnext(scan);         // 返回下一最近邻（按距离递增）
bool amisrm(scan);              // 是否进入 Phase 2（RM 判定）
amclose(scan);                  // 释放
```

**关键**：`amisrm` 由具体 index 实现，封装"我是否已离开 query"的判定。这是 RM 性质从"论文"到"工程接口"的桥梁。详见 [concepts/relaxed-monotonicity.md](../concepts/relaxed-monotonicity.md)。

### 单 vector + filter query 处理（Q2/Q3）

```sql
SELECT recipe_id FROM Recipe
WHERE popularity <= 100
ORDER BY INNER_PRODUCT(images_embedding, $q) LIMIT 50;
```

VBASE 执行流：

```
loop:
  vec_iter.Next()        ← 取下一最近邻 vector tuple
  fetch corresponding row from Recipe
  if popularity <= 100:
    add to result heap
  if heap.size >= 50 AND vec_iter.amisrm():
    break              ← RM 判定：再也不会更近了
return heap
```

→ **不需要 K' 预测**——RM 自动告诉 engine "可以停了"。

### Multi-column TopK 处理（Q4/Q5/Q6）

[zhang-2023-vbase §4.5]

```sql
SELECT recipe_id FROM Recipe
ORDER BY INNER_PRODUCT(images_embedding, $q1)
       + WEIGHT * INNER_PRODUCT(description_embedding, $q2)
LIMIT 50;
```

VBASE 同时打开 2 个 vector iterator，**用 NRA-style 合并**：

```
threshold-based termination：
  对每个 iterator 维护当前最近距离 d_i
  当所有 iterator 都在 Phase 2 且 sum(d_i × weight_i) > top-50 score 时停
```

VBASE **自动选择 strategy**：
- **Round-Robin**：所有 iterator 轮流推进；权重均匀（1:1）时鲁棒
- **Greedy**（NRA）：优先推进当前距离最小的 iterator；权重悬殊时（1:5/1:10）显著快但 1:1/1:2 易陷局部最优
- **VBASE 默认 dynamic 切换**——比 round-robin 在权重悬殊时快、在均匀时不退化

[Table 7] 实测 1:5 weight，Greedy 372 NumOfScans / 14.9 ms / 0.9949 recall vs Round-Robin 463 / 16.93 / 0.9946——Greedy 略胜；1:1 weight Greedy 0.9313 recall 显著低于 VBASE 0.9705。

### Range query（Q7）

[zhang-2023-vbase §4.4 + §5.2]

```sql
SELECT recipe_id FROM Recipe
WHERE INNER_PRODUCT(images_embedding, $q) <= $D;
```

VBASE 实现：

```
loop:
  v = vec_iter.Next()
  d = distance(v, q)
  if d > D:
    break              ← 超出 range
  add v to result
```

**首次原生支持 vector range filter**——其他系统（PASE/Milvus/Elasticsearch）需要构造大 K' 然后过滤，**且无法 stop 早**（必须扫完所有 K' 候选才能保证 recall）。详见 [topics/vector-range-query.md](../topics/vector-range-query.md)。

### Vector Join（Q8）

[zhang-2023-vbase §4.4 + §5.2]

```sql
SELECT Recipe.recipe_id, Tag.tag_name FROM Recipe JOIN Tag
ON INNER_PRODUCT(Recipe.images_embedding, Tag.tag_vector) <= $D;
```

VBASE 用 nested-loop join + outer Tag.tag_vector 作 query → inner vec_iter range search。仅扫到 distance > D 即停。**比 PostgreSQL nested-loop full scan 快 7900×**（Recipe 330K + Tag 10K = 3.3B 距离计算 vs VBASE 仅 ~10^7 范围查询）。

## 关键设计决策

### 1. 不重写 PostgreSQL（§6.1）

- 仅 **~2000 LOC** 修改
- vector indices 通过标准 IndexAmRoutine 接口加（与 [PASE](./pase.md) 同 mechanism）
- **不动** parser / executor / storage / WAL / MVCC——这意味着 VBASE 自动继承 PG 的 ACID / replication / backup / RBAC

**Trade-off**：受 PG 单机能力限制（与 PASE 同）vs 工程代价极低 + 与 PG 生态完全兼容。

### 2. 每个 vector index 集成 <200 LOC（§6.2）

- HNSW 集成：基于 hnswlib，加 amgetnext 单步 traversal + Phase 2 detection
- IVFFlat：维护 cluster heap，单步 pop 最近 cluster 内最近 vector
- SPANN：partition + posting list 顺序，单步 pop posting list 内 vector

**关键**：因为 RM 是 vector index 的内在性质（不是 VBASE 强加），实现上只是把 vector index 内部已有的 traversal logic expose 为 single-step iterator。

### 3. Selectivity estimation by sampling（§5）

[zhang-2023-vbase §5 + Fig 7]

VBASE 用 **0.001 sample rate** 估计 filter selectivity：

| Selectivity | sample 数 | q-error |
|---|---|---|
| 0.05 | 较少 | up to 1.27 |
| 0.15 - 0.95 | 充足 | < 1.1 |

→ Sampling 估计精度足够 query planner 做 vector-index vs B-tree 选择。**对比 [PASE](./pase.md) 默认 selectivity = 0.5**（不估计），导致 plan 选择系统性失误。

### 4. Vector index 与 B-tree 平等竞争（§5.1）

VBASE planner 把 vector index 与 B-tree 看作**两种 equally-rated 选项**：

- 估计 scalar filter selectivity α
- 如果 α 低（如 < 0.18 for 论文 setup）→ B-tree 扫小集合 + vector brute force 验证更优
- 如果 α 高（> 0.18）→ vector index 扫 + scalar filter 更优

实测 [Fig 8a, 8b] VBASE 与 ground truth 选择基本一致；PASE 因 default 0.5 总是选 B-tree（即使 α=0.05 也用 B-tree）。

### 5. Result equivalence 保证（§4.4）

VBASE 数学证明：

> 在 RM iterator 上做 filter / range / TopK 的结果，**严格等价于** TopK with K' = ∞ 的 ground truth 结果。

→ VBASE 路径不损失任何 recall——只是用 dynamic K̃ 替换 static K' 猜测。详见 [concepts/relaxed-monotonicity.md "Result equivalence 证明"](../concepts/relaxed-monotonicity.md)。

### 6. VBASE+SPANN：unified memory + SSD（§5.4）

[zhang-2023-vbase Table 8]

VBASE 在 Azure Standard_L16s_v3 NVMe 上集成 [SPANN](./spann.md)：

- 全部 8 query 类型可行
- Q1 latency: 9.4 ms / 11.6 ms 99p, recall 0.9911（vs HNSW in-memory 4.9 ms 但同 recall）
- Q5 latency: 87.4 ms / 519.7 ms 99p（多 column + filter 上 SSD 随机 IO 放大）

→ VBASE 证明 **partition-based + graph-based 索引可以共用 RM iterator 抽象**——这是 unified query engine 的真实可行性证据。

## Scale 边界

[zhang-2023-vbase §5.1, §5.4]

| 配置 | 数据 | 实测 |
|---|---|---|
| Azure Standard_F64s_v2 (64 vCPU, 128 GiB RAM) | Recipe 330K × 1024-d | 全 8 query, recall 0.99+ |
| Azure Standard_L16s_v3 NVMe | 同上 + SPANN partition | 全 8 query, recall 0.92-0.99 |
| 论文未实测 | billion-scale | wiki 未覆盖 |
| 论文未实测 | distributed PG | wiki 未覆盖 |

> **wiki 解读**：VBASE 是研究原型——million-scale Recipe1M 实测，没有 billion / distributed 实证。这是与工业系统（[Milvus](./milvus.md) billion / [ADBV](./analyticdb-v.md) 13B / [Pinecone](./pinecone.md) SaaS）的根本差异。VBASE 的价值在于**架构 insight**（RM 统一），不是 production scale。

## 与 wiki 已有系统的对比

### 与 [PASE](./pase.md)（同 PG 内核扩展路径）

[zhang-2023-vbase §5.3, Table 4]

| | PASE | **VBASE** |
|---|---|---|
| Vector index 集成方式 | PG IndexAmRoutine + 自管 page | PG IndexAmRoutine + 复用现有 vector lib (hnswlib / SPANN) |
| Compound query 处理 | iterative via amgettuple（早期 RM 思路雏形）| **正式形式化 RM + 跨多 index 协调** |
| K' 预测 | static guess (100 / 1000 / 10000) | **完全无需** |
| Selectivity estimation | default 0.5 | **0.001 sampling, q-error <1.1** |
| Multi-column TopK | 不支持 | **支持** |
| Range filter | 需 LIMIT 拼凑 | **原生** |
| Vector Join | 不支持 | **原生** |
| 实测 SIFT/GIST 1M latency Q1 | 4.8 ms / 5.1 ms 99p | 4.9 ms / 5.3 ms 99p（同等算法等价） |
| Q2 latency 99p | 28.7 ms (K'=1000), 117.4 ms (K'=10000) | **6.3 ms** (RM 自动停) |

→ **VBASE 是 PASE 的"正确版本"**——同 PG 扩展路径，但 PASE 的 amgettuple 思路被 VBASE 形式化为 RM iterator + 跨索引协调。论文 §5.3 明示 "PASE's amgettuple has the spirit but lacks the formalization"。

### 与 [Milvus](./milvus.md)（vector-first DBMS）

[zhang-2023-vbase §5.3, Table 4]

| | Milvus | **VBASE** |
|---|---|---|
| Multi-column TopK Q4 | 9300 ms 99p (Iterative Merging 失败) | 46.4 ms 99p (200× 快) |
| Q5 multi-col + numeric filter | NA（不支持 string filter for Q6） | 160.7 ms |
| 哲学 | vector-first，filter 是次要 column | **vector 与 scalar 平等并重** |
| 集成新 index 工程代价 | 系统级修改 | **<200 LOC** |
| Iterative Merging 算法 | 论文 [wang-2021-milvus §4.2] 声称用 Fagin NRA + doubling K | 实测在 Q4-Q6 失败：**accumulate 大量 random reads** |

→ Milvus 的 Iterative Merging 是 TopK 框架内的 best effort；VBASE 的 NRA 是 iterator 框架内的自然 fit——根本性的实现差异。

### 与 [AnalyticDB-V](./analyticdb-v.md)（OLAP RDBMS-extended）

| | ADBV | **VBASE** |
|---|---|---|
| Compound query 哲学 | 4-plan CBO + accuracy-aware 超参 grid search | **RM iterator + dynamic K̃** |
| Selectivity estimation | online α' 估计 + 离线 grid search per α-bin | **online sampling 0.001 rate** |
| TopK speculation | 仍需要（Plan B/C/D 都基于 PQ Knn Bitmap Scan） | **完全无需** |

→ ADBV 与 VBASE 是 compound query 的两条不同路径——前者把"猜 K'"工程化（4 plan + 离线超参），后者绕开"猜 K'"（RM 替代）。

## 生产案例

VBASE 是 **Microsoft Research 学术原型**——GitHub `microsoft/MSVBASE`，Apache 2.0。论文未声明 production deployment。

> **wiki 解读**：VBASE 的价值在 **architecture insight + open-source reference implementation**——RM 性质是普世的，理论上可移植到 [PASE](./pase.md) / [Milvus](./milvus.md) / [Pinecone](./pinecone.md) 等任何系统。但目前 wiki 已 ingest 系统中**只有 VBASE 显式实现 RM iterator**——这是 talk 时的关键 frontier。

## Open Questions

- **Billion-scale 实测**：论文止步于 Recipe1M（330K-1M）；RM 在更大数据集（10B+）上 Phase 边界检测的稳定性？
- **Distributed PG (Citus / Greenplum / PolarDB) 集成**：VBASE 是单实例 PG；分布式扩展时 RM iterator 跨 shard 的 progress aggregation 如何形式化？论文未涉及
- **PQ-based 索引（IVFPQ / [VGPQ](../concepts/vgpq.md)）的 RM 集成**：lossy distance 下 RM Phase 边界的精度损失？论文 §4 未深入
- **VBASE iterator + [RaBitQ](../concepts/rabitq.md) quantizer 叠加**：RaBitQ 在 quantizer 层用 error-bound 攻击 K' 预测问题；VBASE 在 query engine 层用 RM iterator 攻击同一问题——**正交可叠加**。理论上 VBASE engine + IVF + RaBitQ rerank 应保持 RM Phase 检测正确性（RaBitQ unbiased estimator + sharp bound 满足 RM 假设）；但 VBASE 论文 [zhang-2023] 早于 RaBitQ [gao-2024]，未实证。详见 [topics/topk-vs-iterator-model.md "K' 消除：双层路径"](../topics/topk-vs-iterator-model.md)
- **VBASE engine + [Starling](./starling.md) disk layer 叠加**：[per wang-2024-starling §7] Starling 把 PASE / VBASE / Milvus 列为 "support both ANNS and RS on the same dataset" 同代系统。两者攻击 disk-resident graph 的不同层——VBASE 在 query engine layer（RM iterator）、Starling 在 disk index layer（block shuffling + nav graph）。理论上可叠加（VBASE engine 跨多 segment + 每 segment 内用 Starling disk layer），但论文相互不知（VBASE OSDI 2023 与 Starling SIGMOD 2024 几乎同期）；wiki 内 zero coverage 实证
- **GPU 索引（Faiss-GPU / CAGRA）的 iterator 化**：[WarpSelect](../concepts/warpselect.md) 内 batch 优化与 single-step iterator 的接口冲突——论文未触及
- **Filter-aware 索引（[FilteredVamana](../concepts/filtered-vamana.md) / [ACORN](../concepts/acorn.md)）的 RM 适用性**：filter-aware build 后 RM 仍 hold？理论上 yes，未实测
- **Streaming insertion 下 RM 的稳定性**：[ADBV](./analyticdb-v.md) lambda / [Milvus](./milvus.md) growing segment / [SPFresh](./spfresh.md) LIRE 中 cluster 边界变化时 RM Phase 漂移？
- **VBASE 的 PG 版本兼容**：论文 PG 13 时代；后续 PG 14/15/16 的 executor 变化影响？wiki 未跟踪
- **Multi-column TopK 在权重 1:1 时 Greedy 失败**：[Table 7] 显示 1:1 时 Greedy recall 0.9313——VBASE 用 dynamic 切换补救，但理论上"权重比"的最优 strategy 边界（1:1 vs 1:2 vs 1:5）未给闭式
- **W=10 的 window size 选择**：[per concepts/relaxed-monotonicity.md] Phase 检测窗口；W 与 recall / latency trade-off 未给 closed-form
- **跨 model embedding 升级**：VBASE 论文不涉及；与其他 7 个已 ingest source 一致——这是 wiki 的全 frontier 盲区
- **生产 production 部署**：未见 Microsoft 内部产品采用 VBASE 报告

Cited by: 待 query 引用
