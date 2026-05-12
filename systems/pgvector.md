---
title: pgvector（Postgres 扩展中 industry deployment 最广的 vector 检索）
type: system
sources: [pgvector-docs]
related: [pase.md, vbase.md, analyticdb-v.md, milvus.md, qdrant.md, weaviate.md, faiss.md, ../concepts/hnsw.md, ../concepts/product-quantization.md, ../concepts/relaxed-monotonicity.md, ../topics/attribute-filtering.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../topics/sparse-dense-hybrid-retrieval.md, ../topics/topk-vs-iterator-model.md]
created: 2026-05-12
updated: 2026-05-12
---

# pgvector

**TL;DR**: pgvector 是 Postgres 官方 extension (PostgreSQL License), 提供 vector storage + similarity search **作为 Postgres 原生类型与 index**——industry deployment 最广的 OSS vector 方案 (per industry surveys: Stack Overflow 2024 / DB-Engines 2024-2025 中 pgvector 是 Postgres-extension 类别中最常 mentioned). **对 wiki 内 vector DBs 的核心独特性**: (1) **不是独立 vector DBMS, 而是 Postgres extension** — 与 [PASE](./pase.md) (Ant Financial) 和 [VBASE](./vbase.md) (Microsoft Research) 形成 wiki 内 **"Postgres-extended vector retrieval" 三-source triangle**——pgvector 是其中 industry deployment 最广; (2) **架构哲学: vector 是 Postgres 一等公民类型, 不是 separate plane**——ACID + WAL + replication + JOINs + 完整 SQL 接口全继承, 无需额外 sync infrastructure; (3) **多 vector type 一等公民**: `vector` (单精度, 4 bytes/element, 16K dim 上限) / `halfvec` (半精度, 2 bytes, 16K dim) / `bit` (单 bit binary, Hamming/Jaccard) / `sparsevec` (sparse storage, 16K non-zero element)——通过类型选择直接控制 storage cost; (4) **6 distance operator**: `<->` L2 / `<#>` 负 inner product / `<=>` cosine / `<+>` L1 / `<~>` Hamming / `<%>` Jaccard——SQL 表达式直接使用; (5) **2 index type**: HNSW (graph, 主要性能) + IVFFlat (cluster, build 更快 + memory 更少); (6) **Iterative index scans (v0.8.0+)**——HNSW + filter 时自动扩展 ef_search 范围, **直接对应 wiki 内 [ACORN](../concepts/acorn.md) 哲学** (filter selectivity-aware ANN), 但 pgvector 是**Postgres extension 中首个 generalized iterative scan implementation**, strict_order vs relaxed_order 两 mode; (7) **Binary quantization expression-based indexing**——`binary_quantize(embedding)::bit(d)` 表达式直接作 index 列, 同 SQL 重排序 (cosine over original vector); (8) **Hybrid search via PostgreSQL FTS + vector**——pgvector 推荐 reciprocal rank fusion 或 cross-encoder rerank (对应 wiki [topics/sparse-dense-hybrid-retrieval.md](../topics/sparse-dense-hybrid-retrieval.md))。**Scale 边界**: 非分区表 32 TB / 分区表数千 × 32 TB / vector 16K dim 上限. [per sources/docs/pgvector/README.md]

## 与 wiki 内其他 system 的定位差异

