---
title: Vector Range Query（按距离阈值的全量返回）
type: topic
sources: [zhang-2023-vbase, yang-2020-pase]
related: [../systems/vbase.md, ../systems/pase.md, ../systems/milvus.md, ../systems/analyticdb-v.md, ../concepts/relaxed-monotonicity.md, ../concepts/hnsw.md, ../concepts/product-quantization.md, ./topk-vs-iterator-model.md, ./attribute-filtering.md, ../concepts/range-filter-ann-2024.md, ../benchmarks/vbase-8queries-recipe1m.md]
created: 2026-05-08
updated: 2026-05-21 (lint: link numeric range-filter ANN concept)
---

# Vector Range Query

**TL;DR**: 与 TopK 平行的查询模式——**返回所有 distance ≤ r 的向量**而不是固定 K 个。语义上对应 SQL `WHERE distance(v, q) <= r`，结果数量在 query 时**未知**（可能 1 个也可能 10000 个）。在 TopK 接口下表达困难（需要构造大 K' 然后过滤，且无法 stop 早）；在 [Iterator Model](./topk-vs-iterator-model.md) 下天然 fit（迭代到 distance > r 即停）。**[VBASE](../systems/vbase.md) 是 wiki 内首个原生支持** vector range query 的系统（OSDI 2023 Q7）；[PASE](../systems/pase.md) 仅能通过手动构造 LIMIT 拼凑，[Milvus](../systems/milvus.md) / [AnalyticDB-V](../systems/analyticdb-v.md) / Elasticsearch 都不原生支持。

## 问题陈述

### 与 TopK 的语义差异

| | TopK | Range Query |
|---|---|---|
| 返回数量 | K（固定，调用方指定） | **未知**（取决于数据分布与 r） |
| 语义 | "K 个最近的" | "所有 距离 ≤ r 的" |
| SQL | `ORDER BY ... LIMIT K` | `WHERE distance ≤ r` |
| 工业场景 | 推荐、相似搜索 | **outlier 检测、similarity Join、阈值聚类** |
| Recall 含义 | top-K 中找到了多少 ground truth | distance ≤ r 中找到了多少（**precision = 1 if 系统正确**） |

[zhang-2023-vbase §5.2 + §6.1] 强调：range query 的 **precision 必然 = 1**——系统返回的所有结果**必须**满足约束（不像 TopK 可能 missing）。因此 range query 评估仅 recall。

### 工业 use case

[zhang-2023-vbase §2.1 + §5.2 Q7]

| 场景 | range query 用法 |
|---|---|
| **Outlier detection** | "找所有 距离 query > threshold 的 → 异常点" |
| **Similarity Join** | "Recipe ⋈ Tag where image_emb · tag_vec ≤ 0.1" |
| **聚类阈值** | "找所有 距离 cluster centroid ≤ r 的 → cluster member" |
| **去重** | "找所有 距离已知 vector ≤ ε 的 → 重复" |
| **Web search 相似度过滤** | "返回所有 query similarity > 0.7 的页面" |

→ range query 在工业上**与 TopK 同等重要**，但工业 vector DBMS 普遍未原生支持——这是 wiki 内一个长期被忽视的 gap。

## 在 TopK 接口下的挑战

[zhang-2023-vbase §1, §5.2]

```sql
-- VBASE 形式（自然）：
SELECT recipe_id FROM Recipe
WHERE INNER_PRODUCT(images_embedding, $q) <= $D;

-- TopK 系统的 workaround：
SELECT recipe_id FROM Recipe
ORDER BY INNER_PRODUCT(images_embedding, $q)
LIMIT $K_LARGE;   -- 然后应用层过滤 distance <= D
```

**核心问题**：
1. **K_LARGE 不知设多大**——可能 1（少数符合）也可能 10000（多数符合）
2. **K_LARGE 设小** → recall < 1（precision 仍 = 1，因为系统至少返回了一些满足的）
3. **K_LARGE 设大** → 浪费扫描所有 K_LARGE 个 vector
4. **per-query selectivity 不同** → 无单一 K_LARGE 适配所有 query

[zhang-2023-vbase Table 5 + Table 6] 实测 PASE 用 `Order By + LIMIT K'` workaround：

| K' | recall | average latency | 99p latency |
|---|---|---|---|
| **100** | **0.7103** | 7.3 ms | 8.8 ms |
| 1000 | 0.9387 | 44.3 ms | 54.7 ms |
| 10000 | 0.9991 | 392.1 ms | 484.9 ms |
| **VBASE (range native)** | **0.9840** | **10.8 ms** | **168.9 ms** |

→ PASE K'=10000 才接近 VBASE recall，但 latency 36× 慢；K'=1000 仍 fail recall。VBASE iterator 直接走 distance > r 即停，没有 K' 选择问题。

