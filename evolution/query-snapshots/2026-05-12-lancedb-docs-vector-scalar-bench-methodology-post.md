---
query-key: vector-scalar-bench-methodology
query: "对比多个 vector DB 在向量 + 标量过滤混合查询下的性能，如何公平地横向 benchmark？"
date: 2026-05-12
phase: post
ingest-context: lancedb-docs
wiki-pages-total: 78
cited-pages: [systems/lancedb.md, topics/sparse-dense-hybrid-retrieval.md]
cited-count: 2
---

# Post-snapshot (lancedb-docs): vector-scalar-bench-methodology

## TL;DR (delta from chroma-docs post)

**LanceDB SQL-native standalone (无需 Postgres) 提供 unique benchmark dimension**——`table.search(...).where("...")` 用 SQL filter + vector search 一体, 类似 pgvector 但不需要 Postgres prerequisite. 与 Chroma `Search API + Rrf` 不同, LanceDB 走 SQL standard 路径——benchmark protocol 可直接 reuse SQL knowledge. **关键 NEW**: ML ecosystem integration 让 LanceDB benchmark 可直接通过 DuckDB / Pandas / Polars 跨工具验证.

## Answer

### 与之前 ingest 的演进

| | chroma-docs post | **lancedb-docs post (NEW)** |
|---|---|---|
| Filter API ergonomic | Chroma Rrf + Where 最 declarative | **+ LanceDB SQL standalone + ML ecosystem cross-tool query** |
| ML ecosystem benchmark cross-validation | application-level | **LanceDB DuckDB / Pandas / Polars 直接 read Lance format** |

### LanceDB SQL-native standalone filter

```python
result = table.search([0.12, 0.22, ...]) \
    .where("category = 'wallet' AND price < 500 AND brand IN ('A', 'B')") \
    .limit(10) \
    .to_pandas()
```

vs 其他 vendor:
- pgvector: SQL native 但需 Postgres
- LanceDB: **SQL native standalone embedded library** (no DBMS prerequisite)
- Vespa: YQL specific dialect
- Chroma: `Where(K("price") < 500 & ...)` expression
- Weaviate / Qdrant / Milvus: API filter

→ LanceDB SQL standalone 是 wiki 内**第 2 个 SQL filter standard 的 vendor** (vs pgvector with Postgres prerequisite).

### ML ecosystem cross-tool benchmark (NEW)

[per sources/docs/lancedb/ + Lance format]

LanceDB benchmark 独特优势:
```python
# Same Lance file readable by multiple tools
duckdb.sql("SELECT * FROM 'data.lance' WHERE price < 500")
pl.read_lance("data.lance").filter(pl.col("price") < 500)
pd.read_lance("data.lance").query("price < 500")
```

→ benchmark protocol can be **independently verified by multiple ML tools without LanceDB-specific SDK**——cross-tool fair benchmark 是 unique LanceDB capability.

### 已知盲区

- **LanceDB SQL planner vs Vespa YQL vs Chroma Where**: head-to-head 不公开
- **Cross-tool benchmark using Lance format**: production 实际 case 不公开
- **LanceDB filter performance at giga-scale**: 不公开

## Cited Pages

- [systems/lancedb.md](../../systems/lancedb.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
