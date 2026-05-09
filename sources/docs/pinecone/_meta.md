---
source-url: https://docs.pinecone.io/llms-full.txt
upstream-site: https://docs.pinecone.io
fetched-at: 2026-05-09
acquisition-method: llms-full-txt + llms-txt
api-version: 2025-10
files:
  - llms-full.txt: full prose content, 87K lines / 3.7 MB, fetched 2026-05-08
  - llms.txt: URL catalog index, 414 lines / 75 KB, fetched 2026-05-09
---

# Pinecone 文档快照

`https://docs.pinecone.io` 的两份 LLM-friendly 文档：

1. **`llms-full.txt`** (2026-05-08, 87K 行 / 3.7 MB)：完整 prose 内容；wiki query 时正文引用的 source。
2. **`llms.txt`** (2026-05-09, 414 行 / 75 KB)：URL catalog；每 doc 一行 `[Title](URL): one-sentence description` 形式——便于快速浏览 doc 列表 + 找到具体 doc 直接 URL。

Pinecone 是商业闭源 SaaS，文档是公开内容的唯一权威来源。

## 获取流程

| 步骤 | 状态 |
|---|---|
| 1. 尝试 `https://docs.pinecone.io/llms-full.txt` | ✓（path C 优先级 1）|
| 2. 尝试 `https://docs.pinecone.io/llms.txt` (catalog) | **✓ 2026-05-09 新增** |
| 3. GitHub clone | 不需要（Pinecone 闭源 docs） |
| 4. Sitemap crawl | 不需要 |

## 两文件的角色分工

- **llms-full.txt**: query workflow 中正文引用源；citation `[per sources/docs/pinecone/llms-full.txt §<section title>]`
- **llms.txt**: doc 索引快速查找；citation `[per sources/docs/pinecone/llms.txt]`（少用——通常引 full.txt）

## 主要章节（基于 llms-full.txt）

```
# Agentic IDEs and CLIs
# Concepts                       ← 核心概念
# Architecture                   ← 系统架构（write/read 分离 + slab）
# Pinecone documentation
# Quickstart
# Test Pinecone at scale         ← 10M 向量基准测试指南
# Check data freshness
# Create an index
# Data ingestion overview
# Data modeling
# Dedicated Read Nodes           ← provisioned hardware 路线
# Implement multitenancy
# Import records
# Indexing overview
# Upsert records
# Pod-based ...（多个 legacy 章节）
# Scale pod-based indexes
# Understanding pod-based indexes
# Manage cost / Understanding cost
# Backup an index
# Delete records / Fetch records / List record IDs
# Manage serverless indexes
# Manage namespaces
... 等
```

## llms.txt vs llms-full.txt 的内容差异（NEW）

[per llms.txt 2026-05-09]

llms.txt 比 llms-full.txt 晚一天 fetch；可能包含的 delta：
- **2026 release notes** entry (line 11)——可能是 2026-05-08 → 05-09 间新增
- 不包含具体 doc 内容——仅 URL + 1 sentence

但**绝大多数 doc URL 都已在 llms-full.txt 中作为 `# Section` 标题出现**——内容上 95%+ 重叠。

## 与 wiki 现有 source 的关系

- **正交**：wang-2021-milvus 的 Table 1 把 Pinecone 列为闭源对手；本 source 是 Pinecone 自家描述
- **Milvus docs 的 comparison.md 显式对比 Milvus vs Pinecone**：本 source 提供 Pinecone 一侧的具体数字
- wiki query archive 明确 flag "Pinecone pod-based 架构未覆盖"——已通过 2026-05-08 ingest 填补

## 引用约定

- Citation 格式：
  - 引正文：`[per sources/docs/pinecone/llms-full.txt §<section title>]`
  - 引 URL list：`[per sources/docs/pinecone/llms.txt → <doc URL>]`
- Source key 在 [sources/README.md](../../README.md) 注册为 `pinecone-docs`
- 商业产品文档版本快速演进；当 Pinecone 重大版本变化时新建 `sources/docs/pinecone-<YYYY-MM>/` 快照

## 关键文档版本声明

- API version 2025-10 是当前稳定（dedicated read nodes 的 `scan_factor` / `max_candidates` 需 ≥2025-10）
- Pod-based indexes 对 2025-08-18 后注册的新 Standard/Enterprise 客户**关闭**——本快照含 legacy pod docs 但仅作历史参考
- 2026 release notes entry 自 2026-05-09 llms.txt 列出（具体内容需查 llms-full.txt 或 docs.pinecone.io 站）
