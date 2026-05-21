---
source-key: lancedb-docs-2026-05
title: LanceDB 官方文档深化快照 (docs.lancedb.com llms.txt + 深页 WebFetch)
vendor: LanceDB Inc. (Chang She + Lei Xu 2022)
source-url: https://docs.lancedb.com
acquisition-method: llms-txt-catalog + per-page WebFetch (indexing / quantization / storage / enterprise / geneva)
fetched-at: 2026-05-21
supersedes-note: 深化 2026-05-12 的 partial ingest (sources/docs/lancedb/, 当时仅 GitHub README + docs.lancedb.com 摘要)。旧快照不可修改,本目录是新增 dated 快照。
files:
  - lancedb-deepened.md
---

# LanceDB 文档深化快照 (2026-05-21)

`docs.lancedb.com` 的深页抓取,补足 2026-05-12 partial ingest 缺的 Enterprise 架构 / 完整索引族 / quantization / storage tier / Geneva / benchmark。

## 获取流程

1. `docs.lancedb.com/llms-full.txt` —— 存在但 **API-focused**(SDK/REST 为主,架构与索引内核薄),不够用
2. `docs.lancedb.com/llms.txt` —— 有效 catalog,从中取深页 URL
3. 逐页 WebFetch:`indexing/{vector-index, quantization}.md` / `storage/index.md` / `enterprise/{architecture, benchmarks}.md` / `geneva/index.md`

## 引用约定

- Citation 形如：`[per sources/docs/lancedb-2026-05/lancedb-deepened.md]`
- Source key 注册为 `lancedb-docs-2026-05`(原 partial 为 `lancedb-docs`,两者并存)
- **benchmark 数字是 vendor 自测**(LanceDB Enterprise 自家 benchmark 页),引用须标 self-published 偏向
