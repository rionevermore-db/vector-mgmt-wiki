---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-12
phase: post
ingest-context: chroma-docs
wiki-pages-total: 77
cited-pages: [systems/chroma.md, systems/turbopuffer.md, topics/disk-vs-memory-ann.md]
cited-count: 3
---

# Post-snapshot (chroma-docs): scale-tier-shifts

## TL;DR (delta from pgvector-docs post)

**Chroma 给 tier-shift 增加 "deployment mode tier-shift" 新维度**——Chroma OSS Core (single-node, HNSW) 与 Chroma Cloud (Distributed, SPANN) 是**同 vendor 内 architectural tier-shift**: dev → prod 不需要 vendor migration, 但需要 deployment mode swap. **关键 NEW**: 之前 wiki tier-shift assume vendor 单一架构, Chroma 引入 "**deployment mode pre-built tier-shift**" — vendor 提供 dev path + prod path same brand, 不需 migrate cross vendor.

## Answer

### 与之前 ingest 的演进

| | pgvector-docs post | **chroma-docs post (NEW)** |
|---|---|---|
| Tier-shift 主要维度 | scale × storage × parallel × deployment friction (pgvector vs separate DBMS) | **+ vendor-internal deployment mode tier-shift (OSS Core → Cloud)** |
| Same-vendor architectural shift | 罕见 | **NEW: Chroma OSS Core HNSW → Cloud SPANN within same vendor** |

### Vendor-internal deployment mode tier-shift（NEW）

[per sources/docs/chroma/llms-full.txt §Cloud + §Distributed Chroma]

**典型 Chroma trajectory**:
- Tier 1 (dev / prototype, ≤1M docs): Chroma OSS Core single-node, HNSW, local SQLite
- Tier 2-3 (production, 1M-100M docs): Chroma Cloud Distributed, SPANN, object storage primary
- Tier 4+ (>100M docs): Chroma Cloud single-tenant / BYOC

**vs 其他 OSS + cloud vendor**:
- Milvus + Zilliz Cloud: 同 codebase, cloud-native disaggregated (no architectural change)
- Qdrant + Qdrant Cloud: 同 codebase, single binary + cloud orchestration
- Weaviate + Weaviate Cloud: 同 codebase, single binary + cloud
- **Chroma: OSS Core 与 Cloud Distributed 是不同 codebase + 不同 ANN**

→ Chroma deployment mode tier-shift **不是平滑 scaling** (vs Milvus/Qdrant/Weaviate same code path), 而是 **OSS-to-Cloud "migration with architecture change"**.

### Tier-shift 表（updated 2026-05-12 post chroma-docs）

| Tier | Scale | Chroma path | Other vendor path |
|---|---|---|---|
| ≤1M | dev / prototype | **Chroma OSS Core (HNSW)** | pgvector / Milvus / Qdrant 任选 |
| 1-10M | small prod | **Chroma Cloud (SPANN)** OR migrate to other | pgvector / Qdrant / Weaviate |
| 10-100M | mid prod | Chroma Cloud / Cloud + BYOC | Milvus / Qdrant / Weaviate / Pinecone |
| 100M-1B | large prod | Chroma Cloud (scale不明示) | Milvus / Pinecone / Turbopuffer / Vespa |
| 1B-100B | very large | Chroma Cloud + BYOC (assertion) | Vespa + Pinecone + Turbopuffer + DistributedANN |
| ≥100B | mega scale | n/a (no public claim) | DistributedANN + Turbopuffer (only) |

### 已知盲区

- **Chroma OSS Core → Cloud migration smoothness**: 同 API 但不同 ANN; 实际 case study 不公开
- **Chroma Cloud scale ceiling**: hard limit 不明示
- **Chroma + BYOC 大规模 production case**: 不公开

## Cited Pages

- [systems/chroma.md](../../systems/chroma.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