| | Milvus | Qdrant | Weaviate | Vespa | Pinecone | Turbopuffer | **PASE** | **VBASE** | **pgvector** |
|---|---|---|---|---|---|---|---|---|---|
| 类型 | OSS Go DBMS | OSS Rust DBMS | OSS Go DBMS | OSS C++/Java | closed SaaS | closed SaaS | **PG extension** | **PG extension** | **PG extension** |
| 来源 | Zilliz 2019 | Qdrant 2021 | Weaviate 2019 | Yahoo! 2003 | Pinecone 2019 | Turbopuffer 2024 | Ant Financial 2020 | Microsoft Research 2023 | Community 2021+ |
| Storage layer | NVMe + S3 (cloud-native) | NVMe local | NVMe local | NVMe local | NVMe + slab | **object storage** | PG storage | PG storage | **PG storage (ACID + WAL)** |
| Vector index | HNSW/IVF*/DISKANN/CAGRA/SPARSE | HNSW only | HNSW + RQ8 + HFresh | HNSW + SPANN + Streaming | slab adaptive (黑盒) | SPFresh | IVFFlat + HNSW | per-index abstraction (IVFFlat/HNSW/SPANN 都支持) | **HNSW + IVFFlat** |
| Query model | gRPC TopK | gRPC TopK | gRPC TopK | YQL | API TopK | API multi_query | SQL 嵌入 | **SQL iterator (Relaxed Monotonicity)** | **SQL TopK + ORDER BY + LIMIT** |
| Industry deployment 数据 | Zilliz Cloud production | Stripe / AT&T 等 | 多客户 | Yahoo! 等 | Notion / 多 SaaS | Linear / Cursor 等 | Ant Financial 内部 | 学术 prototype + Microsoft 内部 | **DB-Engines surveys 中 Postgres extension 最多 mentioned vector option, Supabase / Neon / RDS / Aiven 等所有 managed Postgres 都支持** |

**核心论点**：pgvector **不与上述 6 个 vector DBMS 竞争**——pgvector 是**已 deploying Postgres 的 production workload 上 "加一个 extension" 就能用 vector retrieval 的路径**, 不需要 separate vector DBMS infrastructure. wiki 内之前用 PASE / VBASE 代表 Postgres-extension 路径; pgvector 是 industry-most-deployed 实例, 必填补.

## 架构图

```
                              ┌─────────PostgreSQL─────────┐
                              │                            │
              ┌─────────┐     │  ┌──────────────────────┐  │
              │ Client  │─────┼─▶│   SQL Parser         │  │
              └─────────┘     │  └──────────┬───────────┘  │
                              │             ▼              │
                              │  ┌──────────────────────┐  │
                              │  │   Query Planner      │  │
                              │  │   (chooses index)    │  │
                              │  └──────────┬───────────┘  │
                              │             ▼              │
                              │  ┌──────────────────────┐  │
                              │  │   Executor           │  │
                              │  │                      │  │
                              │  │  ┌──────────────┐    │  │
                              │  │  │ pgvector     │    │  │
                              │  │  │ HNSW / IVF   │    │  │
                              │  │  │ access method│    │  │
                              │  │  └──────────────┘    │  │
                              │  │  ┌──────────────┐    │  │
                              │  │  │ B-tree / GIN │    │  │
                              │  │  │ (other indexes)   │  │
                              │  │  └──────────────┘    │  │
                              │  └──────────────────────┘  │
                              │                            │
                              │  Storage: Postgres heap +  │
                              │  WAL + replication + ACID  │
                              └────────────────────────────┘
```

[per sources/docs/pgvector/README.md]

3 关键架构 insight:
- **Vector index access method 是 Postgres 一等公民**——`CREATE INDEX ... USING hnsw (...)` 与 `USING btree` / `USING gin` 同级别
- **Query planner 自动决定 index 选择**——不需要 application 端 hint, planner 看 cost model
- **所有 Postgres 特性继承**: ACID, WAL, point-in-time recovery, replication, JOINs, subqueries, CTEs, window functions 等

## 数据流 / 控制流

### Insert / Update

```sql
CREATE TABLE items (id bigserial PRIMARY KEY, embedding vector(768));
INSERT INTO items (embedding) VALUES ('[1.2, 3.4, ...]');
UPDATE items SET embedding = '[2.5, ...]' WHERE id = 1;
```

- 直接 SQL insert/update, **Postgres MVCC 自动管理 visibility**
- WAL 自动持久化
- 如果有 HNSW/IVFFlat index, index 在 insert 时增量更新

### Query

```sql
-- Cosine similarity, top-10
SELECT id, embedding <=> '[1.2, 3.4, ...]' AS distance
FROM items
ORDER BY distance
LIMIT 10;

-- With filter (uses iterative scan in v0.8.0+)
SELECT id, embedding <=> $1 AS distance
FROM items
WHERE category = 'electronics' AND price < 1000
ORDER BY distance
LIMIT 10;
```

