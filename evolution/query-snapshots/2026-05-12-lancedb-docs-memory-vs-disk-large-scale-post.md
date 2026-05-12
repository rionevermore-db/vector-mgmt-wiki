---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-12
phase: post
ingest-context: lancedb-docs
wiki-pages-total: 78
cited-pages: [systems/lancedb.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (lancedb-docs): memory-vs-disk-large-scale

## TL;DR (delta from chroma-docs post)

**LanceDB 引入 wiki 内第 7 类 disk philosophy**: **OSS columnar format on object storage / local disk**——Lance format 是 OSS standard, 可在 S3 / GCS / Azure / NVMe / local 自由部署. 与之前 6 类 (memory / NVMe vector-specific / object storage proprietary / brute-force / distributed KV / general-RDBMS heap) 不同, LanceDB 是 **format-first storage philosophy**——存储 substrate 由 OSS format 决定, 不是 vendor-specific layout.

## Answer

### Disk philosophy 7 类全景（updated 2026-05-12 post lancedb-docs）

| 方案 | DRAM / Cache | Disk layout | Disk philosophy |
|---|---|---|---|
| HNSW memory | full | n/a | memory-only |
| SPANN | centroids | posting lists | vector-specific |
| DiskANN | PQ compressed graph | full vectors | vector-specific |
| Starling | mid | full + block shuffling | vector-specific |
| Vespa Streaming | very low (45 B/doc) | scan-based | vector-specific |
| Turbopuffer SPFresh | cache | object storage primary | shared backend (object storage proprietary) |
| Chroma Cloud SPANN | cache | object storage primary | shared backend (object storage proprietary) |
| DistributedANN | head + caches | distributed KV store | shared backend (KV store) |
| pgvector | Postgres shared buffers | Postgres heap + index relation | general-RDBMS |
| **LanceDB Lance format (NEW)** | **vendor cache** | **OSS columnar format on any storage (S3 / GCS / NVMe / local)** | **format-first OSS standard** |

→ **LanceDB Lance format 是 wiki 内 first vendor 以 OSS columnar format 作 storage philosophy primary** — 与 Iceberg / Delta / Hudi 在 lakehouse 领域 parallel.

### Lance format vs other shared backend 路径

[per sources/docs/lancedb/ + comparison]

**Lance format 优势 over proprietary object storage layout**:
- Vendor-agnostic readability (DuckDB / Pandas / Polars 可直接 read Lance file)
- ML ecosystem 直接 query (LangChain / LlamaIndex / etc.)
- Future ecosystem unbundling (theoretically: 第三方 vector index over Lance file)

**Lance format vs Iceberg/Delta/Hudi philosophical parallel**:
- Iceberg/Delta/Hudi: OLAP lakehouse format
- Lance: ML / vector lakehouse format
- 三者都是 OSS standard format, 都支持 schema evolution + version control + columnar storage

### Disk path 决策表（updated 2026-05-12 post lancedb-docs）

| Workload | 推荐 |
|---|---|
| **Multimodal lakehouse + vector + raw data + ML ecosystem** | **LanceDB Lance format** |
| Cost-sensitive + multi-tenant cost | Turbopuffer SPFresh / Chroma Cloud SPANN |
| Shared corpus 复杂 ranking | Vespa SPANN + 4-phase ranking |
| 千亿+ single graph | DistributedANN |
| ≤100M docs + 已 Postgres | pgvector |

### 已知盲区

- **Lance format vs Iceberg/Delta/Hudi performance head-to-head on vector workload**: 不公开
- **LanceDB GPU index build + Lance format on object storage**: 实测不公开
- **Lance format + 第三方 vector engine integration**: 理论上 possible 实际 ecosystem maturity 不公开

## Cited Pages

- [systems/lancedb.md](../../systems/lancedb.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
