---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-11
phase: post
ingest-context: turbopuffer-docs
wiki-pages-total: 68
cited-pages: [systems/spann.md, systems/spfresh.md, systems/turbopuffer.md, systems/vespa.md, systems/milvus.md, systems/pinecone.md, concepts/lire.md]
cited-count: 7
---

# Post-snapshot (turbopuffer-docs): index-architecture-global-vs-routed

## TL;DR (delta from vespa-docs post)

**Turbopuffer 给 wiki 引入第 5 种 architecture (e)**：**namespace-as-architectural-primitive**——不是 (a) global / (c) routed / (d) Vespa Streaming no-index 中任意一种，而是 **"千万 namespace × 每 namespace 独立 SPFresh + 独立 S3 prefix"**。**关键 NEW**：与 Vespa 5-layer fanout 路由不同——Turbopuffer 仅 2 层（LB → query node, namespace 由 path 中包含），架构最扁平；但**namespace 数量级 100M+ 远超 peer DBMS** —— 一种全新形态的"超扁平 + 超多分区"路由。

## Answer

### 与之前 ingest 的演进

| | vespa-docs post | **turbopuffer-docs post (NEW)** |
|---|---|---|
| (a) 全局索引 production case | 不变 (Milvus single segment等) | **不变** |
| (c) 路由架构 production case | Pinecone slab + Milvus segment + SPANN + Vespa SPANN | **不变** |
| (d) 无索引 + 物理分区 | Vespa Streaming | **不变** |
| **(e) namespace-as-primitive** | wiki 未列 | **+ Turbopuffer (新 category)** |
| 路由层次最多 | Vespa 5-layer (container/group/SPANN/per-shard/global) | **不变 Vespa 最深；Turbopuffer 最浅 (2 层)** |

### Turbopuffer 2-layer routing topology（NEW）

[per sources/docs/turbopuffer/llms-full.txt §architecture]

```
Query (POST /v2/namespaces/{ns}/query) →
  Layer 1: LB (geographic / load-aware routing to region cluster)
    → Layer 2: Query node (stateless, locality-prefer to namespace's last node)
      → NVMe cache hit? → return (warm)
      → NVMe miss? → S3 read (cold, 3-4 roundtrip × 100ms)
```

**仅 2 层 routing** vs Vespa 5-layer:
- 没有 group selection（namespace 是 hash partition 隐含）
- 没有 SPANN centroid routing（centroid 是 namespace 内部 SPFresh detail）
- 没有 multi-phase ranking（仅 first-stage retrieval, 推到 client app rerank）
- 没有 global container rerank

**架构最扁平 + 分区粒度最细**——Turbopuffer 哲学是"少层抽象 + 多 partition"。

### 五种架构 (a)-(e) 全景对比表（updated 2026-05-11 post turbopuffer-docs）

| 架构 | 适用规模 | Partition 粒度 | Layer 数 | Production case |
|---|---|---|---|---|
| (a) 全局单一索引 | ≤百亿 + 内存装得下 | n/a (单 graph) | 1 | HNSW single-segment Milvus / Weaviate single-class |
| (c) 层次路由 | 百亿-万亿 | content cluster group OR slab OR segment OR centroid | 2-5 | **Vespa SPANN (5)**, Pinecone slab (3), Milvus segment (3), SPANN Bing (3) |
| (d) 无索引 + tenant 分区 | per-tenant ≤百万 docs | per-tenant disk partition | 2 | Vespa Streaming Search |
| **(e) namespace-as-primitive** | per-namespace ≤500M docs / 全局任意大 | **per-namespace S3 prefix (100M+)** | **2 (最浅)** | **Turbopuffer SPFresh** |

### Turbopuffer (e) 与 (c) 的核心差异

| | (c) 层次路由 | **(e) namespace-as-primitive** |
|---|---|---|
| Partition 是 query path 中明示？ | 隐式（query 触发 router） | **显式** (`/v2/namespaces/{ns}/query`) |
| Partition 跨界 query | 默认 (router fanout) | **不支持**——必须 multi-query 应用层 fanout |
| Partition 数量级 | 数千 (Pinecone slab / Vespa group / Milvus segment) | **百万-亿** (Turbopuffer namespace) |
| 每 partition 独立 schema | 部分 (Pinecone yes per-namespace；Milvus yes per-collection) | **YES** (每 ns 独立 vector dim / cell type / index config) |
| Failure isolation | router 共享状态 | **per-namespace 完全独立**（任意 node serve 任意 ns） |
| Multi-tenancy 默认 | filter-based | **architectural** (1 tenant → N namespaces) |

### 架构选型决策表（updated 2026-05-11 post turbopuffer-docs）

| 架构 | 适用规模 | 适用 workload | Production case |
|---|---|---|---|
| (a) 全局单一索引 | ≤百亿 + 内存装得下 | shared corpus + 简单 query | HNSW single-segment |
| (c) 层次路由 | 百亿-万亿 | shared corpus + 复杂 query / ranking | **Vespa SPANN + group routing**, Pinecone slab, Milvus segment |
| (d) 无索引 + tenant 分区 | per-tenant ≤百万 docs | multi-tenant + per-tenant query 隔离 | Vespa Streaming Search |
| **(e) namespace-as-primitive** | 全局任意大, per-ns ≤500M docs | **multi-tenant SaaS + per-tenant data isolation 默认设计** | **Turbopuffer SPFresh + S3** |

### 千亿/万亿规模主流仍是 (c)，但 (e) 是新挑战者

**千亿+ 规模**:
- (c) 层次路由仍是 shared-corpus 千亿规模主流（Vespa SPANN / Pinecone slab / Bing SPANN）
- **(e) Turbopuffer 是 first OSS-known production at 3.5T+ docs**——但**不是 shared corpus, 是 100M namespace 累积**
- 两者**workload assumption 不同**：(c) 假设 query 必须看 corpus 大部分；(e) 假设 query 仅访问 1 namespace

**多租户 SaaS B2B (per-tenant data isolation 默认)**:
- (c) 不够——filter-based multi-tenancy 在百万 tenant 时 overhead 大
- (d) 需自定义 disk scan 路径——仅 Vespa Streaming 提供
- **(e) Turbopuffer 是 architectural default**——namespace 是 100M+ 一等公民

### 已知盲区

- **Turbopuffer 100M namespace 实际 routing 性能**：cold p99 / p999 当 namespace 不在 NVMe cache 时
- **(e) 与 (c) 同 vendor 内能否混合**：Turbopuffer single namespace 是否内部用 (c) 风格 routing？SPFresh centroid sub-routing 是否在 namespace 内？docs 不细谈
- **(e) routing 跨 region failure mode**：multi-region active-active workload 下 routing
- **(c) shared-corpus query vs (e) per-namespace query 性能 head-to-head**：1B docs 单 ns 跑 Turbopuffer vs 1B docs 分散 Vespa SPANN——延迟差距 unknown

## Cited Pages

- [systems/spann.md](../../systems/spann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/pinecone.md](../../systems/pinecone.md)
- [concepts/lire.md](../../concepts/lire.md)