**核心特点**:
- `ORDER BY <distance_op> LIMIT k` triggers vector index usage
- Without ORDER BY + LIMIT → sequential scan
- Filter + ORDER BY 由 planner 决定: B-tree filter first OR HNSW first + filter post

[per sources/docs/pgvector/README.md §Query Optimization]

### Iterative index scan (v0.8.0+)

```sql
-- Strict order mode
SET hnsw.iterative_scan = strict_order;
-- Relaxed order mode (better recall under heavy filter)
SET hnsw.iterative_scan = relaxed_order;
```

[per pgvector v0.8.0 release notes]

**问题**: HNSW + filter 时, 标准查询取 top-k from HNSW 然后 filter, 但 filter 后可能 < k 结果. 之前需要应用层 manual 调整 ef_search 重试.

**iterative scan 自动行为**:
- 取 ef_search results from HNSW, filter, 若 < limit 自动扩大 ef_search 重试
- **直接对应 wiki 内 [ACORN](../concepts/acorn.md) algorithm** 哲学 (filter selectivity-aware ANN)
- 是 wiki 内**第一个把 ACORN-style behavior generalized to extension level 的 implementation**——Qdrant/Weaviate/Vespa 各自实现 vendor-specific filter-aware ANN, pgvector 是 **Postgres extension 通用 implementation**

## 关键设计决策

### 1. Vector as Postgres first-class type, not separate plane

[per sources/docs/pgvector/README.md]

vs Milvus/Qdrant/Weaviate/etc. (separate DBMS):
- 不需要 sync infrastructure (CDC / 双写 / Kafka pipeline)
- ACID 跨 vector + scalar transactions
- WAL replication 自动 cover vector data
- JOINs / subqueries / window functions / CTE 直接 work
- **Production migration cost 极低**——已有 Postgres workload "add an extension" 即用

vs PASE/VBASE (also Postgres extensions):
- pgvector industry deployment 远超——所有 managed Postgres (Supabase / Neon / RDS / Aiven / Crunchy / CloudSQL) 都支持
- PASE 仅 Ant Financial 内部 production
- VBASE 仅学术 + Microsoft internal

### 2. Multi vector type schema-level decision

[per README §Vector Types]

| Type | Per-element size | Max dim | Distance metrics | 用途 |
|---|---|---|---|---|
| `vector` | 4 bytes (float32) | 16,000 | L2 / inner / cosine / L1 | dense embedding default |
| `halfvec` | 2 bytes (float16) | 16,000 | L2 / inner / cosine / L1 | **降 50% storage cost** |
| `bit` | 1 bit | 64,000 | Hamming / Jaccard | binary 向量 (binary quantization 输出) |
| `sparsevec` | per-element variable | 16,000 non-zero | L2 / inner / cosine | sparse vector (SPLADE/BM25 输出) |

→ **pgvector 是 wiki 内 OSS vector DBMS / extension 中 vector type 选择最 explicit 的**——type 直接 cost model. Vespa 通过 `tensor<>(...)` cell type 类似, 但 pgvector 4 type 都是 first-class data type.

### 3. Iterative index scan = generalized ACORN

[per pgvector v0.8.0 + concepts/acorn.md]

之前 wiki 内 ACORN 实例:
- Qdrant Filterable HNSW + ACORN fallback (v1.16.0)
- Weaviate ACORN + correlation optimization
- Vespa Acorn-1 mode (3-mode planner)

→ pgvector 在 v0.8.0 加 **iterative_scan generalized algorithm**——是**ACORN 哲学的 Postgres extension version**, automatic + non-vendor-locked. **Strict order** (维持距离排序保证) vs **relaxed order** (允许小幅 distance reorder 换更高 recall) 2 mode, 比 ACORN-1 paper 提的单一 mode 更 nuanced.

### 4. Binary quantization via expression-based indexing

[per README §Binary Quantization]

```sql
CREATE INDEX ON items USING hnsw ((binary_quantize(embedding)::bit(768)) bit_hamming_ops);

SELECT * FROM (
    SELECT * FROM items 
    ORDER BY binary_quantize(embedding)::bit(768) <~> binary_quantize('[1,-2,3]') 
    LIMIT 20
) ORDER BY embedding <=> '[1,-2,3]' LIMIT 5;
```

