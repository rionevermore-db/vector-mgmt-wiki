---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-12
phase: post
ingest-context: pgvector-docs
wiki-pages-total: 76
cited-pages: [systems/pgvector.md, systems/distributedann.md, systems/vespa.md]
cited-count: 3
---

# Post-snapshot (pgvector-docs): index-architecture-global-vs-routed

## TL;DR (delta from formal-2021-splade-v2 post)

**pgvector 不 native 支持 (a)-(e) 任一 sharding 路径**——pgvector 仅是 Postgres extension, distributed routing **委托给 Postgres ecosystem (Citus / streaming replication / application sharding)**. **关键 NEW**: pgvector 在 sharding architecture 上的哲学不是 "选择哪种 sharding", 而是 "**复用 Postgres distributed solution 而非 build vector-specific routing**". 这与 wiki 内 5 vendor (Milvus / Pinecone / Vespa / Turbopuffer / DistributedANN) **各自 build 不同 vector-specific routing** 形成对比. pgvector 千亿规模 viable 路径 (理论上) = pgvector + Citus, 但 production case 不公开.

## Answer

### 与之前 ingest 的演进

| | formal-2021-splade-v2 post | **pgvector-docs post (NEW)** |
|---|---|---|
| 5 架构 (a)-(e) sharding | sparse + dense 各自 axis | **不变** |
| pgvector 在 5 架构内位置 | 未涵盖 | **NEW: pgvector NOT in (a-e), 委托给 Postgres ecosystem (Citus / replication / app sharding)** |
| Sharding architecture 哲学 二分 | "build vector-specific routing" | **+ "reuse Postgres distributed solution"** |

### pgvector sharding 哲学（NEW philosophy）

[per sources/docs/pgvector/README.md §Scaling]

**pgvector 不实现 vector-specific routing**:
- 不 build distributed graph (vs DistributedANN single graph distributed via KV store)
- 不 build cluster + posting list (vs SPANN centroid routing)
- 不 build namespace-as-primitive (vs Turbopuffer S3 prefix routing)
- 不 build segment + coordinator (vs Milvus disaggregated)

**pgvector distributed scale 路径**:
- **PostgreSQL streaming replication**: read replicas (primary single-writer limit)
- **Citus**: Postgres distributed extension, sharding by hash/range
- **Application-level sharding**: 多 Postgres instances, application 路由
- **Foreign Data Wrapper (FDW)**: 跨 Postgres / external systems federation

→ **pgvector 复用 Postgres distributed primitives**, 不 build new vector-specific routing.

### 5 架构 + 1 (pgvector 复用 Postgres) 完整景观

[per kusupati-2022-matryoshka post 5 architecture + pgvector]

| 架构 | 代表 | Sharding-axis 类型 |
|---|---|---|
| (a) Global single index | DistributedANN (Bing) | vector-specific (single distributed graph) |
| (b) Cluster + partition (SPANN-style historical) | SPANN (历史 @ Bing 2021-2024) + Vespa OSS | vector-specific (partition + routing) |
| (c) Hierarchical routing | Pinecone slab + Milvus segment + Vespa SPANN | vector-specific (multi-level routing) |
| (d) No-index tenant partition | Vespa Streaming | vector-specific (per-tenant disk scan) |
| (e) Namespace-as-primitive | Turbopuffer | vector-specific (100M+ S3 prefix) |
| **(f) Reuse Postgres distributed (NEW)** | **pgvector + Citus** | **general-RDBMS (Postgres standard sharding)** |

→ pgvector 是 wiki 内第 6 类 architecture——**复用 general-purpose distributed RDBMS primitives** instead of build vector-specific routing.

### 千亿/万亿决策表（updated 2026-05-12 post pgvector-docs）

| 场景 | 推荐 |
|---|---|
| 千亿 single corpus + 复杂 ranking | Vespa SPANN + 4-phase ranking |
| 千亿 single corpus + 6× throughput | DistributedANN single-graph distributed |
| 多租户 multimodal SaaS | Turbopuffer namespace-as-tenant |
| 千亿 + multi-modal + 多 index_type | Milvus DISKANN + 多 vector field |
| Pure dense large-scale managed SaaS | Pinecone slab |
| **≤100M docs + 已 Postgres workload + 加 vector** | **pgvector (复用 Postgres distributed primitives)** |
| 千亿 pgvector + Citus | 理论可行, **production case zero (industry assertion against)** |

### pgvector 千亿 viable but rare 的 production reality

[per Citus docs + industry surveys]

**Theoretical path**: pgvector + Citus = distributed sharded Postgres + per-shard HNSW/IVFFlat
- 16 节点 Citus cluster + per-shard pgvector
- Query fanout via Citus query planner
- Per-shard HNSW build

**Production reality**:
- 千亿规模 pgvector + Citus case **不公开**
- Industry assertion: pgvector ≤100M sweet spot, 千亿主流仍是 dedicated vector DBMS
- 原因: vector-aware sharding (per (a)-(e) routes) provides specialized capabilities (centroid routing, head index, single graph distributed, namespace fanout) that Citus hash sharding doesn't optimize

### 已知盲区

- **pgvector + Citus 实际 production 千亿 case**: 不存在公开
- **Citus vs vector-specific sharding 性能对比**: 不公开
- **pgvector ecosystem 未来是否 build vector-aware Citus extension**: Open
- **Foreign Data Wrapper 跨 vector DBMS federation 实际 case**: 不公开

## Cited Pages

- [systems/pgvector.md](../../systems/pgvector.md)
- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/vespa.md](../../systems/vespa.md)
