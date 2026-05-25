# LanceDB reindexing + versioning — captured content (WebFetch, 2026-05-25)

> 来源：docs.lancedb.com `indexing/reindexing.md` + `tables/versioning.md`。补抓页(2026-05-21 深化未含),回答"索引是否多版本"。同属 lancedb-docs-2026-05 快照,新增文件(不改原文件)。

## reindexing（indexing/reindexing.md）

- **新数据在 reindex 前仍可查,但走 flat fallback**:原文 "As data is being added and a reindex operation is running, LanceDB will combine results from the existing index with **exhaustive/flat search on the new data**." → 不漏数据,但 "The more data that you add without reindexing, the impact on latency (due to exhaustive search) can be noticeable."
- **增量,不是全量重建**:`optimize()` 做 "Index update: **adds newly-ingested data to existing** vector, scalar, and FTS indexes" + compaction + cleanup。**Enterprise 自动化;OSS 需手动触发**。
- **旧文件版本默认 7 天保留**后被 prune(pruning old file versions, default 7 days)。
- 未提及:per-version index snapshot / checkout 老版本时用哪一版索引。

## versioning（tables/versioning.md）

- update / add / delete / restore 都产生新 version。
- API:`table.checkout(version)` / `checkout_latest()` / `restore()`;"LanceDB supports **fast rollbacks to any previous version without data duplication**"。
- **关键**:"System operations like **`optimize()`, index updates, and table compaction also increment table version numbers.**" → 索引更新是被记入版本号的事件。
- **未回答**:version 是否捕获 index、旧 version 是否各自保留索引、checkout/restore 时索引如何表现。原文对 index-versioning 语义 **no information / not addressed**。
