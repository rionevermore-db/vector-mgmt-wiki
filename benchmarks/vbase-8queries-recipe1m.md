---
title: VBASE 8-query Benchmark on Recipe1M（vs PostgreSQL / PASE / Milvus / Elasticsearch）
type: benchmark
sources: [zhang-2023-vbase]
related: [../systems/vbase.md, ../systems/milvus.md, ../systems/pase.md, ../systems/analyticdb-v.md, ../systems/spann.md, ../concepts/relaxed-monotonicity.md, ../topics/topk-vs-iterator-model.md, ../topics/attribute-filtering.md, ../topics/multi-vector-queries.md, ../topics/vector-range-query.md]
created: 2026-05-08
updated: 2026-05-08
---

# VBASE 8-query Benchmark on Recipe1M

**TL;DR**: VBASE OSDI 2023 论文 §5 提出的 **8-query benchmark on Recipe1M extension**——首次系统覆盖 vector + relational 混合查询完整空间（pure TopK / TopK+filter / multi-column TopK / multi-column+filter / range / Join 共 8 类），对比 PostgreSQL / PASE / Milvus / Elasticsearch / VBASE 五个系统。**核心发现**：所有 baseline 都用 TopK 接口，在 Q1 single-vector TopK 上算法等价（VBASE 4.9 ms vs PASE 4.8 ms），但**Q4-Q6 multi-column / Q7 range / Q8 Join 上 VBASE 比 baseline 快 100×-7900×**——这是 [Iterator Model + Relaxed Monotonicity](../concepts/relaxed-monotonicity.md) 范式相对 TopK 范式的根本胜利。

## 实验设置

[zhang-2023-vbase §5.1, §5.2]

### 数据集：Recipe1M extension

| Table | 行数 | 向量字段 | 标量字段 |
|---|---|---|---|
| **Recipe** | 330,922 recipes（去 1M 中缺图/instruction 的） | `images_embedding` 1024-d (cross-modal [Recipe1M+]), `description_embedding` 1024-d | `recipe_id`, `images` (URI), `description`, `popularity` (random int [0, 10000]) |
| **Tag** | 10,000 recipes 子集 | `tag_vector` 1024-d (image embedding，与 Recipe `images_embedding` 同 model) | `id`, `tag_name` (manual labels: dessert, salad, pizza, ...) |

### 评估平台

- **VBASE 与 baseline**：Azure VM Standard_F64s_v2，**64 vCPU + 128 GiB RAM**，Linux Ubuntu 20.04 LTS
- **VBASE+SPANN**：Azure VM Standard_L16s_v3 with **NVMe disks**，§5.4
- **每 query 单独运行**避免相互干扰

### 索引设置

[§5.1] 所有 baseline + VBASE 用 **HNSW (M=16, ef_construction=200, ef_search=64)**——HNSW 是 PASE / Milvus / Elasticsearch 共同支持的唯一索引。VBASE 还实验了 IVFFlat 与 SPANN 但 baseline 不支持。

所有 baseline + VBASE 都为 `popularity` 列建 **B-tree** 索引。

### 实现差异（解释 Q1 latency variation）

[§5.3]

| System | 实现语言 |
|---|---|
| Milvus / Elasticsearch | Go / Java |
| PASE / VBASE | C / C++ |

C 实现通常比 Go/Java 快 2-10×（因此 Milvus Q1 9.4 ms vs PASE 4.8 ms 主要是语言差，不是算法差）。

### Baseline 实现细节

[§5.1, §5.3]

| System | TopK K' 策略 |
|---|---|
| **Milvus** | **Iterative Merging algorithm**（[wang-2021-milvus §4.2]，开源未实现，论文复刻）+ doubling K' |
| **Elasticsearch** | OpenDistro 1.13 + HNSW，post-filter |
| **PASE** | static K' ∈ {100, 1000, 10000} 三档对比 |
| **PostgreSQL** | brute force（无 vector 索引）作 ground truth |

## 8 query 类型

[§5.2]

| # | SQL | 类型 |
|---|---|---|
| Q1 | `SELECT recipe_id FROM Recipe ORDER BY INNER_PRODUCT(images_embedding, $q) LIMIT 50` | Single-Vector TopK |
| Q2 | + `WHERE popularity <= $p_popularity` | TopK + Numeric Filter |
| Q3 | + `WHERE description NOT LIKE '%${p_ingredient}%'` | TopK + String Filter (regex) |
| Q4 | `ORDER BY INNER_PRODUCT(images_emb, $q1) + WEIGHT * INNER_PRODUCT(description_emb, $q2)` | Multi-Column TopK |
| Q5 | Q4 + `WHERE popularity <= $p` | Multi-Column TopK + Numeric Filter |
| Q6 | Q4 + `WHERE description NOT LIKE '%...%'` | Multi-Column TopK + String Filter |
| Q7 | `WHERE INNER_PRODUCT(images_embedding, $q) <= $D` | Vector Range Filter |
| Q8 | `Recipe JOIN Tag ON INNER_PRODUCT(images_emb, tag_vector) <= $D` | Vector Similarity Join |

