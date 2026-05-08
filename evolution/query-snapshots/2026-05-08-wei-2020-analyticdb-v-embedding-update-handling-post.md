---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-08
phase: post
ingest-context: wei-2020-analyticdb-v
wiki-pages-total: 42
cited-pages: [systems/analyticdb-v.md, systems/milvus.md]
cited-count: 2
---

# Post-snapshot (wei-2020-analyticdb-v): embedding-update-handling

## TL;DR (delta from guo-2022-manu post)

**ADBV 没有给 embedding model 升级 algorithmic 答案**——与所有已 ingest source 一致。但 ADBV 的 SQL 接口 + UDF 设计**让 re-embedding pipeline 工程上更简单**：`FEATURE_EXTRACT(URL)` 函数可换实现，`INSERT ... FEATURE_EXTRACT(...)` 自动用新 model。这是工程便利但不是 algorithmic 解。**仍未解决**：dimension / metric 切换时索引必须重建。

## Answer

### ADBV 的 embedding 工程支持（NEW）

[per systems/analyticdb-v.md "数据模型 / SQL 方言"]

ADBV 提供：
- `FEATURE_EXTRACT(URL)` UDF——内置或 user 自定义 embedding 函数
- `INSERT INTO ... VALUES (..., FEATURE_EXTRACT('url'))`——自动调函数
- 多 collection / 多 table 隔离 embedding space
- SQL 接口 + JDBC/ODBC 客户端

→ 工程上 cutover 流程：
```sql
-- 1. 创建新 table 用新 embedding model
CREATE TABLE clothes_v2 (
    ...,
    feature float[768],  -- 新维度
    ExtractFrom = 'NEW_MODEL_FEATURE_EXTRACT(image_url)'
);

-- 2. 从旧 table 重新 embed
INSERT INTO clothes_v2 SELECT id, color, ..., NEW_MODEL_FEATURE_EXTRACT(image_url)
FROM clothes_v1;

-- 3. 切换 query 应用层 → clothes_v2
-- 4. DROP TABLE clothes_v1
```

→ 比 [Milvus](../../systems/milvus.md) v2.6.x function field 类似——但 ADBV 因 SQL 接口对 OLAP 工程师更熟悉。

### ADBV 仍不解决的问题（NEW）

[per systems/analyticdb-v.md "Open Questions"]

1. **维度变化必须新 table**：feature float[256] vs float[768] 不能改 schema in-place
2. **metric 变化也必须新 table**：DistanceMeasure 一次定义不能改
3. **跨 model embedding mapping function**：仍 zero coverage——所有 wiki source 都不解
4. **Re-embedding 期间 storage 翻倍**：双 table 共存 → cost 翻倍

→ ADBV 提供工具链，但**模型升级 algorithmic 答案与其他 source 一致——未解**。

### 与之前 ingest 的演进

| | guo-2022-manu post | **wei-2020-analyticdb-v post (NEW)** |
|---|---|---|
| Vector update 解 | + delta τ + time travel | **+ ADBV lambda + UDF re-embed** |
| Embedding upgrade algorithmic 解 | future work（无） | **仍 zero**（工具链更齐但 algorithmic 仍未解） |
| SQL 接口 cutover | n/a（Milvus 不是 SQL） | **ADBV 首次 SQL native** |

### Open / 未覆盖

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo
- **跨 model embedding mapping function**：仍 zero
- **共享 embedding space contrastive fine-tune**：仍 zero
- **ADBV 实际 production embedding 升级实践**：Smart City production 升级历史 docs 不公开

## Cited Pages

- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
- [systems/milvus.md](../../systems/milvus.md)
