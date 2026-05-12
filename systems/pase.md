---
title: PASE（PostgreSQL ANN Search Extension）
type: system
sources: [yang-2020-pase, wang-2021-milvus, pgvector-docs]
related: [analyticdb-v.md, milvus.md, faiss.md, vbase.md, pgvector.md, ../concepts/hnsw.md, ../concepts/product-quantization.md, ../concepts/filtered-vamana.md, ../concepts/relaxed-monotonicity.md, ../topics/attribute-filtering.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/index-selection.md, ../topics/topk-vs-iterator-model.md, ../topics/vector-range-query.md, ../benchmarks/pase-vs-cube-freddy.md, ../benchmarks/vbase-8queries-recipe1m.md]
created: 2026-05-08
updated: 2026-05-12 (pgvector triangulates Postgres-extension path)
---

# PASE

**TL;DR**: Ant Financial 的 PostgreSQL 扩展——**第一个直接在 PG kernel 注册 ANN index type 的方案**。不是 GiST plugin（ImgSmlr/Cube），不是 SQL 扩展（Freddy），而是通过 PG 的 IndexAmRoutine 接口添加 IVFFlat 与 HNSW 两种 index type，**复用 PG 自身的 transaction / WAL / replication / access control 全套 OLTP 能力**。是 [AnalyticDB-V](./analyticdb-v.md) 的 sister-system——同 Alibaba ecosystem，同"DB-extended-vector"路径，但 host DB 不同：PASE 是 **OLTP RDBMS (PostgreSQL)**，ADBV 是 **OLAP**。已在 Ant Financial / Alipay 多个生产场景部署（地铁人脸支付、图像版权检测、Alipay 推荐、PolarDB 人脸识别）。**不支持 billion-scale + distributed**（[wang-2021-milvus Table 1] 标 ✗）。[yang-2020-pase §1, §2]

## 与 wiki 现有系统的定位差异

[per topics/attribute-filtering.md, wang-2021-milvus Table 1, guo-2022-manu §7]

| | [Faiss](./faiss.md) | [Milvus](./milvus.md) | [Pinecone](./pinecone.md) | [AnalyticDB-V](./analyticdb-v.md) | **PASE** |
|---|---|---|---|---|---|
| 起点 | ANN library | Vector-first DBMS | Vector-first SaaS | **关系 OLAP** | **OLTP RDBMS (PG)** |
| 路径 | 算法工具箱 | vector-first DBMS | vector-first SaaS | OLAP 加 vector | **PG 内核加 vector index** |
| 部署模式 | embed | self-host / Zilliz Cloud | SaaS only | Alibaba Cloud | **PG plugin（或 PolarDB integration）** |
| Distributed | ✗ | ✓ | ✓ | ✓ | **✗（受 PG 单机限制）** |
| Billion-scale | ✓ | ✓ | ✓ | ✓ | **✗** |
| 复用宿主 DB 能力 | n/a | n/a | n/a | AnalyticDB SQL/storage | **✓ PG transaction / WAL / replication / access control 全套** |

**核心差异**：PASE 的设计哲学是"vector index 是 PG 的 first-class index type，与 B-Tree / GiST / GIN / BRIN 平级"——所有 PG 自身的运维 / 备份 / 复制 / 监控工具直接复用。代价是受 PG 单机能力限制（million-scale 是当前 sweet spot；billion-scale 需多个 PG 实例 + 应用层分片）。

## 架构图

[yang-2020-pase Fig 3]

```
┌─────────────────────────────────────────────────────┐
│  PG IndexAmRoutine kernel                           │
├─────────────────────────────────────────────────────┤
│  PG index interface                                 │
│  ┌─────────────┬─────────────┬───────────────────┐  │
│  │ BTree mod.  │ GiST mod.   │ PASE Index module │  │
│  │ build/scan  │ build/scan  │ build/scan        │  │
│  │             │             │ + common utils    │  │
│  └─────────────┴─────────────┴───────────────────┘  │
├─────────────────────────────────────────────────────┤
│  PG index storage                                   │
│  ┌─────────┬─────────┬─────────┬───────────────┐    │
│  │ BTree   │ GiST    │ GIN     │ PASE:         │    │
│  │ Index   │ Index   │ Index   │ ├─ IVFFlat   │    │
│  │ pageN   │ pageN   │ pageN   │ └─ HNSW      │    │
│  └─────────┴─────────┴─────────┴───────────────┘    │
├─────────────────────────────────────────────────────┤
│  PG base table（raw data）                          │
└─────────────────────────────────────────────────────┘
```

