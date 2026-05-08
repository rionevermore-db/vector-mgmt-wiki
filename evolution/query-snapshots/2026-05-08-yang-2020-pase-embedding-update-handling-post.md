---
query-key: embedding-update-handling
query: "Embedding model 升级（BERT→SBERT、OpenAI v2→v3）时已有索引和原始文档怎么处理？必须全量重建？有跨模型兼容/复用方案？"
date: 2026-05-08
phase: post
ingest-context: yang-2020-pase
wiki-pages-total: 44
cited-pages: [systems/pase.md, systems/milvus.md, systems/analyticdb-v.md]
cited-count: 3
---

# Post-snapshot (yang-2020-pase): embedding-update-handling

## TL;DR (delta from wei-2020-analyticdb-v post)

**PASE 不解决 embedding model 升级，但 PG ACID transaction + WAL 让 cutover 工程更安全**。在 OLTP RDBMS 内做"双 column / 双 table cutover"是 SQL 工程师熟悉的模式——PG schema migration 工具齐全。PASE 把 vector 当 first-class column 让 cutover 与传统 SQL schema migration 一致。但**仍是工程便利不是 algorithmic 解**——所有 wiki source 的 embedding lifecycle algorithmic 答案仍 zero。

## Answer

### PASE 在 embedding 升级场景的工程支持（NEW）

[per systems/pase.md "PG 全套 OLTP 能力 free 复用"]

PASE 因复用 PG 全套 OLTP 能力，embedding 升级 cutover 可走 PG 标准工作流：

```sql
-- 方式 1：双 column（同 table）
ALTER TABLE clothes ADD COLUMN feature_v2 pase;
UPDATE clothes SET feature_v2 = NEW_MODEL_FEATURE_EXTRACT(image_url);
CREATE INDEX hnsw_v2_idx ON clothes USING pase_hnsw (feature_v2);
-- 切流量到 feature_v2 column
-- ALTER TABLE clothes DROP COLUMN feature_v1;

-- 方式 2：双 table（更安全）
CREATE TABLE clothes_v2 LIKE clothes;
ALTER TABLE clothes_v2 ALTER COLUMN feature TYPE pase_768d;  -- 假设新维度
INSERT INTO clothes_v2 SELECT id, color, ..., NEW_MODEL_FEATURE_EXTRACT(image_url)
FROM clothes;
CREATE INDEX hnsw_idx ON clothes_v2 USING pase_hnsw (feature);
-- 切流量
-- DROP TABLE clothes;
```

**PG 工程优势**：
- **ACID transaction**：UPDATE / INSERT 原子；崩溃不留半成品状态
- **WAL replication**：升级期间复制到 standby PG 提供 read traffic 0 downtime
- **Schema migration 工具**：pg_dump / pg_restore / pgsql 生态成熟
- **Backup**：cutover 前 pg_dump 整 table 作 snapshot

→ 对 SQL 工程师而言，embedding 升级**工作流上 = 加列 + UPDATE + 切流量**——熟悉的模式。

### 不解决的核心问题（NEW）

[per systems/pase.md "Open Questions"]

PASE 仍不解决：
1. **Embedding 维度变化必须新 column / 新 table**：feature pase_512d ≠ pase_768d
2. **跨模型 embedding mapping function**：仍 zero coverage
3. **Re-embedding 期间 storage 翻倍**：双 column / 双 table → cost 翻倍
4. **HNSW build 慢极**：升级时 rebuild HNSW index 是 25-100× build IVFFlat 时间——cutover 窗口长

→ PASE = 工程便利 ≠ algorithmic 解。

### 与之前 ingest 的演进

| | wei-2020-analyticdb-v post | **yang-2020-pase post (NEW)** |
|---|---|---|
| Vector update 工具链 | + ADBV SQL UDF re-embed | **+ PG ACID + WAL replication** |
| Embedding upgrade algorithmic 解 | 仍 zero | **仍 zero**（4 个 ingest 后均确认） |
| Cutover 工程支持 | ADBV SQL 友好 | **PG 标准 schema migration**（更广为熟知） |
| Index rebuild 时间 | VGPQ 144 min on AliCommodity | **PASE HNSW SIFT 1M 5000s——升级窗口长** |

### 已知盲区（仍未覆盖）

- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo，仍未 ingest，是唯一已知 algorithmic 答案 source
- **Mapping function 学习**：仍 zero
- **跨 model embedding fine-tune**：仍 zero
- **PASE production 实际 embedding 升级 history**：Ant Financial 升级历史 docs 不公开

## Cited Pages

- [systems/pase.md](../../systems/pase.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
