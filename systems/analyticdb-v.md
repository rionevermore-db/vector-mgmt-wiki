---
title: AnalyticDB-V（OLAP 扩展向量的 Hybrid Analytical Engine）
type: system
sources: [wei-2020-analyticdb-v, wang-2021-milvus]
related: [milvus.md, faiss.md, pase.md, ../concepts/vgpq.md, ../concepts/product-quantization.md, ../concepts/filtered-vamana.md, ../topics/attribute-filtering.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/disk-vs-memory-ann.md, ../benchmarks/analyticdb-v-vs-twostep.md, ../benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md]
created: 2026-05-08
updated: 2026-05-08
---

# AnalyticDB-V (ADBV)

**TL;DR**: Alibaba 在其 OLAP 数据库 **AnalyticDB** 之上扩展的 hybrid analytical engine——让用户**用 SQL 表达** vector + structured attribute 联合查询，而非 vector-first 系统加 SQL。Lambda 三层框架（streaming HNSW + batching VGPQ + serving merge）支持实时插入；自研 [VGPQ](../concepts/vgpq.md) (Voronoi Graph Product Quantization) 算法比 IVFPQ recall-vs-latency 更优；accuracy-aware 代价优化器在 4 plan 间根据 selectivity 自动选择。已部署到 Alibaba Cloud（13 billion records / 30 TB Smart City 车辆违章检测）。是 wiki 内**首个 OLAP-扩展-向量**路径系统——与 [Milvus](./milvus.md) / [Pinecone](./pinecone.md) "vector-first" 路径**正交**。[wei-2020-analyticdb-v §1, §3]

## 与 wiki 现有系统的定位差异

[per topics/attribute-filtering.md, wang-2021-milvus Table 1]

| | [Faiss](./faiss.md) | [DiskANN](./diskann.md) | [SPANN](./spann.md) | [Milvus](./milvus.md) | [Pinecone](./pinecone.md) | **AnalyticDB-V** |
|---|---|---|---|---|---|---|
| 起点 | ANN library | ANN 算法 | ANN 算法 | Vector DBMS | Vector SaaS | **关系 OLAP** |
| 路径 | 算法工具箱 | 算法系统 | 算法系统 | vector-first DBMS | vector-first SaaS | **OLAP 加 vector** |
| SQL 接口 | ✗ | ✗ | ✗ | 部分（schema 但非 SQL） | ✗ | **✓ 完整 SQL 方言** |
| Hybrid query 路径 | 用户胶合 | 不直接 | 不直接 | partition-based 5 策略 | metadata filtering | **CBO + 4 plan 自动选** |
| 真实数据集 vs SIFT/DEEP | n/a | SIFT1B | SIFT1B / SPACEV | SIFT/Deep10M | docs 不公开 | **AliCommodity 830M × 512-d + 21 结构列** |
| Update 模型 | 算法层无 | 不支持 | 不支持 | LSM segment | slab merge | **Lambda streaming + batching** |

**关键差异**：ADBV 是**第一类公开系统化的"OLAP DB + vector"路径**——这与 [Milvus](./milvus.md) / [Pinecone](./pinecone.md) 等"vector-first"路径**结构性不同**。后者的 schema/collection 是为 vector 设计的；ADBV 直接复用 AnalyticDB 的 SQL parser / query optimizer / storage engine（Pangu 分布式存储），把 vector 加进去作为"特殊的列"。

## 架构图

[wei-2020-analyticdb-v Fig 2]

```
┌────────────────────────────────────────────────┐
│  JDBC/ODBC Client                              │
├────────────────────────────────────────────────┤
│  Coordinator #1 / #2 / #3                      │
│   ├─ Parse / optimize SQL                      │
│   ├─ Hybrid query CBO                          │
│   ├─ Feature Extraction Service                │
│   └─ Dispatch to Read / Write nodes            │
├──────────────────────┬─────────────────────────┤
│  Read Node #1 ...    │  Write Node #1 ...      │
│   (SELECT path)      │   (INSERT/DELETE/UPDATE)│
│   Query processing   │                         │
│   ├─ Streaming layer │   Streaming layer       │
│   │  (HNSW for new)  │   ↓                     │
│   ├─ Batching layer  │   Daily Index MR Job    │
│   │  (VGPQ for old)  │   on Fuxi               │
│   └─ Serving layer   │                         │
│     (merge results)  │                         │
├──────────────────────┴─────────────────────────┤
│  Pangu Distributed Storage（baseline data）   │
└────────────────────────────────────────────────┘
```

## 数据流 / 控制流