每个 query 跑 **10000 substitution parameters**（不同 selectivity / filter conditions）。`p_popularity` 从 1 incrementing 到 10000；`p_ingredient` 从 Recipe1M 关键词采样。

## 主结果（Table 4 + Figure 6）

### Q1 - Single Vector TopK（无 filter）

| System | Recall | avg latency (ms) | 99p (ms) |
|---|---|---|---|
| PostgreSQL (brute) | 1.0000 | 2980.1 | 3133.6 |
| **PASE** | 0.9949 | **4.8** | 5.1 |
| Milvus | 0.9949 | 9.4 | 12.7 |
| Elasticsearch | 0.9949 | 43.1 | 121.4 |
| **VBASE** | 0.9949 | 4.9 | 5.3 |

→ **算法等价**：所有 ANN 系统 recall 都 0.9949（HNSW 同参数）；latency 差异主要是实现语言（C vs Go vs Java）。VBASE 与 PASE 都是 C 实现，latency 几乎相同。

### Q2/Q3 - TopK + Filter

[Table 4]

| System | Q2 99p | Q3 99p |
|---|---|---|
| PostgreSQL | 2286.2 | 9953.0 |
| PASE | 61.7 | (best K' 选好) |
| Milvus | 121.4 | NA (不支持 string filter) |
| Elasticsearch | 118.1 | 100.9 (recall 0.5010) |
| **VBASE** | **6.3** | **51.7** |

→ **VBASE 6.3 ms 99p**（**~10×** 优 PASE 61.7 ms）；Elasticsearch Q3 recall 仅 0.5010（K' 设错严重 undershoot）。

### Q4-Q6 - Multi-Column TopK

[Table 4 + §5.3 Q4-6]

| System | Q4 average | Q4 99p | Recall |
|---|---|---|---|
| PostgreSQL | 5610.0 | 5769.8 | 1.0 (brute) |
| PASE | **6696.4** | **9299.0** | 1.0 |
| Milvus | 19.8 | 46.4 | **0.9696** |
| Elasticsearch | NA (不支持 Q4) | - | - |
| **VBASE** | **5.3** | **5.3** | **0.9696** |

→ **Milvus Q4 比 VBASE 慢 200-300×**（论文 §5.3 明示 Iterative Merging "guesses different K' to produce a sufficiently large intersection"，**导致大量 random reads**）。

| System | Q5 99p | Q6 99p |
|---|---|---|
| Milvus | 36886.9 | 16734.6 |
| **VBASE** | **160.7** | **51.7** |

→ Q5/Q6 同样 200-300× 优势。

### Q7 - Vector Range Filter

[Table 4 + Table 6]

| System | recall | avg | 99p |
|---|---|---|---|
| PostgreSQL | 1 | 8244.9 | 8641.6 |
| PASE (K'=100) | 0.7103 | 7.3 | 8.8 |
| PASE (K'=10000) | 0.9991 | 392.1 | 484.9 |
| Milvus | NA (不支持) | - | - |
| Elasticsearch | NA (不支持) | - | - |
| **VBASE** | **0.9840** | **10.8** | **168.9** |

→ PASE 走 `ORDER BY + LIMIT K'` 拼凑；K'=10000 才接近 VBASE recall 但 latency 36× 慢。VBASE iterator 直接 distance > r 即停。

### Q8 - Vector Join

[Table 4]

| System | recall | avg latency |
|---|---|---|
| PostgreSQL nested-loop | 1 | **129,051,273 ms** (35.8 hours) |
| PASE | NA (不支持) | - |
| Milvus | NA (不支持) | - |
| Elasticsearch | NA (不支持) | - |
| **VBASE** | **0.9992** | **16,335.9 ms** (16.3 sec) |

→ **VBASE 比 PG nested-loop 快 7900×**——是 wiki 内迄今最大单 query 加速比。其他系统**结构性无法处理 Join**——无 unified query engine。

## 多 column 算法对比（Table 7）

[§5.3 Q4-6]

Multi-column TopK 50, **K'-equivalent metric: average NumOfScans**：

| Weight ratio | Round-Robin scans | Greedy scans | Greedy recall | VBASE scans | VBASE recall |
|---|---|---|---|---|---|
| 1:1 | 651.93 | 699.02 | 0.9313 (locally trapped) | **638.56** | **0.9705** |
| 1:2 | 617.22 | 612.51 | 0.9655 | **593.99** | **0.9818** |
| 1:5 | 463.39 | 372.96 | 0.9949 | 409.31 | **0.9961** |
| 1:10 | 363.47 | 274.86 | 0.9985 | 311.66 | **0.9987** |

→ **Greedy** 在权重悬殊时（1:5 / 1:10）显著快但 1:1 时 recall 0.9313（locally trapped）；**Round-Robin** 鲁棒但慢；**VBASE dynamic 切换**总在最优附近。

## Selectivity Estimation 精度（Figure 7）

[§5.5]

VBASE sampling rate **0.001**：

| Range filter selectivity | q-error |
|---|---|
| 0.05 | up to 1.27 |
| 0.15 - 0.95 | < 1.1 |

→ Sampling 估计精度足够 query planner 做 vector-index vs B-tree 选择。

**对比 PASE**：[§5.5] PASE 默认 selectivity = 0.5（不估计）→ scalar selectivity 0.05 时仍选 B-tree（错失最优）。

## Query Planning 实测（Figure 8）

[§5.5]

```sql
SELECT recipe_id FROM Recipe WHERE
  INNER_PRODUCT(q, images_embedding) < $r
  AND popularity < $p;
```

| 实验 | Default (PASE 0.5) | VBASE | Ground Truth |
|---|---|---|---|
| Fig 8a (range sel = 0.13, vary scalar sel 0.05-0.5) | always B-tree → wrong at low sel | matches GT | optimal |
| Fig 8b (scalar sel = 0.90, vary range sel 0.05-0.5) | always B-tree | matches GT | optimal |

→ VBASE plan 选择**接近 ground truth**；PASE default 选择系统性失误。

## VBASE+SPANN（Table 8）

[§5.4]

VBASE 集成 [SPANN](../systems/spann.md) on Azure Standard_L16s_v3 NVMe（NVMe disks）：

| Query | Recall | avg latency | 99p |
|---|---|---|---|
| Q1 | 0.9911 | 9.4 | 11.6 |
| Q2 | 0.9214 | 10.7 | 44.9 |
| Q3 | 0.9847 | 9.7 | 11.8 |
| Q4 | 0.9481 | 32.2 | 68.2 |
| Q5 | 0.9757 | 87.4 | 519.7 |
| Q6 | 0.9516 | 40.1 | 126.2 |
| Q7 | 0.9923 | 17.8 | 283.5 |
| Q8 | 0.9638 | 87,729.3 | (single param) |

→ **全部 8 query 可行**；Q5 99p 高（519.7 ms）因 SSD 随机 IO 放大。证明 RM iterator 抽象适用于 partition-based + SSD 索引。

## 可信度评估

### 实验设计

- ✓ 数据集 + 索引参数完整披露
- ✓ 多 selectivity 测试（10000 substitution per query）
- ✓ 同 hardware 对比
- ✓ Open source 代码 (`microsoft/MSVBASE`)
- ✓ Q1 的 baseline 跨语言（C/C++/Go/Java）解释清楚（§5.3）
- ✗ 仅 Recipe1M 单数据集（330K-10K，million-scale 单实例）——billion-scale 与 distributed 未实证

### 复现难度

- VBASE 开源 `microsoft/MSVBASE`，PG 13 + extension
- PASE 开源 `forrest-2007/PASE`，但 paper 中的 K' 三档需要参数化
- Milvus Iterative Merging 论文复刻而非用 open-source 实现（OSS 未实现）——可能与 production Milvus 行为有差异
- Elasticsearch HNSW（OpenDistro）开源
- 平台 Azure VM 公开

### 偏向

- Microsoft Research 论文，但 VBASE / SPANN / Milvus 都来自 Microsoft 系——cross-team competition 可能影响 baseline 选择动机
- 但**实测代码与配置披露完整**，可独立验证
- Recipe1M dataset 的"random integer popularity"是合成（不是真实数据分布）——影响 selectivity estimation 评估

## Open Questions

- **Billion-scale 实测**：论文止步于 Recipe 330K——10B / 100B vector 的 RM 行为？wiki 未覆盖
- **Distributed 集成**：单实例 PG；分布式 PG 时的 cross-shard RM 协调？
- **更多 baseline**：[AnalyticDB-V](../systems/analyticdb-v.md)（OLAP-extended）/ [Pinecone](../systems/pinecone.md)（SaaS）未对比——前者闭源 + 后者无 self-host
- **GPU 索引集成**：VBASE 未集成 [Faiss-GPU](../systems/faiss.md) / NVIDIA RAFT CAGRA——iterator 抽象在 GPU batch 优化下的可行性？
- **多 vector index 类型混用**：实测仅 HNSW（baseline 限制）；VBASE 同时 HNSW + IVFFlat + SPANN 时 query planner 选择行为？
- **Embedding model 升级**：与所有 wiki 已 ingest source 一致，未涉及 cross-model 升级
