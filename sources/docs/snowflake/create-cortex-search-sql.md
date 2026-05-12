# CREATE CORTEX SEARCH SERVICE — SQL Syntax (WebFetch summary)

Source URL: https://docs.snowflake.com/en/sql-reference/sql/create-cortex-search
Fetched: 2026-05-12
Acquisition: WebFetch AI summary

## Core Syntax — Single-Index Pattern

```sql
CREATE [ OR REPLACE ] CORTEX SEARCH SERVICE [ IF NOT EXISTS ] <name>
  ON <search_column>
  [ PRIMARY KEY ( <col_name> [, ... ] ) ]
  ATTRIBUTES <col_name> [ , ... ]
  WAREHOUSE = <warehouse_name>
  TARGET_LAG = '<num> { seconds | minutes | hours | days }'
  [ EMBEDDING_MODEL = <embedding_model_name> ]
  [ REFRESH_MODE = { FULL | INCREMENTAL } ]
  [ INITIALIZE = { ON_CREATE | ON_SCHEDULE } ]
  [ FULL_INDEX_BUILD_INTERVAL_DAYS = <num> ]
  [ REQUEST_LOGGING = { TRUE | FALSE } ]
  [ AUTO_SUSPEND = <num_seconds> ]
  [ COMMENT = '<comment>' ]
AS <query>;
```

## Multi-Index Pattern (text + vector 多索引)

```sql
CREATE [ OR REPLACE ] CORTEX SEARCH SERVICE <name>
  TEXT INDEXES <text_column_name> [ , ... ]
  VECTOR INDEXES <column_specification> [ , ... ]
  [ PRIMARY KEY ( <col_name> [, ... ] ) ]
  ATTRIBUTES <col_name> [ , ... ]
  WAREHOUSE = <warehouse_name>
  TARGET_LAG = '<num> ...'
  [ ... ]
AS <query>;
```

VECTOR INDEXES 3 种 embedding strategy:
- Managed: `text_column (model='model_name')`
- User-provided: `vector_column`
- User vec + managed query embed: `vector_column(query_model='model_name')`

## 关键参数 + 默认值

| Parameter | Default | Behavior |
|-----------|---------|----------|
| `EMBEDDING_MODEL` | `snowflake-arctic-embed-m-v1.5` | **不可 ALTER**, 改需 recreate |
| `REFRESH_MODE` | `INCREMENTAL` | Full vs incremental refresh |
| `INITIALIZE` | `ON_CREATE` | 同步 or 计划性 initial load |
| `FULL_INDEX_BUILD_INTERVAL_DAYS` | `0` | Full rebuild 软目标 (primary key services only) |
| `REQUEST_LOGGING` | `FALSE` | request tracking |
| `AUTO_SUSPEND` | `NULL` | min 1800s (30 min) |

## 关键约束

- `OR REPLACE` and `IF NOT EXISTS` 互斥
- Multi-index services: 至少 1 column 在 VECTOR INDEXES
- Base tables 必须启用 change tracking 以支持 incremental refresh
- **EMBEDDING_MODEL 不能 ALTER**, 改需 recreate service——cross-model migration 必然全量 reindex