## 数据模型

### 自定义数据类型 `pase`

[yang-2020-pase §2.5]

```sql
-- 创建 pase 数据类型的列
CREATE TABLE vector_table (
    id BIGINT PRIMARY KEY,
    city VARCHAR,
    features pase  -- 支持 plaintext 或 base64 加密
);

-- 创建 PASE 索引（IVFFlat 或 HNSW）
CREATE INDEX hnsw_idx ON vector_table
    USING pase_hnsw (features) WITH (bnn=16, efb=200);

-- 查询：vector similarity ORDER BY
SELECT id FROM vector_table
ORDER BY features <op> '[0.0117,0.0115,0.0087,0.01]'::pase ASC LIMIT 10;

-- Compound query: vector + WHERE
SELECT id FROM vector_table
WHERE city = 'Hong Kong'
ORDER BY features <op> '[...]'::pase ASC LIMIT 10;
```

`<op>` 是自定义 distance operator（用户在 ext 注册时指定 distance function）。支持 Euclidean / cosine / inner product。

## 关键设计决策

### 1. 复用 PG IndexAmRoutine 而非加 plugin（§1.1, §2）

PG 现有 vector 方案的对比：

| 方案 | 类型 | 维度上限 | 性能 | 状态 |
|---|---|---|---|---|
| **ImgSmlr** [yang-2020-pase ref 6] | GiST plugin | **16** | brute-force 类 | "design ideas, not large-scale" |
| **Cube** [yang-2020-pase ref 8] | GiST plugin | **<100** | dim>100 退化 | PG built-in 但实测不可用 |
| **Freddy** [yang-2020-pase ref 9] | SQL function extension | 任意 | 慢（layer over index）| word embedding 主用 |
| **PASE** | **kernel index type** | **2000+ via cross-page** | 快 | 论文方案 |

**Trade-off**：编辑 PG kernel 复杂度高（需懂 IndexAmRoutine + page layout）vs 性能 + 与 PG 内核深度集成。

### 2. 8 KB page-aligned storage（§2.2-2.4）

PG 默认 page 大小 8 KB。PASE 为 IVFFlat 与 HNSW 各设计三类 page：

**IVFFlat 三类 page** [Fig 5]：
- Meta-page：存索引元信息（cluster 数、维度等）
- Centroid-page chain：存 cluster centroids（一个 cluster 可跨多 centroid-tuple）
- Data-page chain：每 cluster 一条 data-page chain，存该 cluster 的全精度 vector

**HNSW 三类 page** [Fig 8]：
- Meta-page：存索引元信息（entry node、初始化参数等）
- Data-page chain：存 vector raw data
- Neighbor-page chain：存图邻居关系（NeighborTuple）

> **wiki 解读**：page 8 KB 上限驱使 PASE 把 vector 存全精度而非压缩——8 KB 装不下大量 vector + PQ codes 同时；用全精度 page 化更直接。这与 [SPANN](./spann.md) 的 12-48 KB posting / [DiskANN](./diskann.md) 4 KB block 都不同——**8 KB 是 PG 历史决策强加的**。

### 3. Cross-page storage for high-dim vectors（§2.4）

8 KB page payload ~7 KB；维度 > 2000（float32 = 8 KB）单 page 装不下单向量。PASE 用 cross-page DataTuple：
- 向量切多 segment，每 page 存一段
- DataTuple header 记 segment offset + segment length

**Trade-off**：支持任意维度 vs cross-page read-time multi-IO 开销。

### 4. Contiguous page storage（§2.3）

PG 默认 page 分配是 sparse（VACUUM 等导致 fragmentation）。PASE 主动通过 N-page block 分配确保 IVFFlat data-page chain 连续：
- Build / Insert / Update：分配 N-page block 而非单 page
- Vacuum：rebuild data chain 到 contiguous space

**Trade-off**：write path 更复杂 vs read 时 sequential scan 速度（IVFFlat scan 多页时显著）。

