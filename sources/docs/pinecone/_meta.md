---
source-url: https://docs.pinecone.io/llms-full.txt
upstream-site: https://docs.pinecone.io
fetched-at: 2026-05-08
acquisition-method: llms-full-txt
api-version: 2025-10
---

# Pinecone 文档快照

`https://docs.pinecone.io/llms-full.txt` 在 2026-05-08 的快照（87,495 行 / 3.7 MB）。Pinecone 是商业闭源 SaaS，文档是公开内容的唯一权威来源。

## 获取流程

1. 尝试 `https://docs.pinecone.io/llms-full.txt` → ✓（Pinecone 有官方 llms-full.txt，CLAUDE.md 路径 C 优先级 1 命中）
2. 不需要 GitHub clone（Pinecone 闭源，docs 不公开 markdown 仓库）
3. 单文件存储 `sources/docs/pinecone/llms-full.txt`，结构通过 `^# ` headers 划分 source URL

## 主要章节

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

## 与 wiki 现有 source 的关系

- **正交**：wang-2021-milvus 的 Table 1 把 Pinecone 列为闭源对手；本 source 是 Pinecone 自家描述
- **Milvus docs 的 comparison.md 显式对比 Milvus vs Pinecone**：本 source 提供 Pinecone 一侧的具体数字
- **wiki query archive 明确 flag "Pinecone pod-based 架构未覆盖"**：本 ingest 直接填补

## 引用约定

- Citation 格式：`[per sources/docs/pinecone/llms-full.txt §<section title>]`
- Source key 在 [sources/README.md](../../README.md) 注册为 `pinecone-docs`
- 商业产品文档版本快速演进；当 Pinecone 重大版本变化时新建 `sources/docs/pinecone-<YYYY-MM>/` 快照

## 关键文档版本声明

- API version 2025-10 是当前稳定（dedicated read nodes 的 `scan_factor` / `max_candidates` 需 ≥2025-10）
- Pod-based indexes 对 2025-08-18 后注册的新 Standard/Enterprise 客户**关闭**——本快照含 legacy pod docs 但仅作历史参考
