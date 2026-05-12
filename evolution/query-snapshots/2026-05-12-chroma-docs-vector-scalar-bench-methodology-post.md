---
query-key: vector-scalar-bench-methodology
query: "对比多个 vector DB 在向量 + 标量过滤混合查询下的性能，如何公平地横向 benchmark？"
date: 2026-05-12
phase: post
ingest-context: chroma-docs
wiki-pages-total: 77
cited-pages: [systems/chroma.md, topics/sparse-dense-hybrid-retrieval.md, concepts/splade-sparse-retrieval.md]
cited-count: 3
---

# Post-snapshot (chroma-docs): vector-scalar-bench-methodology

## TL;DR (delta from pgvector-docs post)

**Chroma 引入 wiki 内**最 explicit hybrid + filter benchmark methodology vendor**——`Search API` + `Where` filter + `Rrf` ranks + `Knn` ranks 4 building block 全 first-class composable. **关键 NEW**: Chroma 在 fair benchmark 上提供 most ergonomic protocol 表达——单 `Search` 调用内可表达 sparse + dense + filter + RRF + limit + pagination 一体, 比 Weaviate `hybrid(α)` / Vespa rank-profile / Turbopuffer multi_query 都更 declarative.

## Answer

### 与之前 ingest 的演进

| | pgvector-docs post | **chroma-docs post (NEW)** |
|---|---|---|
| 8 vendor filter inventory | unchanged | **+ Chroma StringInvertedIndex + IntInverted + FloatInverted + BoolInverted + FtsIndexConfig** |
| Filter API ergonomic | Vespa rank-profile / Weaviate hybrid() / pgvector SQL / Turbopuffer multi_query | **+ Chroma Search API + Where + Rrf 最 declarative** |

### Chroma fair benchmark protocol（NEW most declarative）

[per sources/docs/chroma/llms-full.txt §Search API + §Filtering with Where]

```python
# Most declarative hybrid + filter + paginate fair benchmark
result = collection.search(
    rank=Rrf([
        Knn(query=user_query, limit=200, return_rank=True),  # dense
        Knn(query=user_query, key="sparse_emb", limit=200, return_rank=True)  # sparse SPLADE
    ], k=60, weights=[0.6, 0.4]),
    where=K("category") == "wallet" & (K("price") < 500) & K("").contains("italian"),
    limit=10,
    offset=0
)
```

**单 search call** 内表达:
- Dense + sparse RRF fusion
- Multi-condition filter (StringInverted + FloatInverted + FTS)
- Pagination
- All composable, all return rank-aware

vs 其他 vendor:
- Weaviate: `hybrid(query, alpha, where)` — 仅 α-blend, 简单但 less control
- Vespa: rank-profile + YQL — flexible 但 verbose
- pgvector: SQL expression — standard 但 verbose
- Turbopuffer: multi_query — 推到 client-side fusion

→ Chroma `Search API + Rrf + Where` **最 ergonomic for ad-hoc benchmark protocol**.

### Wiki 8 vendor filter 实现表（updated 2026-05-12 post chroma-docs）

| Vendor | Filter API | Hybrid API |
|---|---|---|
| Milvus | 5 strategies | application |
| Qdrant | Filterable HNSW + ACORN | application |
| Weaviate | ACORN + correlation | first-class `hybrid(α)` |
| Vespa | 3-mode YQL planner | rank-profile expression |
| Pinecone | filter + slab 黑盒 | Sparse-Dense Hybrid Index |
| Turbopuffer | native filtering | multi_query + app RRF |
| pgvector | SQL planner (4 mode) | SQL expression |
| **Chroma (NEW)** | **6 index type (FTS + String/Int/Float/Bool inverted) + Where expression** | **first-class `Rrf()` with k + weights** |
| DistributedANN | paper 不深入 | n/a |

### 已知盲区

- **Chroma 8 vendor fair benchmark head-to-head**: 不存在
- **Chroma Where filter strategies vs Qdrant/Weaviate/Vespa ACORN**: head-to-head zero
- **Chroma RRF k / weights production optimal**: 不公开

## Cited Pages

- [systems/chroma.md](../../systems/chroma.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
