---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-12
phase: post
ingest-context: lancedb-docs
wiki-pages-total: 78
cited-pages: [systems/lancedb.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (lancedb-docs): scale-tier-shifts

## TL;DR (delta from chroma-docs post)

**LanceDB 在 tier-shift 表上 unique 之处**: **OSS embedded + Enterprise managed 双模式 architectural continuity** — 同 Lance format core, dev → prod 不切换 codebase (vs Chroma OSS HNSW → Cloud SPANN). **关键 NEW**: LanceDB 是 wiki 内**架构连续性最强 dev-to-prod path**——dev 用 Lance format embedded, prod 用 Lance format Enterprise managed, same format + same query interface + same SDK.

## Answer

### 与之前 ingest 的演进

| | chroma-docs post | **lancedb-docs post (NEW)** |
|---|---|---|
| OSS-to-Cloud 架构连续性 | Chroma OSS HNSW → Cloud SPANN (不同 codebase) | **+ LanceDB OSS Lance format → Enterprise Lance format (same core)** |
| Tier-shift "deployment friction" | pgvector ≤100M sweet spot | **LanceDB: dev-to-prod 几乎 zero friction (same Lance format)** |

### LanceDB OSS → Enterprise 架构连续性（NEW）

[per sources/docs/lancedb/]

**vs Chroma OSS Core (HNSW) → Cloud (SPANN)**:
- Chroma: 不同 codebase, dev-to-prod migration 涉及 architectural change
- LanceDB: **same Lance format core**, OSS embedded + Enterprise managed only differ in distributed engine layer
- Lance format 是 OSS standard — dev / prod 都可独立操作

→ **LanceDB tier-shift 平滑度最强**——这是 lakehouse-philosophy 直接结果 (format 是 stable interface).

### Tier-shift 表 (updated 2026-05-12 post lancedb-docs)

| Tier | Scale | LanceDB path | Other vendor friction |
|---|---|---|---|
| Tier 1 (≤1M) dev | OSS embedded Lance format | pgvector / Chroma 任选 |
| Tier 2 (1-10M) | OSS embedded continues | Chroma OSS HNSW or Cloud, pgvector |
| Tier 3 (10-100M) | OSS embedded large + Enterprise option | Multiple vendor candidates |
| Tier 4 (100M-1B) | Enterprise (same Lance core) | Milvus / Vespa / Turbopuffer dedicated |
| Tier 5 (1-100B) | Enterprise (petabyte-scale multimodal) | Vespa + DistributedANN + Turbopuffer + Pinecone |
| Tier 6 (≥100B) | Enterprise (assertion-level) | DistributedANN + Turbopuffer (2 public production) |

→ **LanceDB tier-shift 在 vendor 内 same Lance core, 比 Chroma OSS-to-Cloud architectural break 更连续**.

### 已知盲区

- **LanceDB Enterprise hard scale ceiling**: 不公开
- **OSS embedded → Enterprise migration cost**: same format, but migration tooling cost 不公开
- **Lance format 跨 OSS / Enterprise 兼容性 implications**: 客户 self-host OSS 是否能 read Enterprise 写入的 Lance file? 反之?

## Cited Pages

- [systems/lancedb.md](../../systems/lancedb.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
