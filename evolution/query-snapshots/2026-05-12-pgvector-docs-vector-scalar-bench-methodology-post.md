---
query-key: vector-scalar-bench-methodology
query: "对比多个 vector DB 在向量 + 标量过滤混合查询下的性能，如何公平地横向 benchmark？"
date: 2026-05-12
phase: post
ingest-context: pgvector-docs
wiki-pages-total: 76
cited-pages: [systems/pgvector.md, topics/attribute-filtering.md, systems/pase.md, systems/vbase.md]
cited-count: 4
---

# Post-snapshot (pgvector-docs): vector-scalar-bench-methodology

## TL;DR (delta from formal-2021-splade-v2 post)

**pgvector 在 vector-scalar benchmark methodology 上提供 wiki 内最全面 SQL-native filter 策略 inventory**: 4 种 filter 策略 (B-tree on filter column / iterative HNSW + ef_search / partial index / table partitioning), **全部 Postgres standard feature**——不需要 vendor-specific filter API. **关键 NEW**: 公平 benchmark 必须 explicit 区分 "vendor-specific filter integration" (Qdrant Filterable HNSW / Weaviate ACORN / Vespa Acorn-1) vs "SQL-native filter via Postgres planner" (pgvector). pgvector iterative_scan v0.8.0+ 是 wiki 内 ACORN 哲学的**Postgres extension generalization**.

## Answer

### Wiki 当前 vendor filter 实现表（updated 2026-05-12 post pgvector-docs）

| Vendor | Filter 实现 | Filter API 类型 |
|---|---|---|
| Milvus | 5 strategies (Pre-A/Pre-B/Per-segment-A/Per-segment-B/Per-segment-partition E) | vendor-specific |
| Qdrant | Filterable HNSW + ACORN fallback (v1.16.0) + Tenant/Principal Index | vendor-specific |
| Weaviate | ACORN + correlation optimization | vendor-specific |
| Vespa | 3-mode (pre/post/Acorn-1) by YQL planner | vendor-specific (YQL) |
| Pinecone | filter + slab adaptive (不公开) | vendor-specific (黑盒) |
| Turbopuffer | "native filtering" with SPFresh clustering-hierarchy aware | vendor-specific |
| **pgvector (NEW)** | **4 策略**: B-tree exact / iterative HNSW + ef_search / partial index / table partitioning | **SQL standard (Postgres planner)** |
| DistributedANN | paper 不深入 | vendor-specific (Bing internal) |

### pgvector filter 策略 4 mode（NEW SQL-native pattern）

[per sources/docs/pgvector/README.md §Filtering]

**1. B-tree exact index on filter column**:
```sql
CREATE INDEX ON items (category);  -- standard PG B-tree
SELECT * FROM items WHERE category = 'electronics' 
ORDER BY embedding <=> $1 LIMIT 10;
```
- 低 cardinality, 高 selectivity (e.g., category)
- Planner 选 B-tree first → HNSW post (or full scan if HNSW skip)

**2. Iterative HNSW + increased `ef_search`**:
```sql
SET hnsw.ef_search = 200;  -- broader matching
SET hnsw.iterative_scan = relaxed_order;  -- v0.8.0+ auto-adjust
SELECT * FROM items WHERE category = 'electronics'
ORDER BY embedding <=> $1 LIMIT 10;
```
- 中 selectivity, HNSW 先取 broader → filter post

**3. Partial index** (Postgres standard, not vector-specific):
```sql
CREATE INDEX ON items USING hnsw (embedding vector_cosine_ops)
WHERE active = true;  -- 只 index active=true 的 row
```
- 特定 category subset, **HNSW build 只对该 subset**

**4. Table partitioning**:
```sql
CREATE TABLE items (
  id bigserial,
  tenant_id int,
  embedding vector(768)
) PARTITION BY HASH (tenant_id);
-- Per-partition HNSW index
```
- 高 cardinality (per-tenant), declarative partitioning

→ **pgvector filter strategies 全部 Postgres standard feature**——不需要 vendor-specific filter API. SQL planner 自动决定 plan.

### Iterative index scan = ACORN philosophy generalized

[per pgvector v0.8.0 + concepts/acorn.md]

**之前 wiki ACORN production**:
- Qdrant v1.16.0 Filterable HNSW + ACORN fallback
- Weaviate ACORN + correlation optimization
- Vespa Acorn-1 mode (3-mode planner)

**pgvector v0.8.0+ iterative_scan**:
- 自动扩展 ef_search 当 filter 后 < limit results
- strict_order / relaxed_order 2 mode (比 ACORN-1 paper 单 mode 更 nuanced)
- **第 4 个独立 ACORN-style implementation** (Qdrant + Weaviate + Vespa + pgvector)

→ pgvector 让 ACORN 哲学**通过 Postgres extension 进入 universal Postgres ecosystem**.

### Multi-vendor fair benchmark methodology (updated 2026-05-12 post pgvector-docs)

**Tier 1: Capability matrix (now 8 vendor)**:
- Milvus / Qdrant / Weaviate / Vespa / Pinecone / Turbopuffer / DistributedANN / **pgvector**

**Tier 2: Benchmark axis**:
- Filter integration type (vendor-specific vs SQL-standard)
- Selectivity sweep
- Filter complexity (Eq / range / glob-regex / AND-OR)
- iterative scan mode (where applicable)

**Tier 3: Fair benchmark pitfall**:
- 用 vendor-specific API 测试 vs 用 SQL standard 测试 — 不可直接比较
- pgvector iterative_scan 必须区分 strict_order vs relaxed_order
- B-tree first vs HNSW first plan 选择影响 latency

### 已知盲区

- **pgvector 4 mode filter 实测 head-to-head**: 不公开
- **pgvector iterative_scan vs Qdrant/Weaviate/Vespa ACORN**: 不公开
- **pgvector + partial index 大规模 production case**: 不公开
- **8 vendor filter fair benchmark**: 不存在

## Cited Pages

- [systems/pgvector.md](../../systems/pgvector.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
- [systems/pase.md](../../systems/pase.md)
- [systems/vbase.md](../../systems/vbase.md)