> **wiki 解读**：HNSW 不需 contiguous storage——HNSW neighbor 本来就 random access；IVFFlat 才需要因为 cluster 内 sequential scan。这是 PASE 针对 IVFFlat 与 HNSW 不同访问模式的工程优化。

### 5. Iterative search via `amgettuple`（§2.6）

[yang-2020-pase Fig 13]

Compound query 处理流程：

```
prepare global priority queue
seek next top-K via index scan
push into queue
loop:
  pop top + check other conditions（WHERE 子句）
  if queue empty:
    seek next top-K
  if results enough:
    end
```

**关键**：用 PG 自身的 iterative search interface（`amgettuple` 返回单 tuple）——避免预估"amplification factor σ"（[ADBV](./analyticdb-v.md) 路线）。优点：不需要 cost model 预估；缺点：单线性 iterative，不能批量优化。

> **VBASE [zhang-2023-vbase §5.3] 评价**："PASE's amgettuple has the **spirit** but lacks the **formalization**"——PASE 是 [Relaxed Monotonicity](../concepts/relaxed-monotonicity.md) 的早期 workmark：iterative 思路对了，但缺少 (a) RM 性质的严格定义，(b) 跨多 vector index 的 iterator 协调，(c) selectivity sampling estimation。**PASE 仅支持单 vector column TopK + filter**（Q1-Q3）；multi-column TopK / range filter / vector Join（Q4-Q8）都不支持或性能崩溃。详见 [topics/topk-vs-iterator-model.md](../topics/topk-vs-iterator-model.md) 与 [VBASE benchmark](../benchmarks/vbase-8queries-recipe1m.md)。

> **wiki 解读**：[ADBV 4-plan CBO](./analyticdb-v.md) 与 PASE iterative search 是 compound query 的**两种工程哲学**：
> - ADBV：前置 cost model + α 估计 + plan 选择
> - PASE：iterative + 增量 fetch + 自然 short-circuit
>
> ADBV 处理大 selectivity（高过滤）需 plan 切换；PASE 增量 fetch 自然 fallback。论文未直接对比；理论上 ADBV 在 known α 下更优，PASE 在 unknown α 下更稳。

### 6. PG 全套 OLTP 能力 free 复用

PASE 创建 vector index 即享受 PG 全套：
- ACID transaction（INSERT/UPDATE/DELETE 在 vector index 上原子）
- WAL replication / point-in-time recovery
- Multi-version concurrency control
- Row-level security / RBAC
- Backup / pg_dump
- Streaming replication

**Trade-off**：复用 PG 内核 vs 受限于 PG 单机能力（无 cluster mode；distributed 需应用层分片或 PolarDB-style 分布式 PG fork）。

## Scale 边界

[yang-2020-pase §2.2.3, §4 + wang-2021-milvus Table 1]

| 配置 | 数据 | 应用 |
|---|---|---|
| 单 PG 实例 | million-scale, 100-512 dim | Alipay 推荐 / 人脸识别 / 图像版权 |
| 单 PG + ext storage | 1M × 512-d 实测 build / search | benchmark §4 |
| **billion-scale** | **不支持**（wang-2021 Table 1 ✗） | 需多 PG 实例 + 应用分片 |
| **distributed** | **不支持**（wang-2021 Table 1 ✗）| 同上 |
| Distributed PG (Citus/Greenplum) + PASE | 论文未涉及 | 推断可行未实测 |

> **wiki 解读**：PASE 是 wiki 已 ingest 系统中**唯一明确不支持 billion-scale 的工业级 vector DBMS**——这是 PG 单机能力天花板的代价。million-scale 应用（推荐、人脸、版权检测）在 Ant Financial 已 cover 大部分场景。

## 生产案例（§2.2.3）

| 应用 | 维度 | Recall 要求 | Latency 要求 | 数据规模 |
|---|---|---|---|---|
| **图像搜索（版权检测）** | **512** | R1@100 metric | **<300 ms** | **billions 级，但分多 PG 实例** |
| **Face recognition + PolarDB** | 512 | R1@1 > 90% | n/a | ~1M |
| **Alipay 推荐** | **40（低维）** | 80-90% | **<10 ms, 万 QPS** | n/a |
| **地铁人脸支付** | 256 | **R1@1 > 99%（严格）** | n/a | hundreds of thousands |

