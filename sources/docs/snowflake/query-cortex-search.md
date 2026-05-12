# Cortex Search Query API (WebFetch summary)

Source URL: https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/query-cortex-search-service
Fetched: 2026-05-12
Acquisition: WebFetch AI summary

## REST API endpoint

```
https://<account_url>/api/v2/databases/<db_name>/schemas/<schema_name>/cortex-search-services/<service_name>:query
```

Auth: PAT / JWT / OAuth via `Authorization: Bearer $PAT`.

## Request format

```json
{
  "query": "<search_query>",
  "columns": ["col1", "col2"],
  "filter": <filter>,
  "limit": <limit>
}
```

## Hybrid weighting (multi-index search)

```json
"scoring_config": {
  "weights": {
    "texts": 3,
    "vectors": 2,
    "reranker": 1
  }
}
```

> "weights are applied relative to each other" — `{3,2,1}` == `{30,20,10}`.

## Per-index boost

```json
"functions": {
  "vector_boosts": [{"weight": 2, "column": "vector_col_name"}],
  "text_boosts": [{"weight": 1, "column": "text_col_name"}]
}
```

## Filter operators

| Op | Use |
|---|---|
| `@eq` | text/numeric equality |
| `@contains` | array membership |
| `@gte` / `@lte` | numeric/timestamp range |
| `@primarykey` | PK match |

Composable via `@and` / `@or` / `@not`.

```json
{
  "@and": [
    {"@gte": {"numeric_col": 10.5}},
    {"@lte": {"numeric_col": 12.5}}
  ]
}
```

## Limits

- `limit`: max **1000**, default **10**
- Response: REST/Python 10 MB, SQL `SEARCH_PREVIEW` 300 KB
- `reranker: "none"` 禁用 rerank
- `multi_index_query` 限制 search index 子集——降 cost
- `numeric_boosts` / `time_decays` 列加权 / 时间衰减
