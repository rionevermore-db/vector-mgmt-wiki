---
source-url: https://docs.vespa.ai/llms-full.txt
upstream-site: https://docs.vespa.ai
fetched-at: 2026-05-11
acquisition-method: llms-full-txt + llms-txt
files:
  - llms-full.txt: 92,586 lines / 4.2 MB full prose content
  - llms.txt: 966 lines / 68 KB catalog with overview prefix
vespa-version-coverage: Vespa 8.x (Vespa 8.78+ for multi-threaded pre-filter, 8.287+ for streaming search auto-config)
---

# Vespa 文档快照

`https://docs.vespa.ai/llms-full.txt` + `https://docs.vespa.ai/llms.txt` 在 2026-05-11 的快照（共 4.3 MB）。Vespa 是 Yahoo! 2003 内部开发的 web search + recommendation engine，**2017 开源** (Apache-2.0)，现由 Vespa.ai (前 Yahoo! 团队) 维护。C++ + Java 实现。

## 获取流程

| 步骤 | 状态 |
|---|---|
| 1. `https://vespa.ai/llms-full.txt` | ✗ 404 |
| 2. `https://vespa.ai/llms.txt` | ✗ 404 |
| 3. `https://docs.vespa.ai/llms-full.txt` | **✓ 200** |
| 4. `https://docs.vespa.ai/llms.txt` | **✓ 200** |

→ Path C 优先级 1 命中——Vespa 与 Pinecone 同模式（llms.txt = catalog + llms-full.txt = full prose），但 Vespa llms.txt 前缀含完整 overview prose（966 行）后接 URL list。

## 文件清单

```
sources/docs/vespa/
├── _meta.md                ← 本文件
├── llms-full.txt           ← 92K 行 full prose，覆盖所有 doc pages
└── llms.txt                ← 966 行 overview prefix + URL catalog
```

## 关键章节（基于 llms-full.txt header scan）

```
## Approximate Nn Hnsw                ← HNSW 实现 + Acorn-1 + 多 quantization 类型
## Billion Scale Image Search         ← LAION-5B + CLIP + PCA + HNSW-IF hybrid
## Billion Scale Vector Search        ← **SPANN production deployment**
## Binarizing Vectors                 ← Binary quantization (int8 + 1-bit)
## Bm25                               ← BM25 实现
## Embedding                          ← 内置 embedders + ONNX
## Indexing                           ← LSM storage layer
## Nearest Neighbor Search            ← Exact + ANN, YQL nearestNeighbor operator
## Nearest Neighbor Search Guide      ← 实用指南
## Phased Ranking                     ← 4-phase ranking (retrieval / first / second / global)
## Practical Search Performance Guide ← 性能调优
## Ranking                            ← Tensor framework + ONNX/XGBoost/LightGBM
## Reranking In Searcher              ← Custom Java reranker
## Search                             ← 搜索流程
## Streaming Search                   ← **No-index per-user search (45 bytes/doc)**
## Tensor Examples                    ← Tensor framework usage
## Tensor User Guide                  ← Tensor primitives
## Vector Search Intro                ← 向量搜索入门
... (其他 100+ 章节)
```

## 与 wiki 已 ingest source 的关系

- **wang-2021-milvus Table 1 列 Vespa 为开源对手**——但当时 Milvus 论文未深入；本 source 是 Vespa 自家完整描述
- **chen-2021-spann SPANN 论文**：Vespa 实现 SPANN 作 billion-scale ANN——**wiki 内 SPANN 第二个 production deployment** (继 Microsoft Bing)
- **patel-2024-acorn ACORN 论文**：Vespa "Acorn-1" 模式（filtering before distance calculation）——**wiki 内 ACORN 第三个 production deployment** (继 Qdrant + Weaviate, frontier 完全闭合)
- **gao-2024-rabitq**：Vespa 不集成 RaBitQ；当前 binary quantization 是与 Qdrant / Weaviate 同代 production
- **qdrant-docs / weaviate-docs / pinecone-docs / milvus-docs**：四 OSS/SaaS vector DBMS 共存的"vector-first" vendor 哲学；**Vespa 是不同 lineage**（来自 web search engine，不是从 vector store 演化）
- **guo-2022-manu (Milvus 2.x)**：streaming 索引——Vespa Streaming Search 走完全相反路径（no-index brute-force within small subset）

## 引用约定

- Citation 格式：`[per sources/docs/vespa/llms-full.txt §<section>]` 或 `[per sources/docs/vespa/llms.txt overview]`
- Source key: `vespa-docs` (registered in [sources/README.md](../../README.md))

## 关键 Vespa 版本声明

- **Vespa 8.x** 当前主线（2024+）
- Vespa 8.78+: pre-filter 支持多线程
- Vespa 8.287+: streaming search auto-config

## llms.txt vs llms-full.txt 模式对比（更新）

| Vendor | llms.txt | llms-full.txt |
|---|---|---|
| Pinecone | 414 行 URL catalog | 87K 行 full prose |
| **Vespa** | **966 行 overview prefix + URL catalog** | **92K 行 full prose** |
| Qdrant | ✗ 404 | ✗ 404 |
| Weaviate | 455 行 substantive prose | ✗ 404 |

→ Vespa 采用 **"hybrid"**：llms.txt 既有 overview prose 也有 URL list；llms-full.txt 是 full prose dump。是 wiki 内迄今**最完整 LLM-friendly 文档发布模式**。
