---
query-key: embedding-update-handling
query: "当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？"
date: 2026-05-12
phase: post
ingest-context: lancedb-docs
wiki-pages-total: 78
cited-pages: [systems/lancedb.md, systems/chroma.md, systems/pgvector.md]
cited-count: 3
---

# Post-snapshot (lancedb-docs): embedding-update-handling

## TL;DR (delta from chroma-docs post)

**LanceDB 提供 wiki 内独特 zero-copy schema evolution embedding migration pattern**——`table.add_columns({"new_embedding_v2": ...})` 不复制 existing data 即添加新 vector column. 与 Chroma fork (collection-level CoW) 不同——LanceDB 是 **column-level zero-copy** (column-level schema evolution). **关键 NEW**: zero-copy add column 是 wiki 内**最 cost-efficient embedding migration primitive**——不需要 fork collection (避免 fork tree limit), 不需要重建 schema (避免 reindex cost).

## Answer

### 与之前 ingest 的演进

| | chroma-docs post | **lancedb-docs post (NEW)** |
|---|---|---|
| Migration primitive 全景 | row-level (pgvector ACID) / collection-level CoW (Chroma) | **+ column-level zero-copy (LanceDB Lance format)** |
| Migration cost-efficient 最佳 | Chroma CoW fork (CoW storage) | **LanceDB zero-copy add column (no storage duplication 直接)** |

### LanceDB column-level zero-copy embedding migration

[per sources/docs/lancedb/]

```python
# Add new embedding column without copying existing data
table.add_columns({"voyage_4_embedding": lambda row: voyage_4_api(row.text)})

# Backfill via UDF or batch update — only new column allocated storage
# Existing voyage_3_embedding column unchanged (zero-copy isolation)

# Query with new column
result = table.search([0.1, 0.2, ...], vector_column_name="voyage_4_embedding") \
    .limit(10).to_pandas()

# When validated, optionally drop old column
table.drop_columns(["voyage_3_embedding"])
```

→ **column-level migration primitive**——比 Chroma fork (collection-level) 更细颗粒.

### Migration primitive 全景表（updated 2026-05-12 post lancedb-docs）

| Vendor | Migration primitive | Granularity | Storage cost |
|---|---|---|---|
| Qdrant Aliases | atomic alias swap | collection-level | new collection full storage |
| Vespa Application Package | atomic deploy | cluster-level | new schema full reindex |
| Turbopuffer copy_from_namespace | full copy | namespace-level | full copy (50% discount) |
| Weaviate named vectors | per-object switch | object-level | shared storage |
| pgvector ALTER TABLE | new column | row-level (ACID) | new column full storage |
| **Chroma fork** | CoW fork | collection-level | **CoW incremental only** |
| **LanceDB zero-copy add column** | column-level schema evolution | **column-level** | **zero-copy for existing, new column 自然 storage** |

→ **LanceDB column-level zero-copy + Chroma CoW fork 是 wiki 内 2 个最 cost-efficient migration primitive**, 各自 axis 不同 (column-level vs collection-level).

### 已知盲区

- **LanceDB zero-copy add column production case at scale**: 100M+ docs 实测 不公开
- **LanceDB add column + index rebuild**: 新 column 需重 index, 实际 cost 不公开
- **LanceDB vs Chroma fork head-to-head migration cost**: 不公开

## Cited Pages

- [systems/lancedb.md](../../systems/lancedb.md)
- [systems/chroma.md](../../systems/chroma.md)
- [systems/pgvector.md](../../systems/pgvector.md)