### Lambda 三层框架（§3.2）

[wei-2020-analyticdb-v Fig 3, 4]

```
                  ┌────────────┐
   Insert ───→    │ Streaming  │  ← HNSW（新数据，in-memory，real-time）
                  └────────────┘
                       │
                       │ async merge (周期)
                       ↓
                  ┌────────────┐
                  │ Batching   │  ← VGPQ（baseline，on Pangu，offline 训练）
                  └────────────┘
                       │
   Query ←───────  ┌────────────┐
                  │ Serving    │  ← merge results from streaming + batching
                  └────────────┘
```

**关键设计点**：
- **Streaming layer 用 HNSW**：原因——HNSW 支持 real-time insert（增量加点）；缺点——内存重，但 incremental 数据量小
- **Batching layer 用 VGPQ**：原因——baseline 数据量大（PB 级），需要 codebook-based 压缩；缺点——offline 训练耗时（可异步）
- **Async merge**：incremental 数据合并到 baseline + 重建 VGPQ（HNSW 也丢弃）；旧 baseline 仍提供查询直到新 baseline 就绪
- **Per-record bitset (data-status)**：标记删除（tombstone style，类 Milvus / SPFresh）

### 写入路径

```
INSERT → Coordinator → 选 streaming 节点
                     → HNSW 加点 + WAL（Pangu）
                     → 200 OK
```

### 查询路径

```
SELECT ... → Coordinator → 同时发 streaming 节点 + batching 节点
                       → 结果合并 + data-status bitset 过滤删除
                       → 返回 client
```

## 关键设计决策

### 1. SQL 作为唯一 hybrid query 接口（§2.2）

ADBV **完整支持 SQL**：CREATE TABLE 含 vector field 类型、INSERT 用 array 字面量或 `FEATURE_EXTRACT(URL)`、SELECT 用 `DISTANCE(f, query)` 作为 ORDER BY。

**Trade-off**：用户不需要学新 API，关系 SQL 知识直接复用 vs 失去某些 vector-specific 优化（如 batch 接口、专用 streaming API）。

### 2. Lambda 框架处理 real-time（§3.2）

**Trade-off**：streaming + batching 双索引复杂度高 + merge 开销 vs 两层都好（streaming 实时 / batching 大规模）。论文 §6.6 实测 mixed write/read 8:2 仍 4400 QPS——架构成熟。

### 3. Clustering-based partitioning（§3.3）

不像传统数据库按 hash / range / list 分区——ADBV 按 **vector cluster centroid** 分区（数据 ingest 时 k-means 先分到最近 centroid）。Query 时仅 dispatch 到 closest N 分区（用户 query hint）。

**Trade-off**：partition 内 search 速度 10×-100× 提升 vs 跨 partition query / partition rebalance / centroid 漂移成本。论文承认 default 是按 structured column 分区，按 vector cluster 是 opt-in。

### 4. [VGPQ](../concepts/vgpq.md) (§4.2)

**核心算法贡献**——IVFPQ 的 successor。Voronoi diagram + subcell 分割让 query 仅扫覆盖 query 邻域的小段而非整个 IVF cell。详见 [concepts/vgpq.md](../concepts/vgpq.md)。

### 5. Accuracy-aware 4-plan CBO（§5）

[per topics/attribute-filtering.md "Strategy D"]

```
Plan A: B-tree + brute-force（小候选）
Plan B: PQ Knn Bitmap Scan（中等候选）
Plan C: VGPQ Knn Bitmap Scan（较大候选）
Plan D: VGPQ Knn Scan + filter（多数 record pass filter）
```

**Cost-based optimizer (CBO)** 计算每 plan 代价（受 selectivity α、超参 σ/β/γ 影响），选最低代价 + 满足 user-specified recall。**Accuracy-aware**：超参在离线 grid search per α-bin 中预调，online 仅 estimate α' 选 bin。

**Trade-off**：复杂度高（需要 cost model、超参 grid search、selectivity 估计）vs **首个对所有 selectivity 区间都有 optimal plan** 的工业系统。

> **wiki 解读**：这是 [topics/attribute-filtering.md] 中讨论的"Strategy D（cost-based AnalyticDB-V 方案）"原文——Milvus 的 partition-based Strategy E 后来声称比此快 13.7×，但 ADBV 的 4 plan 框架仍是 wiki 内最完整的 hybrid query optimizer 论述。

### 6. 复用 AnalyticDB OLAP 内核