- **Index on expression** (Postgres native feature) — 不需 separate binary copy
- 二阶段查询: binary shortlist (HNSW + Hamming) → full vector rerank (cosine)
- 直接对应 wiki [topics/adaptive-retrieval-shortlist-rerank.md](../topics/adaptive-retrieval-shortlist-rerank.md) 的 AR pipeline
- **优势 vs 其他 vendor binary quantization**: 利用 Postgres expression index 复用 storage, 不像 Qdrant/Weaviate 需要 vendor-specific binary index 配置

### 5. Hybrid search via PostgreSQL FTS + vector

[per README §Hybrid Search]

```sql
-- PostgreSQL native FTS
SELECT id, ts_rank(search_vector, query) AS rank,
              embedding <=> $1 AS distance
FROM items, plainto_tsquery('english', 'search query') query
WHERE search_vector @@ query
ORDER BY (1 - distance) * 0.5 + rank * 0.5 DESC  -- α-blend fusion
LIMIT 10;
```

- 利用 Postgres native FTS (tsvector / tsquery) — 不需要 separate sparse vector store
- **Application 端 RRF or cross-encoder rerank** (pgvector README 推荐)
- vs Weaviate `hybrid(α)` first-class API: pgvector 需要 explicit SQL fusion expression, ergonomics 略差但 100% control
- vs Vespa rank-profile: pgvector SQL 比 Vespa expression 更 standard 但 less expressive

### 6. Filter strategies (4 mode)

[per README §Filtering]

| Strategy | 适用 selectivity | Postgres mechanism |
|---|---|---|
| Exact B-tree index on filter column | low cardinality, high selectivity (e.g., category) | standard PG |
| Approximate HNSW + increased `ef_search` | mid selectivity, automatic via iterative scan | pgvector v0.8.0+ |
| **Partial index** | specific category subset (e.g., active=true rows only) | `CREATE INDEX ... WHERE active=true` |
| Table partitioning | high cardinality (e.g., per-tenant) | PG declarative partitioning |

→ 这是 wiki 内 vector DBMS 中 filter strategies 最 SQL-native 的——4 种都是 Postgres standard feature, 不需要 vendor-specific filter API.

### 7. Scaling: vertical Postgres or distributed (Citus)

[per README §Scaling]

vs cloud-native vector DBMS (Milvus / Pinecone / Turbopuffer):
- **Vertical scale**: 单 Postgres instance (Supabase / RDS / Aiven 等托管) 32 TB per non-partitioned table
- **Horizontal scale options**:
  - PostgreSQL replication (read replicas)
  - Citus distributed cluster (sharding via citus_data extension)
  - Custom sharding (application-level)

**Limitation**: pgvector itself **不内置 distributed routing**——分布式依赖 Citus 或 application. vs Milvus/Pinecone native distributed.

## Scale 边界

[per sources/docs/pgvector/README.md §Limits]

| Metric | pgvector limit | Production observed |
|---|---|---|
| Per-table size (non-partitioned) | 32 TB | Supabase / Neon 等 managed PG 内见 多 TB cases |
| Per-table size (partitioned) | thousands × 32 TB | unknown |
| Vector dimensions (vector / halfvec) | 16,000 | typical 768-3072 |
| Vector dimensions (bit) | 64,000 | binary embedding output |
| Sparsevec non-zero elements | 16,000 | SPLADE / BM25 output |
| HNSW m (max connections per layer) | default 16 | tune 8-64 |
| HNSW ef_construction | default 64 | tune 64-200 |
| HNSW ef_search | default 40 | tune 40-200, iterative scan auto-adjusts |
| IVFFlat lists | recommend `rows/1000` for ≤1M, `sqrt(rows)` for >1M | typical 100-10000 |
| IVFFlat probes | default 1 | tune 1-50 |

### 瓶颈

- **Single-instance scale**——单 Postgres instance limit. Multi-region active-active 复杂 (Postgres replication primary-only).
- **Index build memory**——HNSW build 需 `maintenance_work_mem` 足够大 (大数据 build 是 production 痛点)
- **VACUUM impact on HNSW**——HNSW index 需要 vacuum 优化, vacuum 频率影响 production write throughput
- **IVFFlat needs data before indexing**——empty table 不能建 IVFFlat (HNSW 可以)
- **Cross-table JOINs on vector retrieval**——可行但 planner 选 plan 可能 suboptimal (复杂 hybrid query 需 manual hint)