> **wiki 解读**：四个场景**都是 million-scale 内**——印证 wang-2021 Table 1 PASE billion-scale ✗ 标注。Alibaba 用 PASE 不是因为 PASE 能 scale 到 billion，而是因为 PG 已经在场景里部署，加 PASE 比换 system 便宜。

## 与 wiki 已有系统的对比

### 与 [AnalyticDB-V](./analyticdb-v.md) 的关系（同 Alibaba ecosystem）

| | AnalyticDB-V (ADBV) | **PASE** |
|---|---|---|
| Host DB 类型 | **OLAP** (AnalyticDB, MPP analytical) | **OLTP** (PostgreSQL) |
| 团队 | Alibaba Cloud DB team | **Ant Financial** team |
| 部署 | Alibaba Cloud SaaS | PG plugin 或 PolarDB integration |
| Distributed | ✓（按 ID / cluster partition） | ✗（PG 单机） |
| Billion-scale | ✓（Smart City 13B production） | ✗（million-scale only） |
| Compound query 哲学 | **4-plan CBO** | **iterative via amgettuple** |
| 主要用户 | OLAP workload + vector | **OLTP workload + vector** |

→ ADBV 与 PASE 是**Alibaba 同代但不同 host DB**的两条非 vector-first 路径——同 ecosystem，覆盖 OLAP / OLTP 两大场景。

### 与 [Milvus](./milvus.md) 的关系（vector-first vs PG-extended）

[per wang-2021-milvus Table 1] Milvus 把 PASE 列为竞品（PG ✓ Dynamic ✓ Attr filter，但 ✗ Billion / GPU / Multi-vector / Distributed）：
- Milvus：vector-first DBMS，从 ANN 算法出发——max scale, 但不支持 SQL OLTP join
- PASE：OLTP RDBMS-extended-vector，从 PG 出发——SQL + transaction 但不能 billion

→ 用户根据"已有 PG 投资 / SQL 用户基础大 / OLTP 工作负载"选 PASE；根据"vector workload 主体 / billion-scale"选 Milvus。

### 与 pgvector / pgvecto.rs（推断对比）

> [推测，wiki 未 ingest pgvector / pgvecto.rs source]

PASE (2020) 是 PG ANN extension 早期工作。后续 community 出现：
- **pgvector**：更广为使用的开源 PG vector extension（更简单，主要是 IVFFlat + HNSW）
- **pgvecto.rs**：Rust-based PG extension

PASE 论文未 cover 这些后继工作（早于其发布）。当前 wiki 未 ingest，所以与 pgvector 的对比 wiki 内 zero coverage。

## Open Questions

- **PASE vs pgvector / pgvecto.rs 实测对比**：wiki 未 ingest pgvector source；PASE 是否仍是 PG ext 最佳？
- **PASE 在 distributed PG (Citus / PolarDB) 下的表现**：论文未实验；推断可行但未实证
- **billion-scale PG + PASE**：单 PG 实例上限；多 PG 实例 + 应用分片是工业实践但论文未深入
- **Compound query 性能：PASE iterative vs ADBV 4-plan CBO**：两种工程哲学未直接 benchmark
- **PASE 维度 > 2000 实测**：论文 §2.4 描述 cross-page storage 但 §4 实验仅 SIFT 128-d / GIST 960-d
- **HNSW build 在 PASE 中慢**：论文实测 HNSW build 比 IVFFlat 慢 20×（GIST 1M: HNSW 20875s vs IVFFlat 372s）；PG 内核约束如何减缓这一差距？未深入
- **PASE 在 PostgreSQL 主线（PG 12/13/14...）下的兼容性**：论文 PG 11 时代；PG 内核版本演进影响未 follow
- **PASE 的 K' 静态选择限制 vs VBASE Iterator + RM**：[per benchmarks/vbase-8queries-recipe1m.md Table 5/6] PASE Q2 K'=100 recall 0.0567，K'=10000 latency 99p 36900 ms——PASE 没有 selectivity estimation（默认 0.5），无法自适应。VBASE 用 sampling 0.001 估计精度 q-error <1.1，且用 RM iterator 完全绕开 K' 决策。能否给 PASE 加 RM iterator？理论可行——`amgettuple` 已经是 single-step——但需要 (a) Phase 2 检测的 RM 形式化扩展，(b) selectivity sampling，(c) cross-index 协调

Cited by: 待 query 引用
