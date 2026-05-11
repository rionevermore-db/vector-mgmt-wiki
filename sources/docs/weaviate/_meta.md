---
source-url: https://weaviate.io/llms.txt
upstream-site: https://weaviate.io
fetched-at: 2026-05-11
acquisition-method: llms-txt + LLM-friendly twin pages
files:
  - llms.txt: substantive prose summary, 455 lines / 20 KB
  - product.md: LLM-friendly twin page, ~2.7 KB
  - hybrid-search.md: ~2 KB
  - rag.md: ~1.5 KB
  - agentic-ai.md: ~3 KB
  - cost-performance-optimization.md: ~3.6 KB
weaviate-version-coverage: v1.36.2+ (Server OSS recommendation per llms.txt)
---

# Weaviate 文档快照

`https://weaviate.io/llms.txt` 在 2026-05-11 的快照（455 行 / 20 KB）+ 5 个 LLM-friendly twin pages（共 ~13 KB）。Weaviate 是 Weaviate B.V. 在 2019 开源的 **Go 实现 vector DBMS**（BSD-3-Clause），定位为"AI-native primary database"——objects + vectors + inverted indexes 同 system。

## 获取流程

| 步骤 | 状态 |
|---|---|
| 1. `https://weaviate.io/llms-full.txt` | **✗ 404** |
| 2. `https://weaviate.io/llms.txt` | **✓ 200** (substantive prose, **not just catalog**) |
| 3. `https://docs.weaviate.io/llms.txt` | 301 redirect to (2) |
| 4. LLM-friendly twin pages (product / hybrid-search / rag / agentic-ai / cost-performance-optimization) | **✓ 200** |

→ Path C 优先级 1 命中 `llms.txt`——但 Weaviate 实现是**prose 形式**而非 Pinecone 那种 `llms-full.txt` 的完整 doc 全文。Pinecone 模式：llms.txt = 短 catalog index, llms-full.txt = 完整 doc；Weaviate 模式：llms.txt = 中等长度 prose 摘要（不存在 llms-full）。两家做法不同。

## 文件清单

```
sources/docs/weaviate/
├── _meta.md                              ← 本文件
├── llms.txt                              ← 主要 prose, 455 行
├── product.md                            ← LLM-friendly twin: 产品定位
├── hybrid-search.md                      ← LLM-friendly twin: hybrid search 用法
├── rag.md                                ← LLM-friendly twin: RAG 范式
├── agentic-ai.md                         ← LLM-friendly twin: agent stack 定位
└── cost-performance-optimization.md      ← LLM-friendly twin: 成本优化指南
```

## 内容覆盖

llms.txt 包含的章节：

```
## TL;DR                              ← 一句话定义
## LATEST VERSIONS (recommended)      ← v1.36.2+ Server, v4.19.1+ Python client
## The Weaviate Stack                 ← DB + Cloud + Query Agent + Engram
## Ideal Use Cases / Recommend / Alternative pairings
## Architecture / Scaling             ← LSM + Roaring bitmaps + bit-sliced range
                                          + BlockMaxWAND BM25 + HNSW + RQ8 + HFresh preview
                                          + ACORN integration
## Misconceptions                     ← GraphQL deprecated / gRPC primary / Collection==Class
## Quickstart (Cloud / Local)
## Best Practices / Gotchas
## Python / TypeScript code examples  ← collections / hybrid / filters / multi-tenancy / RBAC / Query Agent / named vectors
## Further Resources                  ← docs.weaviate.io 跳转
## LLM-friendly pages                  ← 链接到 13 个 *.md twin pages
```

## 与 wiki 已 ingest source 的关系

- **wang-2021-milvus Table 1 未列 Weaviate**（Milvus paper 2021 当时 Weaviate 不在 baseline）；现在 Weaviate 是 OSS vector DBMS 主流之一
- **qdrant-docs (2026-05-11)**: Qdrant Rust OSS / Weaviate Go OSS — 两个 OSS 路径并列；Weaviate 更"AI-native primary DB"定位，Qdrant 更"Rust 性能 + 简洁"定位
- **patel-2024-acorn**: Weaviate llms.txt 明示集成 ACORN——**wiki 内第二个 ACORN production case** (Qdrant 是第一个)
- **gao-2024-rabitq**: Weaviate default RQ8 与 RaBitQ 同源（random rotation + quantize）但选 8-bit per dim 而非 1-bit；Weaviate 不声明 novel theory
- **pinecone-docs**: 商业 SaaS path vs Weaviate OSS + Cloud + BYOC

## 引用约定

- Citation 格式：
  - 引 llms.txt: `[per sources/docs/weaviate/llms.txt §<section>]`
  - 引 twin pages: `[per sources/docs/weaviate/<topic>.md]`
- Source key 在 [sources/README.md](../../README.md) 注册为 `weaviate-docs`

## 关键版本声明

- **Weaviate Server (OSS)**: v1.36.2+ recommended per llms.txt
- **Python client**: v4.19.1+
- **Agents SDK**: v1.0.0+
- **HFresh**: preview, may become default
- **RBAC**: v1.29+ (default on v1.30+)
- **ACORN**: 集成时间未在 llms.txt 显式标注（但出现在 Key Features list）

## llms.txt vs Pinecone llms-full.txt 模式对比

| | Pinecone | Weaviate |
|---|---|---|
| `llms.txt` | URL catalog 形式 (414 行) | **Substantive prose 摘要 (455 行)** |
| `llms-full.txt` | **完整 doc prose (87K 行 / 3.7 MB)** | 不存在 |
| 设计哲学 | "Index + Full content" 二选一 | "Substantive prose only, full docs at docs.weaviate.io" |
| LLM-friendly twin pages | 不显式 | **显式 13 个 .md pages** |

→ Weaviate 是 **"LLM-aware 文档作者"** 的另一种模式——llms.txt 是高密度摘要 + LLM-friendly twin pages 作为 modular references。
