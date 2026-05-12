---
vendor: Databricks (Vector Search)
source-url: https://docs.databricks.com/en/generative-ai/vector-search.html
fetched-at: 2026-05-12
acquisition-method: WebFetch AI summary
license: 商业产品文档, 公开可阅读
notes: |
  Databricks docs at docs.databricks.com 类似 Snowflake, 无 llms.txt 也无 GitHub-hosted markdown source.
  Path C 退化为 WebFetch single-page summary——仅抓 Vector Search 主 page 作 citation anchor.
---

# Databricks Vector Search docs snapshot

- `vector-search-overview.md` — landing page (docs.databricks.com WebFetch summary)
- `vector-search-detailed.md` — **完整架构** (Azure docs raw markdown source-of-truth at MicrosoftDocs/databricks-pr GitHub): HNSW + L2 + RRF 公式 + 4 类 index 选项 + 完整 limit table + Storage-optimized 约束 + 加密 + Unity Catalog
- `vector-search-creation.md` — Delta Sync vs Direct Access creation + endpoint types + target_qps
