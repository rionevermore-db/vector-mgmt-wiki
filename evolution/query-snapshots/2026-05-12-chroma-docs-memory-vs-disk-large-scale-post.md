---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-12
phase: post
ingest-context: chroma-docs
wiki-pages-total: 77
cited-pages: [systems/chroma.md, systems/turbopuffer.md, systems/spann.md, topics/disk-vs-memory-ann.md]
cited-count: 4
---

# Post-snapshot (chroma-docs): memory-vs-disk-large-scale

## TL;DR (delta from pgvector-docs post)

**Chroma Cloud 是 wiki 内第 3 个 object-storage-primary production vector DB** (Turbopuffer SPFresh + Chroma SPANN + DistributedANN KV store)——三者都是 "shared backend" 哲学但具体 backend 不同 (object storage vs KV store). **关键 NEW**: SPANN family 在 object-storage-primary path 占主流——Turbopuffer (SPFresh extends SPANN) + Chroma Cloud (SPANN baseline) + Vespa OSS (SPANN as billion-scale option). 共 3 SPANN-family object-storage deployments active.

## Answer

### 与之前 ingest 的演进

| | pgvector-docs post | **chroma-docs post (NEW)** |
|---|---|---|
| Disk philosophy 6 类 | unchanged | **不变** |
| Object-storage-primary production vendors | Turbopuffer | **+ Chroma Cloud (2 个)** |
| SPANN-family object-storage path | Turbopuffer SPFresh extension | **+ Chroma Cloud baseline SPANN (2 个 production)** |

### Object-storage-primary path 全景（updated 2026-05-12 post chroma-docs）

| Vendor | ANN | Engine | OSS variant |
|---|---|---|---|
| Turbopuffer | SPFresh (SPANN + LIRE) | Rust | n/a (closed only) |
| **Chroma Cloud** | **SPANN (baseline)** | **Rust** | **Chroma Core OSS (HNSW single-node)** |
| DistributedANN (Bing) | DiskANN distributed | C++ | n/a (closed) |

→ **Object storage primary path 三 vendor production**——SPANN family 占 2/3 (Turbopuffer + Chroma Cloud), DistributedANN 是 Bing-specific distributed graph variant.

### Chroma Cloud 与 Turbopuffer 哲学一致性

[per sources/docs/chroma/llms-full.txt §Cloud + topics/disk-vs-memory-ann.md]

Chroma Cloud 与 Turbopuffer 共享:
- ✓ Object storage primary (S3/GCS)
- ✓ Rust execution engine
- ✓ SSD + memory cache (3-tier)
- ✓ Usage-based transparent pricing
- ✓ Cost-effectiveness via object storage
- ✓ > 90% recall production target

Differences:
- ANN: SPANN vs SPFresh
- OSS variant: Chroma Core HNSW (dev) vs no OSS for Turbopuffer
- Fork primitive: Chroma CoW fork vs Turbopuffer copy_from_namespace (full copy)
- Scale public claim: Chroma 不明示 vs Turbopuffer 3.5T+ explicit

### Disk path 决策表（updated 2026-05-12 post chroma-docs）

| Workload | 推荐 |
|---|---|
| Cost-sensitive + multi-tenant + RAG dev experience + AI agent (MCP) | **Chroma Cloud SPANN + object storage primary** |
| Cost-sensitive + multi-tenant + per-tenant namespace | Turbopuffer SPFresh |
| Single graph distributed + Microsoft infra | DistributedANN |
| Shared corpus 复杂 ranking | Vespa SPANN + 4-phase ranking |
| Shared corpus + Milvus 生态 | Milvus DISKANN |
| ≤100M docs + 已 Postgres | pgvector |

### 已知盲区

- **Chroma Cloud SPANN vs Turbopuffer SPFresh head-to-head实测**: 不公开
- **Chroma Cloud 大规模 production scale**: 不公开 hard limit
- **Chroma + BYOC object storage 选择 (S3 / GCS / R2 / Azure Blob)**: docs 明示 AWS + GCP, 其他 cloud 不详

## Cited Pages

- [systems/chroma.md](../../systems/chroma.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/spann.md](../../systems/spann.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