## 生产案例

[per industry surveys + sources/docs/pgvector/README.md §Installation]

**Managed Postgres services 全部支持 pgvector**:
- Supabase (主推 pgvector + AI assistant ecosystem)
- Neon
- AWS RDS for PostgreSQL
- Google Cloud SQL
- Azure Database for PostgreSQL
- Aiven for PostgreSQL
- Crunchy Bridge
- TimescaleDB Cloud
- CockroachDB (pgvector-compatible)

**Industry deployment 数据**:
- DB-Engines 2024-2025 Postgres extension surveys: pgvector 是 Postgres-extension 中 vector retrieval 最 mentioned
- Supabase ecosystem: pgvector + supabase-js / AI features standard part of stack
- HackerNews / dev community: 主流 "starting your AI app" 推荐方案

**vs separate vector DBMS production reality**:
- "已有 Postgres workload + 加 vector retrieval" 是 pgvector 主要 deployment path
- "From scratch new AI app" 选 Milvus/Qdrant/Weaviate/Pinecone/Turbopuffer 等
- pgvector 占 "smaller scale (≤100M docs) but 已有 Postgres" market segment 主流

## Open Questions

- **pgvector vs PASE 哲学对比**: 都是 PG extension; PASE (Ant Financial 2020) 仅 internal production, pgvector industry deployment 远超. **PASE 路径在 pgvector 之外是否仍 active**? 论文层 PASE 2020, pgvector 2021+——pgvector 取代 PASE 是 likely 但具体 Ant Financial 内部状况未公开
- **pgvector vs VBASE iterator + Relaxed Monotonicity**: VBASE (Microsoft Research 2023 OSDI) 提出 RM + iterator model 在 PG 上实现; pgvector 当前是 TopK + ORDER BY + LIMIT (standard SQL). pgvector 是否 future 引入 VBASE iterator pattern? 未公开
- **pgvector + Citus distributed**: 实际 production case (e.g., Cloud Citus + pgvector at >100M docs) 不公开
- **VACUUM + HNSW production impact**: HNSW index 需要 vacuum; vacuum 与 write throughput 的 trade-off production case 不公开
- **pgvector vs Milvus/Pinecone/Weaviate scale ceiling**: industry assertion pgvector "≤100M docs sweet spot" 但具体上限 production case wiki 不公开
- **Iterative index scan vs ACORN-1**: pgvector v0.8.0 iterative_scan 与 Qdrant/Weaviate/Vespa 各自 ACORN implementation 实测 head-to-head 不存在
- **Binary quantization expression index VS Qdrant/Weaviate native BQ**: pgvector "expression-based + 二阶段 rerank" pattern 与 vendor-specific BQ 性能对比 不公开
- **Multi-vector 一等公民**: pgvector 当前不内置 multi-vector per row first-class——多 vector field 需 separate columns + 多 ANN index. vs Milvus/Vespa/Weaviate multi-vector 一等公民
- **Sparse vector (sparsevec) production usage**: SPLADE/BM25 输出可直接 store as sparsevec, 但 production case (e.g., pgvector + SPLADE end-to-end) 不公开
- **GPU acceleration**: pgvector 当前 CPU only; GPU build / GPU query 是否 future direction? 不公开
- **Foreign Data Wrapper (FDW) integration**: 跨 PG instance / 跨 vector DBMS 联邦查询是否 pgvector 通过 FDW 配合 Milvus/Pinecone 等 separate DBMS? Production case 不公开
- **MRL prefix-aware in pgvector**: pgvector 当前不感知 MRL embedding 的 nested prefix structure; query-time prefix truncation 在 SQL 层 work (substring on `vector(n)` to `vector(m)`) 但 native indexing prefix 选项 不存在
- **pgvector vs Lantern / pgvecto.rs 等其他 PG extension**: pgvector 当前主流但同代有 Lantern / pgvecto.rs (Rust 实现)——specific feature 差异 wiki 不覆盖

Cited by: 待 query 引用
