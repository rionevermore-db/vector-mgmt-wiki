---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-12
phase: post
ingest-context: chroma-docs
wiki-pages-total: 77
cited-pages: [systems/chroma.md, systems/turbopuffer.md, systems/spann.md]
cited-count: 3
---

# Post-snapshot (chroma-docs): index-architecture-global-vs-routed

## TL;DR (delta from pgvector-docs post)

**Chroma Cloud architecture 与 (c) hierarchical routing 一致**——SPANN-based + centroid routing + posting list on object storage. 与 Turbopuffer namespace-as-primitive (e) 不同, **Chroma 没有强 namespace 模型**, collection 是主要 isolation unit + collection forking via CoW 是版本 isolation primitive. **关键 NEW**: Chroma collection fork (CoW) 是 wiki 内**第 7 种 architecture variant**——branching collection tree per collection, 不是 namespace fanout, 不是 single graph distributed, 是 **versioned collection tree** primitive.

## Answer

### 与之前 ingest 的演进

| | pgvector-docs post | **chroma-docs post (NEW)** |
|---|---|---|
| 5 架构 (a)-(e) + (f) pgvector general-RDBMS | 6 | **不变 + Chroma 路径** |
| Chroma 在 5+1 内位置 | n/a | **(c) hierarchical routing (SPANN) + 新 axis: collection fork tree** |
| Collection version isolation primitive | Qdrant Aliases / Vespa AppPkg / Turbopuffer namespace | **+ Chroma CoW fork tree (independent axis)** |

### Chroma Cloud architecture 在 5+1 内位置

[per sources/docs/chroma/llms-full.txt §Cloud]

Chroma Cloud:
- (c) Hierarchical routing: ✓ SPANN centroid + posting on object storage (与 Turbopuffer SPFresh / Vespa SPANN 同 family)
- (e) Namespace-as-primitive: 部分支持, 但**不像 Turbopuffer 100M+ namespace as architectural primitive**. Chroma 用 tenant → database → collection 三层 hierarchy
- (f) reuse Postgres distributed: ✗ (不是 RDBMS extension)

→ Chroma 主要走 (c) hierarchical routing 路径——但加 **collection fork tree** 作 version isolation 的额外 axis.

### Chroma collection fork tree (NEW 7th architecture variant)

[per sources/docs/chroma/llms-full.txt §Collection Forking]

不是 traditional sharding:
- 不是 data partitioning (vs SPANN centroid / Pinecone slab / Milvus segment)
- 不是 namespace fanout (vs Turbopuffer 100M+)
- 不是 no-index tenant scan (vs Vespa Streaming)
- 不是 single graph distributed (vs DistributedANN)
- 不是 SQL-extension general distributed (vs pgvector + Citus)
- **是 versioned collection tree** — fork tree 是 collection version hierarchy

**Use cases**:
- Prompt engineering iteration: fork production collection per prompt variant
- PR-based dev workflow: fork repo index per PR
- A/B testing: fork production for variant testing
- Version control: collection snapshot at point in time

→ **这不是 scaling architecture, 是 collection lifecycle architecture**——orthogonal axis 与 traditional sharding routes.

### 7 种 architecture variant 全景（updated 2026-05-12 post chroma-docs）

| Variant | 代表 | Axis 类型 |
|---|---|---|
| (a) Global single index | DistributedANN | data routing (single graph distributed) |
| (b) Cluster + partition routing | SPANN @ Bing 历史 + Vespa OSS + Turbopuffer SPFresh + **Chroma Cloud SPANN** | data routing (cluster + partition) |
| (c) Hierarchical multi-level | Pinecone slab + Milvus segment | data routing |
| (d) No-index tenant partition | Vespa Streaming | data routing |
| (e) Namespace-as-primitive | Turbopuffer 100M+ namespace | data + tenant routing |
| (f) Reuse Postgres distributed | pgvector + Citus | general-RDBMS sharding |
| **(g) Versioned collection tree fork (NEW)** | **Chroma CoW fork** | **collection lifecycle / version isolation** |

→ **(g) 是 collection-lifecycle axis** — 不是 scaling axis 但是 production deployment hygiene primitive.

### 已知盲区

- **Chroma collection fork tree large-scale**: 256 edge limit 实测 / production case 不公开
- **Chroma (c) SPANN + (g) fork 联合**: forked collection 是否独立 SPANN index? CoW 是否覆盖 index 不明示
- **Chroma multi-tenant namespace**: tenant → database → collection 三层 hierarchy 实际 production scale 不公开

## Cited Pages

- [systems/chroma.md](../../systems/chroma.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/spann.md](../../systems/spann.md)
