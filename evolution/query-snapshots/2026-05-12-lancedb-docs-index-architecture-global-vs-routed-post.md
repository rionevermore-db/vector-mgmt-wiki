---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-12
phase: post
ingest-context: lancedb-docs
wiki-pages-total: 78
cited-pages: [systems/lancedb.md, systems/chroma.md, systems/turbopuffer.md]
cited-count: 3
---

# Post-snapshot (lancedb-docs): index-architecture-global-vs-routed

## TL;DR (delta from chroma-docs post)

**LanceDB 引入 wiki 内第 8 种 architecture variant**: **(h) format-first 多模态 lakehouse architecture**——Lance format columnar partitioning + 多 storage tier (S3 / NVMe / local) + 分布式查询引擎. 与之前 7 variant (data routing 主导) 不同, LanceDB 是 **format-first 架构** — sharding 由 Lance format partitioning 内置, distributed engine 是 Enterprise 层 capability. 7 variant data-axis + LanceDB format-axis 形成完整 architecture taxonomy.

## Answer

### Architecture variant 8 种全景（updated 2026-05-12 post lancedb-docs）

| Variant | 代表 | Axis 类型 |
|---|---|---|
| (a) Global single index | DistributedANN | data routing |
| (b) Cluster + partition | SPANN @ Bing + Vespa OSS + Turbopuffer SPFresh + Chroma Cloud | data routing |
| (c) Hierarchical multi-level | Pinecone slab + Milvus segment | data routing |
| (d) No-index tenant partition | Vespa Streaming | data routing |
| (e) Namespace-as-primitive | Turbopuffer 100M+ namespace | data + tenant routing |
| (f) Reuse Postgres distributed | pgvector + Citus | general-RDBMS sharding |
| (g) Versioned collection tree fork | Chroma CoW fork | collection lifecycle |
| **(h) Format-first multimodal lakehouse (NEW)** | **LanceDB Lance format + Enterprise** | **OSS format substrate + distributed engine** |

→ wiki 内 architecture 8 种完整 axis: 6 种 data routing (a-e + f) + 1 种 collection lifecycle (g) + 1 种 format substrate (h). Each axis 与其他 axis orthogonal.

### LanceDB (h) format-first 哲学

[per sources/docs/lancedb/]

**Format-first 关键 properties**:
- Lance format 是 OSS standard (类似 Iceberg / Delta / Hudi)
- Storage tier 独立: S3 / GCS / Azure / NVMe / local — same Lance format
- Distributed query engine 是 Enterprise 层 — OSS 不需要
- Sharding 内置 Lance format columnar partitioning

**与 (b) SPANN / (e) Turbopuffer namespace 哲学对比**:
- (b) SPANN: 选 vector-specific routing (centroid + posting)
- (e) Turbopuffer: 选 tenant-specific routing (S3 prefix per tenant)
- (h) LanceDB: 选 **format-specific partitioning** (Lance columnar partition + distributed query)

→ **三者 orthogonal**——可在概念上叠加 (例: Lance format + namespace-as-primitive + SPANN routing) 虽然现实没 vendor 这样做.

### 已知盲区

- **LanceDB Lance format vs Iceberg/Delta/Hudi 在 OLAP + vector overlap 领域**: 不公开
- **LanceDB Enterprise 分布式 query engine 细节**: docs 不深入
- **Lance format + 8 种 architecture variant 组合可能性**: 理论 vs 实际 未明示

## Cited Pages

- [systems/lancedb.md](../../systems/lancedb.md)
- [systems/chroma.md](../../systems/chroma.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