ADBV 不重新设计 storage / scheduler / SQL parser——直接用：
- **Pangu** 分布式存储（Alibaba 自家，Snowflake 风格）
- **Fuxi** 资源管理 + job scheduler
- **AnalyticDB** Java 引擎为基础（vector ANN 用 C++ JNI 内嵌）

**Trade-off**：vector-specific 优化受限于 OLAP 内核 vs 工程成本极低 + 与 OLAP query 无缝融合（join、aggregation、subquery 都直接可用）。

## Scale 边界

[wei-2020-analyticdb-v §1, §6]

| 场景 | 配置 | 实测 |
|---|---|---|
| Smart City 车辆违章检测 | 70-node Alibaba Cloud cluster | **13B records, ~30 TB**, 24h 稳定 |
| Freshippo（盒马）超市 | 不公开规模 | **800M × 512-d**, 4000 QPS 峰值, 80%+ hybrid query |
| SIFT1B / Deep1B benchmark | 16 node | 4400 QPS sustained, 4-16 node 线性 |
| 万亿 | 论文未实测 | 推断需更多节点 |

> **wiki 解读**：13B records / 30 TB 是 wiki 已 ingest 的工业部署中**第二大**（仅次于 Meta 1.5T Faiss），且**与万亿向量 wang-2021-milvus 隐含的 Faiss 案例不同**——这是真实 production 实测而非论文 case study。

## 与 wiki 已有系统的对比

### 与 Milvus 的关系

[per wang-2021-milvus Table 1]：Milvus 1.x 论文把 ADBV 列为竞品——能力对比：

| | ADBV | Milvus 1.x |
|---|---|---|
| Billion-scale | ✓ | ✓ |
| Dynamic data | ✓ | ✓ |
| GPU | ✗ | ✓ |
| Attribute filtering | ✓ (4-plan CBO) | ✓ (5-plan partition-based) |
| Multi-vector query | ✗ | ✓ |
| Distributed | ✓ | ✓ |

[per topics/attribute-filtering.md] Milvus partition-based Strategy E **比 ADBV cost-based Strategy D 快 13.7×**（Milvus 论文实测）——但 ADBV 仍是 strategy D 的工业开创者。

### 与 OLAP 系统的关系

[wei-2020-analyticdb-v §7] 论证 ADBV 与传统 OLAP 系统差异：
- OLAP（Vertica, Greenplum, Spark-SQL, Redshift, BigQuery, AnalyticDB）—— "**only on traditional structured datasets**"
- vector search engines（Faiss, ES vector plugin, GRIP, Milvus）—— "**only on vector data**"
- ADBV —— **first to natively combine OLAP + vector with cost-based optimization**

## 生产案例

[wei-2020-analyticdb-v §6.7]

**Smart City Transportation - 车辆违章检测**：
- 20,000+ 路口摄像机视频流
- 每帧 → vehicle detection (faster-RCNN, YOLO) → image embedding
- 13 billion records, 30 TB 累积
- Hybrid query：按 timestamp / location / camera-id / color 过滤 + 找视觉相似车辆
- 实时查询从"hundreds of seconds → milliseconds"

**Freshippo 盒马超市**：
- 数字化零售商品库
- 800 million × 512-d vectors
- 21 structured columns（color, sleeve_type, style, create_time, ...）
- **4000 QPS peak, 80%+ are hybrid queries**

## Open Questions

- **VGPQ 在 Milvus / 其他系统的 portability**：[per concepts/vgpq.md] VGPQ 仅在 ADBV 集成；其他系统是否能复用？
- **Accuracy-aware CBO 的 selectivity 估计精度**：α' estimation 出错时 plan 选错的代价；论文未深入
- **Lambda 框架的 streaming/batching 一致性**：query 时刚 insert 的 vector 是否在 streaming layer + 已 merge 到 batching layer 中重复 hit？data-status bitset 处理删除但**重复 hit** 论文未明示
- **Multi-vector query**：[per wang-2021-milvus Table 1] 标 ✗——Milvus 论点之一
- **GPU support**：论文未提，wang-2021-milvus 标 ✗
- **Cluster-based partitioning 在 vector 漂移下的行为**：centroid 不变但数据分布变了，partition 不平衡如何处理？论文 §3.3 末尾建议 re-cluster 但未详述触发条件
- **ADBV 与 [Manu/Milvus 2.x](./milvus.md) 的 lambda 与三层架构对比**：两者同代但来自不同公司（Alibaba vs Zilliz）；具体 trade-off wiki 未直接 benchmark
- **Pangu 与 S3/HDFS 的差异**：Pangu 是 Alibaba 自家闭源——其他云用户需 ADBV-like 系统时如何替换？

Cited by: 待 query 引用
