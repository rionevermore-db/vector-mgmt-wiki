---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下索引架构选 (a) 全局单一索引 还是 (c) 层次路由结构？"
date: 2026-05-08
phase: post
ingest-context: yang-2020-pase
wiki-pages-total: 44
cited-pages: [queries/index-architecture-global-vs-routed.md, systems/pase.md, systems/analyticdb-v.md, systems/milvus.md]
cited-count: 4
---

# Post-snapshot (yang-2020-pase): index-architecture-global-vs-routed

## TL;DR (delta from wei-2020-analyticdb-v post)

**PASE 不参与 (a) vs (c) 千亿规模架构讨论**——论文 PASE billion-scale ✗。但 PASE 提供**第 6 种 (c) routing 形态**：PG 应用层分片到多 PG 实例。这是**应用层 routing**而非系统层（区别于 Milvus shard / Pinecone namespace / Manu vchannel / SPANN posting）——把 routing 决策推到应用代码，**与 PG 自身的 sharding extension（pg_shard / Citus）正交**。

## Answer

### PASE (c) 路由的应用层模式（NEW）

[per systems/pase.md "Scale 边界"]

PASE 单 PG 实例 million-scale 上限；处理 billions 需多 PG 实例 + 应用层 routing：

```
Application
  ├─ Hash(user_id) → 选 PG instance 1/2/.../N
  ├─ Each PG instance：local PASE HNSW + vector data
  └─ Cross-instance query：应用层 fan-out + merge
```

→ 这是**应用层手动 routing**——与系统层 (c) 不同：

| (c) 系统 | Routing 层 |
|---|---|
| Faiss IVF + HNSW coarse | 系统层（HNSW centroids） |
| SPANN | 系统层（SPTAG centroids） |
| Milvus 1.x segment | 系统层（segment-level coarse）|
| Milvus 2.x vchannel | 系统层（shard 路由 + Streaming Node） |
| SPFresh | 系统层（持续 LIRE 维护 centroids） |
| Pinecone Pod-based | 系统层（pre-configured pod hash） |
| Pinecone Serverless slab | 系统层（slab + namespace） |
| AnalyticDB-V | 系统层（cluster-based partition） |
| **PASE** | **应用层（user code routing）** |

→ PASE 的 (c) routing 是**应用层 ad-hoc**——失去系统级优化但获得 PG 全套 OLTP 能力 free 复用。

### Distributed PG + PASE 的可能性（NEW）

[per systems/pase.md "Scale 边界"]

理论可行但论文未实证：
- **Citus**：PostgreSQL 横向扩展 extension（按 shard key 分片）
- **PolarDB**：阿里云分布式 PG fork（共享存储架构）

PASE + Citus / PolarDB 组合：
- ✓ Citus 处理 routing layer（系统层）
- ✓ PASE 在每 shard 提供 vector ANN
- ⚠️ 论文未实测；推断 vector cross-shard query 需要 Citus / PolarDB 自身支持

### 完整 (c) 形态对比（updated with PASE）

[per queries/index-architecture-global-vs-routed.md] 当时归档说"千亿/万亿主流走 (c)"——本 ingest 后 (c) 形态完整覆盖：

| (c) 形态 | 起点 | scale 上限 | wiki 已 ingest |
|---|---|---|---|
| Library + manual sharding | Faiss + 用户胶合 | 1.5T (Meta) | ✓ |
| 算法系统 routing-as-mutable | SPFresh LIRE | 1B 实证 | ✓ |
| Vector-first DBMS | Milvus / Pinecone | 千亿+ | ✓ |
| OLAP-extended vector-aware partition | ADBV cluster-based | 13B production | ✓ |
| **OLTP RDBMS app-layer sharding** | **PASE** | **billions production via app-layer** | **✓（本次 ingest）** |
| HTAP DBMS + vector | TiDB / SingleStore | 未 ingest | ✗ |

### 与之前 ingest 的演进

| | wei-2020-analyticdb-v post | **yang-2020-pase post (NEW)** |
|---|---|---|
| (c) 形态数 | 5 | **6**（+ PASE 应用层） |
| Routing 实现层 | 系统层全部 | **+ 应用层模式（PASE 独有）** |
| (c) 千亿可行 system 数 | 4 | **不变**（PASE 不在千亿可行内） |

### 已知盲区

- **PASE + Citus / PolarDB 实测**：理论可行未实证
- **HTAP DBMS（TiDB / SingleStore）+ vector** routing：wiki 未 ingest
- **应用层 routing vs 系统层 routing 的工程权衡**：未量化对比

## Cited Pages

- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
- [systems/pase.md](../../systems/pase.md)
- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
- [systems/milvus.md](../../systems/milvus.md)
