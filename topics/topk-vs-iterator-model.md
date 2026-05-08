---
title: TopK 接口 vs Iterator Model（向量索引集成范式之争）
type: topic
sources: [zhang-2023-vbase, wang-2021-milvus, yang-2020-pase, wei-2020-analyticdb-v]
related: [../systems/vbase.md, ../systems/milvus.md, ../systems/pase.md, ../systems/analyticdb-v.md, ../systems/pinecone.md, ../concepts/relaxed-monotonicity.md, ../concepts/hnsw.md, ../concepts/product-quantization.md, ./attribute-filtering.md, ./multi-vector-queries.md, ./vector-range-query.md, ./index-selection.md, ../benchmarks/vbase-8queries-recipe1m.md]
created: 2026-05-08
updated: 2026-05-08
---

# TopK 接口 vs Iterator Model

**TL;DR**: 工业 vector DBMS 集成 vector index 时面对的 **架构二选一**：(A) 把 vector index 当 black-box **TopK 提取器**（[Milvus](../systems/milvus.md) / [AnalyticDB-V](../systems/analyticdb-v.md) / [PASE](../systems/pase.md) / Elasticsearch / [Pinecone](../systems/pinecone.md) 全选这条），或 (B) 把 vector index 当 **iterator**（VBASE 路径，OSDI 2023 提出）。前者是过去 5 年的事实标准但有 **K' 预测问题**——filter / multi-column / range / Join 都因为 K' 静态猜测而系统性失败；后者基于 [Relaxed Monotonicity](../concepts/relaxed-monotonicity.md) 完全绕开 K' 猜测。VBASE 实测在 Q4-Q6 multi-column TopK 上 **比 Milvus 快 200-300×**，Q8 vector Join **比 PostgreSQL nested-loop 快 7900×**——两个范式的工程差距是数量级。

## 问题陈述

向量+关系混合查询在工业上至少包含 5 类：

| Query 类型 | 例子 | 评估难度 |
|---|---|---|
| **Q1**: Pure TopK | "找 50 个最像 query 的 recipe" | 简单——任何 ANN 算法直接答 |
| **Q2/3**: TopK + scalar filter | "找 50 个最像 query 且 popularity ≤ 100 的 recipe" | **K' 问题**——选 50 个不够，多数被过滤 |
| **Q4-6**: Multi-column TopK | "按 image 与 description 综合最相似排序" | **多 K' 协调问题** |
| **Q7**: Vector range filter | "找 distance ≤ 0.1 的所有 recipe" | **K 未知**——可能 1 个也可能 10000 个 |
| **Q8**: Vector Join | "Join Recipe 与 Tag 当 image 与 tag_vector 距离 ≤ 0.1" | **每行 outer 都需 K' 决策** |

[zhang-2023-vbase §2 + Table 4] 的 8 query benchmark 把这 5 类全部覆盖——这是 wiki 内**首个系统覆盖 vector + relational 混合查询完整空间**的基准。

## 范式 A：TopK 接口（事实标准，2017-2023）

### 接口签名

```python
results = vector_index.topk(query_vec, K)
# returns K nearest neighbors as (vector, distance, id)
```

### 处理 filter / range / multi-column / Join 的方式

[wang-2021-milvus §4 + yang-2020-pase §2.6 + wei-2020-analyticdb-v §5]

**Filter (Q2/3)**：

```python
K' = K / estimated_filter_selectivity   # 静态猜测
candidates = vector_index.topk(query_vec, K')
filtered = [c for c in candidates if c.attr_passes_filter()]
return filtered[:K]
```

**问题**：
- selectivity 估计不准 → K' 不准
- K' 过小 → recall 不足（filtered 不足 K 个）
- K' 过大 → 浪费（扫不必要的 vector）
- selectivity **per-query 不同** → 无法用一个固定 K'

[zhang-2023-vbase Table 5] 实测 PASE 三档 K' 全部 fail：
- K'=100，selectivity=0.03 → recall **0.0567**（极低）
- K'=10000，selectivity=0.9 → latency **41.8 ms** 99p（浪费）
- K'=1000，selectivity 中等 → 单档不能 cover 全 selectivity 范围

**Multi-column TopK (Q4-6)**：

[wang-2021-milvus §4.2 "Iterative Merging"]：

```python
K' = K
loop:
  candidates_per_index = []
  for col in vector_columns:
    candidates_per_index.append(col.topk(query_vec, K'))
  merged = NRA_aggregate(candidates_per_index)
  if len(merged) >= K and confident:
    return merged[:K]
  K' = K' * 2   # 翻倍重试
```

**问题**：
- 每轮"翻倍 K'"是**几何级 random reads** → I/O 爆炸
- Merge 多次都不收敛 → tail latency 极高
- [zhang-2023-vbase Table 4] Milvus Q4 average 6696 ms / 99p 9300 ms vs VBASE 5.3 ms / 5.3 ms 99p

**Range query (Q7)**：

