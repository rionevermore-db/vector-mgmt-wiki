---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-12
phase: post
ingest-context: pgvector-docs
wiki-pages-total: 76
cited-pages: [concepts/hnsw.md, systems/pgvector.md, systems/pase.md, systems/vbase.md]
cited-count: 4
---

# Post-snapshot (pgvector-docs): hnsw-vs-nsg-selection

## TL;DR (delta from formal-2021-splade-v2 post)

**pgvector ingest 把 HNSW production deployment 数从 4 OSS vector DBMS 推到 5**——pgvector 是 wiki 内**第 5 个 production OSS** 选 HNSW (Milvus + Qdrant + Weaviate + Vespa + **pgvector**), 加上 IVFFlat 作 build-friendly 替代. **关键 NEW**: pgvector 是**部署最广 OSS HNSW implementation** (per industry surveys), industry "默认 HNSW choice" 是 pgvector. NSG production case 仍零.

## Answer

### 与之前 ingest 的演进

| | formal-2021-splade-v2 post | **pgvector-docs post (NEW)** |
|---|---|---|
| HNSW production OSS DBMS | 4 (Milvus + Qdrant + Weaviate + Vespa) | **5 (+ pgvector)** |
| Postgres-extension path HNSW | PASE (Ant only) + VBASE (research) | **+ pgvector (industry-most-deployed)** |
| HNSW industry deployment breadth | 4 OSS DBMS production | **pgvector universal Postgres managed services support** |
| NSG production | Taobao 2B + NSSG | **不变** |

### pgvector HNSW production deployment ubiquity（NEW）

[per sources/docs/pgvector/README.md + industry surveys]

pgvector 是 wiki 内 HNSW deployment**最广** vendor:
- **All major managed Postgres services 默认支持**: Supabase / Neon / RDS / CloudSQL / Aiven / Crunchy / Azure DB for PG / TimescaleDB Cloud / CockroachDB
- DB-Engines 2024-2025 Postgres extension surveys: pgvector 是 vector retrieval 最 mentioned Postgres option
- HackerNews / dev community: "starting your AI app" 主流推荐

→ 与 4 separate OSS DBMS 相比, pgvector 通过 Postgres ecosystem 直接覆盖 "已 Postgres + 加 vector" 巨大 deployment surface.

### pgvector HNSW 参数完整暴露（NEW production reference）

[per README.md]

```sql
-- Build params (per-index)
CREATE INDEX ON items USING hnsw (embedding vector_cosine_ops) 
WITH (m = 16, ef_construction = 64);

-- Query params (per-session)
SET hnsw.ef_search = 100;
SET hnsw.iterative_scan = relaxed_order;  -- v0.8.0+
```

| Param | pgvector default | typical tuning range |
|---|---|---|
| m (max connections per layer) | 16 | 8-64 |
| ef_construction | 64 | 64-200 |
| ef_search | 40 | 40-200 (iterative scan auto-adjusts) |

### HNSW vs IVFFlat 在 pgvector 内的选择（NEW dimension）

[per README §IVFFlat + HNSW]

pgvector 是 wiki 内 vendor 中**HNSW 与 IVFFlat 共存且 explicit 选择** 的:
- HNSW: 查询性能更高, build 慢, memory 重
- IVFFlat: build 快, memory 轻, 需要 data 已 present (不能 build on empty)

**典型选择**:
- Production write-heavy + 小批量 build → HNSW (no training needed)
- Bulk-load 一次性 + read-heavy → IVFFlat (build 极快, lists = sqrt(rows) for >1M)

### HNSW vs NSG 选择决策表（updated 2026-05-12 post pgvector-docs）

| 工程考量 | HNSW (5 OSS DBMS + pgvector) | NSG |
|---|---|---|
| Production OSS DBMS deployment 数 | **5 (Milvus + Qdrant + Weaviate + Vespa + pgvector)** | NSSG + Taobao only |
| Postgres extension 路径 | **pgvector primary + PASE + VBASE** | ✗ |
| Industry surveys top mention | **pgvector** | n/a |
| Managed services support | universal (Supabase / RDS / Aiven / etc.) | n/a |
| Filter-aware (ACORN-style) | **pgvector v0.8.0 iterative scan generalized** | ✗ |
| GPU-native | ✓ via CAGRA | ✗ |
| Streaming + α=1.2 patch | HFresh / FreshVamana | ✗ |

### 选择决策（updated 2026-05-12 post pgvector-docs）

- **已部署 Postgres workload + 加 vector retrieval** → **pgvector** (universal managed support, lowest deployment friction)
- 大规模 production vector workload + 独立 DBMS → Milvus / Qdrant / Weaviate / Vespa
- 闭源 SaaS managed → Pinecone / Turbopuffer
- 复杂 ranking pipeline + tensor framework → Vespa
- 学术 single-vector TopK + 静态 → NSG (academic only)

NSG 在 wiki 内 production OSS DBMS 仍**零部署**——HNSW deployment 占绝对主流, **pgvector 让 HNSW 在 Postgres ecosystem 中也成 default**.

### 已知盲区

- **NSG production**: 仍零 (无 Postgres extension 支持 NSG)
- **pgvector HNSW vs Qdrant Filterable HNSW filter behavior 实测**: 不公开
- **pgvector iterative_scan 算法细节 vs Qdrant/Weaviate/Vespa ACORN implementation**: head-to-head zero coverage
- **pgvector scale ceiling for HNSW** vs separate DBMS (Milvus / Vespa): production case 不公开 (assertion: pgvector ≤100M docs sweet spot, 但具体上限未量化)

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [systems/pgvector.md](../../systems/pgvector.md)
- [systems/pase.md](../../systems/pase.md)
- [systems/vbase.md](../../systems/vbase.md)
