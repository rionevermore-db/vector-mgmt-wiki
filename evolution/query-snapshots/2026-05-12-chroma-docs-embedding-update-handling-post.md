---
query-key: embedding-update-handling
query: "当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？"
date: 2026-05-12
phase: post
ingest-context: chroma-docs
wiki-pages-total: 77
cited-pages: [systems/chroma.md, concepts/matryoshka-embedding.md]
cited-count: 2
---

# Post-snapshot (chroma-docs): embedding-update-handling

## TL;DR (delta from pgvector-docs post)

**Chroma 引入 wiki 内**首个 CoW collection forking native pattern**——`source.fork(new_name="...")` 0-cost instant fork + 仅 incremental storage. 适合 prompt engineering / embedding model iteration / A/B test. **关键 NEW**: Chroma fork 是 vector DB 内**第一次提供 CoW 版本化 primitive**, 把 embedding upgrade 从 "重 embed + 重建 index" 变成 "fork collection + 重 embed (CoW incremental)" — 实质降低 migration risk 与 storage cost.

## Answer

### 与之前 ingest 的演进

| | pgvector-docs post | **chroma-docs post (NEW)** |
|---|---|---|
| Migration atomicity | row-level (pgvector ACID transaction) | **collection-level CoW fork (Chroma)** |
| CoW fork native primitive | 不存在 | **NEW: Chroma collection fork (Cloud only, $0.03/fork)** |
| Embedding migration cost model | 全量 backfill | **CoW: 仅 incremental storage billed** |

### Chroma fork-based embedding migration (NEW pattern)

[per sources/docs/chroma/llms-full.txt §Collection Forking]

```python
# Pattern: fork collection for embedding upgrade
source = client.get_collection("production-emb-voyage-3")
fork = source.fork("production-emb-voyage-4")

# Backfill new embeddings (CoW: 仅 incremental storage)
# Application 端: re-embed via voyage-4 + update fork
fork.upsert(ids=batch_ids, embeddings=voyage_4_embeddings)

# A/B test new model vs production
# Atomically swap default reference after validation
```

**vs 其他 vendor**:
| Vendor | Migration tool | Atomicity | Storage cost |
|---|---|---|---|
| Qdrant Collection Aliases | atomic alias swap | collection-level | new collection 全 copy |
| Vespa Application Package | atomic deploy | cluster-level | new schema 全 reindex |
| Turbopuffer copy_from_namespace | full copy (50% discount) | namespace-level | new namespace 全 copy |
| Weaviate named vectors | per-object switch | object-level | shared storage same collection |
| pgvector ALTER TABLE | new column | row-level | new column full storage |
| **Chroma fork** | **CoW fork** | **collection-level** | **incremental only (CoW)** |

→ **Chroma fork is wiki 内 cost-most-efficient embedding migration pattern**——零 backfill 启动 + CoW incremental billing.

### Limitations

- Chroma fork is **Chroma Cloud only** (OSS Core single-node 不支持 CoW storage engine)
- Fork tree limit: **256 fork edges per tree** — 不能无限 fork
- Cross-model semantic preserve 仍未关闭 (CoW 只解决 storage cost, 不解决 model semantic 差异)

### 已知盲区

- **CoW fork 256 edge limit production case**: 是否实际客户撞 limit? 不公开
- **Chroma fork performance at scale (100M+ docs source)**: 不公开
- **Cross-vendor CoW migration tool**: 仅 Chroma; 其他 vendor 不引入

## Cited Pages

- [systems/chroma.md](../../systems/chroma.md)
- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