```python
K' = ???   # 不知道有多少 vector 距离 ≤ r
candidates = vector_index.topk(query_vec, K_LARGE)
return [c for c in candidates if c.distance <= r]
```

**问题**：
- K 不知道 → K_LARGE 必须很大 → 浪费
- 即使 K_LARGE 够大，仍然必须扫完全部 K_LARGE 候选才能保证 recall

**Vector Join (Q8)**：

```python
for outer in outer_relation:
  inner_results = inner_vector_index.topk(outer.vec, K')
  filter and join
```

**问题**：每条 outer 重复 K' 决策——K' 错误的 multiplier 是 |outer relation| 倍。

### 哪些系统选 TopK 范式

[per wang-2021-milvus Table 1 + yang-2020-pase + wei-2020-analyticdb-v + pinecone-docs]

| 系统 | TopK 实现 | 处理 filter 的细节 |
|---|---|---|
| **[Milvus](../systems/milvus.md)** | TopK + 5 filter 策略 + iterative merging | Strategy E partition-based 缓解 K' 但仍是 TopK |
| **[AnalyticDB-V](../systems/analyticdb-v.md)** | TopK + 4-plan CBO | 离线超参 grid search per α-bin（一种 K' 工程化） |
| **[PASE](../systems/pase.md)** | iterative pop via amgettuple | 早期 RM 思路雏形但未形式化；single-vector only |
| **[Pinecone](../systems/pinecone.md)** | TopK + metadata filter | 黑盒；具体算法不公开 |
| **Elasticsearch** | TopK + post-filter | open-source HNSW + bool query |
| **OpenSearch / Open Distro** | TopK + post-filter | 同 Elasticsearch |

→ **wiki 内 6/7 已 ingest 的 vector DBMS 都走 TopK**——VBASE 是唯一例外。

## 范式 B：Iterator Model（VBASE 路径，2023-）

### 接口签名

[zhang-2023-vbase §4 + §6]

```c
amopen(scan, query_vec, ef);
Tuple* amgetnext(scan);    // 返回下一最近邻（按距离递增）
bool amisrm(scan);          // 是否进入 RM Phase 2（"再也不会更近了"）
amclose(scan);
```

每次 `amgetnext()` 返回 **一个**最近邻；调用方决定继续还是停止。

### 处理 filter / range / multi-column / Join 的方式

[zhang-2023-vbase §4.2-4.5]

**Filter (Q2/3)**：

```c
loop:
  v = vec_iter.Next();
  if v.passes_filter():
    add to result
  if result.size >= K AND vec_iter.amisrm():
    break;       // RM Phase 2 → 自动停
```

→ **不需要 K' 预测**——RM 自动检测"再也不会更近"。

**Multi-column TopK (Q4-6)**：

```c
opened iterators: it1, it2, ...
threshold-based termination：
  while not all in Phase 2:
    pick next iterator (greedy / round-robin) and Next()
    update threshold
  return top-K from accumulated buffer
```

→ 经典 NRA / Fagin top-K aggregation——50 年前的 DB 技术，自然 fit iterator interface。

**Range query (Q7)**：

```c
loop:
  v = vec_iter.Next();
  if v.distance > r:
    break;   // 超出 range，自然停
  add to result
```

→ **K 不需要事先知道**——iterator 走到边界就停。

**Vector Join (Q8)**：

```c
for outer in outer_relation:
  open inner_iter for outer.vec
  range-search inner_iter until distance > r
  close inner_iter
```

→ 每条 outer 仅扫到 distance 超出，**没有 K' 翻倍重试**。VBASE 实测 7900× 快于 PG nested-loop。

### Iterator 范式的理论基础

[concepts/relaxed-monotonicity.md](../concepts/relaxed-monotonicity.md) 形式化的 RM 性质：HNSW / IVFFlat / SPANN 都满足"过了某一刻只会离开 query"的两阶段遍历——这让 iterator 接口的 `amisrm()` 可以被 vector index 内部状态准确判定。

**Result equivalence 证明**：[zhang-2023-vbase §4.4] 证明 iterator 路径的结果 ≡ TopK 路径在 K' = ∞ 时的 ground truth。换言之，iterator **不损失正确性**，只把 K' 从"静态猜"变成"动态决"。

## 范式对比

| 维度 | TopK 接口 | Iterator Model |
|---|---|---|
| **K 是否需要事先指定** | ✓ 必须 | ✗ 不需要 |
| **filter selectivity 处理** | 静态猜 K' = K/sel | RM 自动停 |
| **multi-column 处理** | iterative merging（O(K' × log K') 翻倍） | NRA threshold-based 一次性收敛 |
| **range query 处理** | 不自然（需 K_LARGE） | 自然（distance > r 就停） |
| **vector Join 处理** | 不自然（per outer K'） | 自然（per outer 范围搜索） |
| **Recall 保证** | 受 K' 选择影响 | 严格等价于 K' = ∞ |
| **集成新 index 的工程代价** | 高（系统级 K' 调度） | 低（<200 LOC per index） |
| **selectivity estimation** | 必需（cost-based plan） | 可选（VBASE 0.001 sampling） |