## 在 Iterator Model 下的自然实现

[zhang-2023-vbase §4.4 + §6.1]

```c
// VBASE 实现（HNSW / IVFFlat / SPANN 通用）
amopen(vec_iter, query, ef);
loop:
  v = vec_iter.amgetnext();
  d = distance(v, query);
  if d > r:
    break;          // 超出 range
  add v to result;
amclose(vec_iter);
```

**关键**：依赖 [Relaxed Monotonicity](../concepts/relaxed-monotonicity.md) 性质——iterator 进入 Phase 2 后**不会再返回更近的向量**，所以遇到 d > r 后**所有后续都 d > r**，可以安全 break。

VBASE 论文 §4.4 给出 result equivalence 证明：
- 这种迭代方式返回的结果集**严格等于** {v ∈ X : distance(v, q) ≤ r} 的全集
- 没有 missing，没有 false positive
- recall = precision = 1（理论上；实测 VBASE recall 0.9840 因为 ANN 索引本身的近似性）

## Vector Join：range query 的 hardest case

[zhang-2023-vbase §5.2 Q8 + Table 4]

```sql
SELECT Recipe.recipe_id, Tag.tag_name FROM Recipe JOIN Tag
ON INNER_PRODUCT(Recipe.images_embedding, Tag.tag_vector) <= $D;
```

每条 outer (Tag) 行需要对 inner (Recipe) 做一次 range query。**每条**都面临 K' 决策——如果用 TopK 范式，K' 错误的代价乘以 |Tag| = 10000。

[Table 4] 实测：

| System | Q8 average | recall |
|---|---|---|
| PostgreSQL nested-loop full scan | 129,051,273 ms (35.8 hours) | 1 (ground truth) |
| Milvus | **NA**（不支持 Join） | - |
| Elasticsearch | **NA**（不支持 Join） | - |
| PASE | **NA**（不支持 Join） | - |
| **VBASE** | **16,335.9 ms** (16.3 sec) | **0.9992** |

→ VBASE **比 PG 快 7900×**，且是 wiki 内**唯一原生支持 vector Join 的系统**。这是 range query + iterator 范式的最大胜利。

## 现有 wiki 系统的 range query 现状

| 系统 | 原生 range query 支持 | workaround |
|---|---|---|
| **[VBASE](../systems/vbase.md)** | **✓ 原生**（Q7） | n/a |
| **[Milvus](../systems/milvus.md)** | ✗ | 用大 K LIMIT 然后 filter |
| **[AnalyticDB-V](../systems/analyticdb-v.md)** | ✗ | SQL `WHERE distance ≤ r` 但底层走 4-plan TopK |
| **[PASE](../systems/pase.md)** | ✗ | `ORDER BY + LIMIT K'`，K' 静态调（[Table 6]） |
| **[Pinecone](../systems/pinecone.md)** | ✗（API 文档无明确支持） | TopK + score threshold post-filter |
| **Elasticsearch** | ✗ | TopK + min_score post-filter |

→ VBASE 是 **唯一原生** vector range query——这是 RM iterator 范式的"独占特性"。其他系统理论上可以加 RM 接口，但目前未见公开实现。

## 与 [topics/topk-vs-iterator-model.md] 的关系

range query 是 TopK 范式**最暴露其缺陷**的查询类型——比 multi-column TopK 更甚。multi-column TopK 至少可以用 iterative merging 强行 fit；range query 的**结果数量未知**特性让 K' 猜测从根本上不可行。

## Open Questions

- **范围 r 的选择**：用户如何选 r？query selectivity 严重依赖 r——VBASE 论文未给 r 的工业实践指南
- **Range query 的 selectivity estimation**：VBASE [§5 Fig 7] 给出 q-error vs range r 的关系（r ≈ 0.5 时 q-error 最低；极端 r 估计差），实测 default 0.5 是合理近似但**自适应估计**未给闭式
- **Range query + filter 联合**：filter 与 range 都活跃时 cost model 如何选 plan？VBASE [Fig 8b] 实测但未给出理论公式
- **Distributed range query**：跨 shard / partition 的 range query progress aggregation？VBASE 单实例
- **Range query 在 PQ-based 索引下**：lossy distance 是否影响 range 边界判定？理论上 false positive/negative 都可能；VBASE 未实测 IVFPQ
- **Range query 在 [FilteredVamana](../concepts/filtered-vamana.md) / [ACORN](../concepts/acorn.md)**：filter-aware build 后 range query 的 RM 适用性？
- **Vector Join 算法的进一步优化**：VBASE 用 nested-loop（每条 outer 一次 range search）；理论上可以 hash join / sort-merge join——但 vector 没有总序——这是开放问题

Cited by: 待 query 引用
