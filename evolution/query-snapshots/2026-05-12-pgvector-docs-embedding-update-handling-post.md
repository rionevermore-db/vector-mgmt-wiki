---
query-key: embedding-update-handling
query: "当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？"
date: 2026-05-12
phase: post
ingest-context: pgvector-docs
wiki-pages-total: 76
cited-pages: [systems/pgvector.md, systems/pase.md, systems/vbase.md]
cited-count: 3
---

# Post-snapshot (pgvector-docs): embedding-update-handling

## TL;DR (delta from formal-2021-splade-v2 post)

**pgvector 把 embedding upgrade 路径推到 Postgres standard SQL operations**——之前 wiki 内 embedding upgrade tool (Qdrant Aliases / Vespa Application Package / Turbopuffer namespace) 都是 vendor-specific. pgvector 通过 **Postgres native features** 实现等价 pattern: (a) `ALTER TABLE ADD COLUMN` 加新 vector column, (b) backfill via batch update + LLM API, (c) atomic SQL transaction swap, (d) `DROP COLUMN` 清理旧版. **关键 NEW**: pgvector + Postgres standard transactions 让 embedding upgrade **完全可在 single SQL transaction within ACID guarantees 内完成**——是 wiki 内**最 transaction-clean** 的 embedding migration 路径.

## Answer

### 与之前 ingest 的演进

| | formal-2021-splade-v2 post | **pgvector-docs post (NEW)** |
|---|---|---|
| Embedding migration tool 一等公民 | Qdrant Aliases / Vespa AppPkg / Turbopuffer namespace | **+ pgvector standard Postgres SQL operations (ACID transaction-bound)** |
| Multi-version vector coexistence | Vespa multi-tensor / Turbopuffer namespace-per-version / Milvus 多 vector field | **+ pgvector ALTER TABLE ADD COLUMN (native SQL)** |
| Transaction-bound migration | partial (Qdrant Alias atomic) | **NEW: pgvector full SQL transaction guaranteed atomic swap** |

### pgvector embedding migration via Postgres SQL（NEW pattern）

[per sources/docs/pgvector/README.md + Postgres standard]

**典型 upgrade workflow (voyage-3 → voyage-4)**:

```sql
-- Step 1: 加新 dim column 共存旧 dim (dimension may differ)
ALTER TABLE products ADD COLUMN embedding_v4 vector(1536);

-- Step 2: Build new HNSW index on v4 column
CREATE INDEX CONCURRENTLY products_emb_v4_idx 
  ON products USING hnsw (embedding_v4 vector_cosine_ops);

-- Step 3: Backfill in batches (application-side LLM API call)
-- 应用层: for batch in batches: 
--   UPDATE products SET embedding_v4 = $1 WHERE id = $2;

-- Step 4: Atomic application switch (in transaction)
BEGIN;
  -- Rename or swap defaults; queries 切换到 embedding_v4
COMMIT;

-- Step 5: Cleanup
DROP INDEX products_emb_v3_idx;
ALTER TABLE products DROP COLUMN embedding_v3;
```

→ **完全 ACID + WAL + replication-protected**, 比 Qdrant Aliases / Vespa AppPkg 更细颗粒——row-level transaction guarantee.

### 与 wiki 其他 vendor migration pattern 对比

| Vendor | Migration mechanism | Atomicity granularity |
|---|---|---|
| Qdrant Aliases | atomic collection alias swap | collection-level |
| Vespa Application Package | atomic deploy | cluster-level |
| Turbopuffer | namespace per version | namespace-level (S3 prefix) |
| Weaviate | named vectors | collection-level |
| Milvus | multi-vector field | collection-level |
| **pgvector** | **ALTER TABLE + transaction** | **row-level (ACID within transaction)** |

→ pgvector 是 wiki 内**migration atomicity 最细颗粒** (row-level) 的 vendor——通过 Postgres ACID transaction 自然实现.

### MRL prefix migration in pgvector (NEW use case)

[per concepts/matryoshka-embedding.md + pgvector vector type]

**Within-MRL-model multi-tier 在 pgvector** (轻量, 不需 schema change):
```sql
-- Query with prefix truncation (application-side)
SELECT id, 
       (substring(embedding::text, ...)::vector(256)) <=> $1 AS distance
FROM items
ORDER BY distance LIMIT 10;
```
- **Limitation**: 当前 pgvector 不感知 MRL nested structure; native indexing on prefix 需要 separate column (e.g., `embedding_256 vector(256)`) + 应用层维护

**Vespa matryoshka cell type 同等比较**:
- Vespa: native multi-tensor field + matryoshka cell type, query 内置 prefix selection
- pgvector: 通过 separate column 或 SQL substring (无 native prefix-aware index)

→ pgvector multi-tier MRL pattern 比 Vespa 重 (separate column + index), 但仍可 work via standard Postgres.

### 处理决策表（updated 2026-05-12 post pgvector-docs）

| 场景 | 推荐方案 |
|---|---|
| **已部署 Postgres + 加 embedding upgrade pipeline** | **pgvector ALTER TABLE + ACID transaction** (row-level granularity) |
| Per-tenant 独立 embedding model | Turbopuffer namespace-as-tenant |
| Atomic switch + cluster-wide deploy | Vespa application package |
| Collection-level simple swap | Qdrant Collection Aliases |
| 同 doc 多 model embedding 共存 | Vespa multi-tensor / Milvus 多 vector field / **pgvector ALTER TABLE** |
| Cross-model semantic preserve | 仍未有 production——SIGMOD 2026 |

### 已知盲区

- **pgvector CONCURRENTLY index build production cost**: 大规模 (>10M docs) `CREATE INDEX CONCURRENTLY` 实测时间不公开
- **Backfill batch size optimization**: 应用层 batch 大小 vs Postgres WAL pressure trade-off
- **MRL prefix native indexing in pgvector future**: 当前不存在
- **Cross-model semantic preserve**: 仍 talk SIGMOD 2026 唯一 candidate

## Cited Pages

- [systems/pgvector.md](../../systems/pgvector.md)
- [systems/pase.md](../../systems/pase.md)
- [systems/vbase.md](../../systems/vbase.md)