## Q4-6 实测：TopK 范式的崩溃

[zhang-2023-vbase Table 4]

Multi-column TopK 50 on Recipe 330K 1024-d：

| System | Q4 99p latency (ms) | recall |
|---|---|---|
| Milvus (TopK + iterative merging) | **9300** | 0.9041 |
| PASE | **5769** | 1.0 (但 brute-force) |
| **VBASE (iterator)** | **46.4** | **0.9696** |

→ Milvus 200× 慢；这是 [topics/multi-vector-queries.md](./multi-vector-queries.md) 的关键反例。

## Q8 实测：vector Join 在 TopK 范式下不可行

[zhang-2023-vbase Table 4]

Vector Join Recipe 330K × Tag 10K = 3.3B 距离对：

| System | Q8 average (ms) |
|---|---|
| PostgreSQL nested-loop | 129,051,273 ms (35.8 hours) |
| Milvus | NA — **不支持 Join** |
| Elasticsearch | NA — **不支持 Join** |
| PASE | NA — **不支持 Join** |
| **VBASE** | **16,335.9 ms** (16.3 sec) |

→ TopK-based 系统**结构性无法处理 vector Join**（除非应用层重写）；VBASE 用 iterator + range search 自然支持。

## 历史脉络

| 年份 | 事件 | 范式 |
|---|---|---|
| 2017-2020 | [Faiss](../systems/faiss.md) 主导，library-level TopK 接口 | TopK 是事实标准 |
| 2020 | [AnalyticDB-V](../systems/analyticdb-v.md) [wei-2020] 提出 4-plan CBO 缓解 K' 选择 | TopK + plan CBO |
| 2020 | [PASE](../systems/pase.md) [yang-2020] amgettuple 走 iterative pop——RM 雏形 | TopK 单 column iterative |
| 2021 | [Milvus](../systems/milvus.md) 1.x [wang-2021] 提出 partition-based filter 缓解 K' selectivity | TopK + 数据布局优化 |
| 2022 | [Milvus](../systems/milvus.md) 2.x (Manu) [guo-2022] iterative merging for multi-vector | TopK + Fagin doubling K' |
| **2023** | **VBASE [zhang-2023]** 形式化 RM + iterator model | **范式转换** |

→ TopK 范式经过 5 年的工程优化（CBO / partition / iterative merging）仍无法根本解决 K' 问题；VBASE 直接换范式。

## 工业方案对比

| 方案 | 选择者 | 优势 | 劣势 |
|---|---|---|---|
| **TopK + post-filter** | 早期所有系统 | 简单、易实现 | K' 预测失败时 recall 崩 |
| **TopK + partition-based** | [Milvus](../systems/milvus.md) E | 高频固定 filter 时 13.7× 快于 cost-based | 新加 filter 维度需重组 |
| **TopK + 4-plan CBO** | [AnalyticDB-V](../systems/analyticdb-v.md) | 全 selectivity 范围有 plan | 离线超参成本高 |
| **TopK + iterative merging** | [Milvus](../systems/milvus.md) multi-vec | 通用算法 | 多 column TopK 200-300× 慢于 iterator |
| **Iterative pop via amgettuple** | [PASE](../systems/pase.md) | RM 思路雏形 | single-vector only，未形式化 |
| **Iterator + RM** | [VBASE](../systems/vbase.md) | **绕开 K' 问题，自然支持 Q1-8** | 学术原型，未 production |

## Open Questions

- **TopK + iterator 混合系统**：能否在已有 vector DBMS（[Milvus](../systems/milvus.md) / [Pinecone](../systems/pinecone.md)）上**增量**集成 RM iterator？理论可行（RM 是 vector index 内在性质）但工程代价未量化
- **Iterator 范式在 distributed 场景**：跨 shard 的 RM Phase 协调？多 reader 的 progress aggregation？VBASE 论文未涉及
- **Iterator 范式在 streaming insertion**：[ADBV](../systems/analyticdb-v.md) lambda / [Milvus](../systems/milvus.md) growing segment 中 RM Phase 漂移？
- **Iterator 范式在 GPU 索引**：[WarpSelect](../concepts/warpselect.md) 的 batch 优化与 single-step iterator 接口冲突——VBASE 未集成 GPU
- **Iterator 范式在 lossy quantization**：[VGPQ](../concepts/vgpq.md) / IVFPQ 的距离误差对 RM Phase 检测的影响？
- **Filter-aware index ([FilteredVamana](../concepts/filtered-vamana.md) / [ACORN](../concepts/acorn.md)) 的 iterator 化**：build-time filter 后 RM 是否仍 hold？
- **TopK 范式的合理空间**：哪些 query 形态下 TopK 仍优于 Iterator？例如**纯 Q1 single TopK 无 filter** 时两者算法等价（VBASE Table 4 Q1: VBASE 4.9 ms vs PASE 4.8 ms）——RM 的 overhead 仅在 multi-column / range / Join 时才有 dramatic gain

Cited by: 待 query 引用
