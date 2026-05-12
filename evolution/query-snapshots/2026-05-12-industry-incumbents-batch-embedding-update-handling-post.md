---
query-key: embedding-update-handling
date: 2026-05-12
phase: post
ingest-context: industry-incumbents-batch
wiki-pages-total: 96
cited-pages: [systems/databricks-vector-search.md, systems/mongodb-atlas-vector-search.md, systems/snowflake-cortex-search.md]
cited-count: 3
---

# Post-snapshot (industry-incumbents-batch): embedding-update-handling

## TL;DR

**重大 NEW** (talk SIGMOD 2026 主题直接关键发现):

**Vendor-default embedding model 列表**——cross-model migration 在 production 的 vendor lock-in:

| Vendor | Default embedding | 升级 cost |
|---|---|---|
| **Snowflake Cortex Search** | `snowflake-arctic-embed-m-v1.5` (自研) | Vendor controls model; auto-update if Snowflake releases new version |
| **MongoDB Atlas** | **Voyage AI** (2024 acquisition) | Voyage 升级 → MongoDB auto-pipeline 自动 re-encode (隐式) |
| **Databricks Vector Search** | User-specified Foundation Model API | User-controlled, 升级时 manual re-encode + reindex |
| **Elasticsearch** | User-provided (ELSER 是 Elastic 自研 sparse) | User-controlled |
| **OpenSearch** | User-provided (Neural Sparse 是 OS 自研) | User-controlled |
| **Redis Stack** | User-provided | User-controlled |

**关键 NEW**: production embedding model 升级 cost 分 **3 类 vendor-control mode**:
1. **Fully vendor-controlled** (Snowflake): user 无选择, vendor 升级时 corpus 自动 re-encode (in theory). 但 vendor 更换 model 时**用户应用语义可能漂移**, 是 talk SIGMOD 2026 论文 cross-model migration 的隐藏 production scenario.
2. **Vendor-default + auto-pipeline** (MongoDB Atlas + Voyage AI): vendor 收购的 embedding provider, "Automated Embedding service keeps embeddings in sync as data changes"——但跨 Voyage model 版本升级 (v1 → v2 → v3) 的 corpus migration 行为 doc 未明示, 推测 user-controlled.
3. **User-controlled** (Databricks / ES / OpenSearch / Redis): 完全用户责任. 是 [BGE-M3](../concepts/bge-m3.md) / [NV-Embed](../concepts/modern-embedding-paradigms.md) 等 OSS embedding 用户的默认路径.

**talk SIGMOD 2026 implication**: cross-model migration 工具 / methodology **对 vendor-controlled mode (mode 1) 的用户无用 (vendor 已隐藏 problem), 但对 user-controlled mode (mode 3) 用户极关键**——后者占 production 多数.

## Cited Pages

- [systems/databricks-vector-search.md](../../systems/databricks-vector-search.md)
- [systems/mongodb-atlas-vector-search.md](../../systems/mongodb-atlas-vector-search.md)
- [systems/snowflake-cortex-search.md](../../systems/snowflake-cortex-search.md)
