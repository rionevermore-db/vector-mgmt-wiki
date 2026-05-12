---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-12
phase: post
ingest-context: pgvector-docs
wiki-pages-total: 76
cited-pages: [systems/pgvector.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (pgvector-docs): memory-vs-disk-large-scale

## TL;DR (delta from formal-2021-splade-v2 post)

**pgvector 在 memory-vs-disk 维度上 leverage Postgres storage layer 而非 vector-specific disk path**——之前 wiki 内 disk path (SPANN/DiskANN/Starling/FreshDiskANN/SPFresh/Turbopuffer object storage/DistributedANN KV store) 都是 vector-specific disk infrastructure. pgvector **inherits Postgres heap + shared buffers + WAL + replication** 的 mature storage layer. **关键 NEW**: pgvector disk path 是**第 6 种 disk philosophy**——"vector 是 Postgres tuple 一部分, disk layout 由 Postgres heap 管理". 不与 DiskANN/SPANN-style vector-aware disk layout 竞争, 而是**复用 50 年成熟 RDBMS storage stack**. Performance trade-off: less specialized but operationally simpler.

## Answer

### 与之前 ingest 的演进

| | formal-2021-splade-v2 post | **pgvector-docs post (NEW)** |
|---|---|---|
| Disk path categories | 5 (memory / NVMe / object storage / brute-force / distributed KV) | **6 (+ RDBMS heap + shared buffers + WAL)** |
| pgvector disk path | 未涵盖 | **NEW: Postgres heap-based, vector tuple in heap pages** |
| Disk philosophy 二分 | vector-specific (SPANN/DiskANN) vs general (object storage Turbopuffer / KV store DistributedANN) | **+ general-RDBMS (pgvector 复用 Postgres storage)** |

### pgvector disk path: Postgres heap-based (NEW)

[per sources/docs/pgvector/README.md + Postgres storage architecture]

**Vector tuple storage**:
- Vector 作为 Postgres tuple 一部分存 in heap pages (8 KB default)
- 大 vector (e.g., 768-d float32 = 3 KB) 通过 **TOAST** mechanism (Postgres standard) 处理 — compressed + out-of-line if needed
- Shared buffers (default 25% RAM) cache 热数据
- WAL 自动持久化 + replication

**HNSW index storage**:
- HNSW graph + neighbor list 存 in pgvector access method index relation
- Page-aligned via Postgres standard index infrastructure
- Index pages 也通过 shared buffers cache

**与 vector-specific disk path 对比**:
- DiskANN: PQ in DRAM + full vector on raw SSD page (page-aligned for sequential read)
- SPANN: centroid in-mem + posting list on SSD (page-aligned per posting)
- pgvector: vector tuple in heap page + index in index relation (Postgres standard, not vector-optimized layout)

→ **pgvector 不优化 vector-specific disk layout**, 而是**复用 Postgres standard layout**——简单但 less optimal for pure ANN workload.

### Disk path 全景表（updated 2026-05-12 post pgvector-docs）

| 方案 | DRAM | SSD/Disk layout | Disk philosophy |
|---|---|---|---|
| HNSW (memory) | full | n/a | memory-only |
| HNSW + quantization | quantized | n/a | memory-only |
| SPANN | centroids | posting lists (page-aligned) | **vector-specific** |
| DiskANN | PQ compressed graph | full vectors (page-aligned) | **vector-specific** |
| Starling | mid (compressed graph) | full + segment block shuffling | **vector-specific** |
| FreshDiskANN | mid | full + streaming merge | **vector-specific** |
| SPFresh | low | posting (raw SSD via SPDK) | **vector-specific** |
| Vespa Streaming | very low (45 B/doc) | scan-based | **vector-specific** |
| Turbopuffer SPFresh | cache | object storage (S3 primary) | **shared backend (object storage)** |
| DistributedANN | head index + caches | distributed KV store | **shared backend (KV store)** |
| **pgvector** | **Postgres shared buffers** | **Postgres heap + index relation** | **general-RDBMS (Postgres standard)** |

→ **pgvector 是第 6 类 disk philosophy**: 既不是 vector-specific specialized, 也不是 shared backend (object storage / KV store), 而是**复用 RDBMS heap + shared buffers + WAL**.

### Trade-off analysis: RDBMS storage path vs vector-specific path

[per pgvector philosophy + DiskANN/SPANN papers]

**Pros (pgvector RDBMS storage)**:
- ACID transaction guarantee on vector + scalar mixed updates
- WAL + replication + point-in-time recovery
- Shared infrastructure with existing Postgres workload
- VACUUM / autovacuum maintenance (mature tooling)
- Operational expertise reusable (DBA skill ports directly)

**Cons (pgvector RDBMS storage)**:
- No vector-aware page layout optimization (vs DiskANN page-aligned full vector + SPANN posting alignment)
- TOAST overhead for large vectors (out-of-line + decompress per fetch)
- Heap page (8 KB) not optimized for vector access pattern
- VACUUM impact on HNSW maintenance (vs Vamana incremental friendly)
- No native distributed routing (depends on Citus)

→ **pgvector ≤100M docs sweet spot 部分由 disk path 简单决定**——大规模 vector workload 需 vector-specific disk infrastructure.

### 决策表（updated 2026-05-12 post pgvector-docs）

| Workload | 推荐方案 |
|---|---|
| ≤100M docs + 已 Postgres + ACID + 操作简单 | **pgvector (RDBMS storage path)** |
| Cost-sensitive + multi-tenant | Turbopuffer SPFresh + object storage |
| Shared corpus 复杂 ranking | Vespa SPANN + 4-phase ranking |
| 千亿+ single graph distributed | DistributedANN |
| 百亿 SPANN/DiskANN-style 主流 | Milvus DISKANN / Vespa SPANN |

### 已知盲区

- **TOAST overhead on vector tuple**: 实测 large vector (1536-d / 3072-d) Postgres TOAST cost vs raw layout 不公开
- **pgvector shared buffers tuning best practice**: Vector workload 与 OLTP workload share buffer 的 trade-off 不公开
- **VACUUM cost on production HNSW write workload**: 不公开
- **pgvector + Citus distributed disk path**: 不公开
- **pgvector + DiskANN-style optimization possibility**: pgvector 未来是否 native page-aligned vector layout? Open

## Cited Pages

- [systems/pgvector.md](../../systems/pgvector.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
