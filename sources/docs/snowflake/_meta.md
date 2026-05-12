---
vendor: Snowflake (Cortex Search)
source-url: https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/
fetched-at: 2026-05-12
acquisition-method: WebFetch AI summary (no llms.txt available; docs JavaScript-rendered, no GitHub repo for raw markdown)
license: 商业产品文档, 公开可阅读
notes: |
  Snowflake docs at docs.snowflake.com 是 JavaScript-rendered React app, 无 llms.txt.
  Path C 退化为 WebFetch single-page summary——仅抓 Cortex Search Overview 页作 citation anchor.
  完整 docs 含多 page (REST API, SQL syntax, tutorial, advanced filtering, etc), 未来需详细 ingest 时可扩展.
---

# Snowflake Cortex Search docs snapshot

- `cortex-search-overview.md` — landing page, 架构 + 关键组件 + scale limit
- `create-cortex-search-sql.md` — **CREATE CORTEX SEARCH SERVICE SQL** full syntax (single-index + multi-index pattern), embedding model 选项, **EMBEDDING_MODEL 不可 ALTER 约束**, refresh modes
- `query-cortex-search.md` — REST API query format, **hybrid weight `scoring_config.weights` 公开** (texts/vectors/reranker relative), filter operators (@eq/@gte/@lte/@and/@or/@not), limits
