---
title: Snowflake Cortex Search
type: system
sources: [snowflake-docs]
related: [
  ./databricks-vector-search.md,
  ./pinecone.md,
  ./turbopuffer.md,
  ../concepts/modern-embedding-paradigms.md,
  ../concepts/blockmaxwand-bm25.md,
  ../topics/sparse-dense-hybrid-retrieval.md,
]
created: 2026-05-12
updated: 2026-05-12
---

# Snowflake Cortex Search

**TL;DR**: Snowflake 数据仓库内嵌的 **hybrid retrieval service** (vector + keyword + semantic rerank), 2024-2025 起作为 Cortex 系列 AI 功能之一. 关键定位: **数据湖原生** vector search——索引建在 Snowflake 表上, 不需要 ETL 出仓; 计算/存储分离架构原生支持 (indexing warehouse + 多租户 serving + materialized index storage). **默认 embedding model = `snowflake-arctic-embed-m-v1.5`** (768d, 512-token, 英语), 也支持 `arctic-embed-l-v2.0` 多语言/长 context + Voyage. **首要约束: 单 service 100M rows 上限**——是 RAG 中等规模工具, 不是亿+规模通用 vector DB.

## 架构图

```
                  ┌──────────────────────────────────────┐
                  │  User SQL: CREATE CORTEX SEARCH      │
                  │  SERVICE FROM SELECT * FROM table    │
                  └───────────────┬──────────────────────┘
                                  │
              ┌───────────────────┴──────────────────┐
              │                                       │
              ▼                                       ▼
    ┌─────────────────┐                    ┌─────────────────┐
    │  User Warehouse │                    │  Source Snowflake│
    │ (indexing tier) │                    │     Table       │
    │  MEDIUM or less │◄──── reads from ───┤  (Delta-like)   │
    └────────┬────────┘                    └─────────────────┘
             │
             │ 1. Compute embeddings
             │ 2. Build index structures
             ▼
    ┌─────────────────────────────────┐
    │ Materialized Index (User Account)│
    │  • vector index (HNSW? IVF?)    │ ← 索引内部不公开
    │  • inverted index (BM25-style)  │
    │  • per-row stored vectors       │
    │  • limit: ≤ 100M rows           │
    └────────┬────────────────────────┘
             │
             │ Query via REST / SQL function
             ▼
    ┌─────────────────────────────────┐
    │  Multi-tenant Serving Compute   │
    │  (Snowflake-managed, charged    │
    │   per GB/month indexed data)    │
    │   • Hybrid retrieve             │
    │   • Semantic rerank             │
    │   • Default 20 QPS/service      │
    │   • 140 QPS/account-wide        │
    │   • HTTP 429 if overload        │
    └─────────────────────────────────┘
```

[per sources/docs/snowflake/cortex-search-overview.md]

## 数据流 / 控制流

### Index 构建
1. User SQL: `CREATE CORTEX SEARCH SERVICE my_search ON column_name FROM SELECT ... FROM source_table`
2. Service materializes 查询结果到优化索引结构 (vector 索引 + keyword 索引)
3. Indexing 用 user-provided virtual warehouse 算 embedding + build——**warehouse 大小受限 (MEDIUM or smaller recommended)**
4. Index 存到 user account (storage 计 GB/month)
5. Source table 更新时, index incrementally refresh (Delta-CDC-style, 但 Snowflake 内部实现 not exposed)

### Query
1. Client REST 调用 (or SQL function `SEARCH(...)`)
2. Hybrid pipeline 内部 3 阶段:
   - **Vector retrieve**: 语义相似 documents
   - **Keyword retrieve**: 字面匹配 (BM25-style, but Snowflake **未点名 BM25**)
   - **Semantic rerank**: 在合并 candidate 上 rerank (可禁用)
3. 返 top-K + metadata

## 关键设计决策

| Decision | Trade-off |
|---|---|
| **HNSW vs IVF vs 其他** | Snowflake **未公开索引算法**——黑盒. 推断 HNSW (因低延迟 + 中等规模 fit), 但无 source 确认 |
| **数据湖原生 (不出仓)** | 优势: 无 ETL, 与 Snowflake 表 + governance 一体; 劣势: 100M 行硬上限, 不适合通用大规模 |
| **Multi-tenant serving** | 优势: 用户不管 serving infra, 按 GB/month 计费; 劣势: 默认 20 QPS/service 上限, throughput 受限 |
| **Default embedding = arctic-embed-m-v1.5** | 优势: Snowflake 自研 model, integration deep; 劣势: 768d 英语单语言, multilingual 需切 `arctic-embed-l-v2.0` |
| **Optional Voyage-multilingual-2** | 32k token context 长文档优势; 但依赖第三方 (Voyage AI), 跨 cloud 调用 cost |
| **Hybrid 默认开启** | 优势: 自动 sparse+dense fusion; 劣势: BM25 vs vector 权重不公开, tuning 不灵活 |
| **Semantic rerank 可禁用** | 灵活——latency-critical 场景可关 rerank |

## Scale 边界

[per sources/docs/snowflake/cortex-search-overview.md]

| Axis | Limit |
|---|---|
| Service 内最大行数 | **100M rows** (硬上限) |
| Default throughput | 20 QPS/service, 140 QPS/account |
| Embedding dim | 取决于选 model (arctic-m=768, arctic-l=1024, voyage=自定) |
| Source table size | 隐式由 100M-row materialized 上限决定 |
| Latency target | ~ sub-second (具体 SLA 未公开) |

> **核心定位**: ≤ 100M docs RAG. 千亿规模工具 (e.g. SPIRE / DistributedANN, [per concepts/frontier-2025-distributed-vector-search.md]) 与 Cortex Search 不在同 scale tier.

## 与同类 / 邻接系统对比

| 与 Cortex Search 对比 | 共同 | 差异 |
|---|---|---|
| [Databricks Vector Search](./databricks-vector-search.md) | 数据湖原生 + 集成 governance + hybrid retrieval | Databricks 公开 HNSW + BM25 + RRF (透明); Cortex 索引算法黑盒, 100M 行上限 vs Databricks 1B vector 上限 (storage-optimized) |
| [Pinecone](./pinecone.md) | RAG-optimized SaaS, multi-tenant serving | Pinecone 独立专用 vector DB (非数据湖原生), 规模 10B+; Cortex 在 Snowflake 内, 100M 上限但数据已在仓 |
| [Turbopuffer](./turbopuffer.md) | SaaS multi-tenant + 对象存储原生 | Turbopuffer 对象存储原生 (S3 + cache); Cortex 在 Snowflake 表存储上 (列式 Parquet) |
| [pgvector](./pgvector.md) | DBMS 内嵌 vector | pgvector 是 OSS Postgres extension; Cortex 是闭源 Snowflake 服务 + 数据湖原生 |

## 生产案例

> [推测, wiki 未覆盖]: Snowflake Cortex Search 公开案例尚未广泛披露——2024 GA, 主要客户在 enterprise BI shop (Snowflake 已有客户内做 RAG-in-warehouse).

## 关键 wiki 影响

1. **数据湖原生 vector search 范式**——新 wiki concept: vector search **不一定要 separate 系统**, 可作为数据仓 / 数据湖的 add-on feature. 与 Databricks Vector Search 共同代表此 paradigm.
2. **Cortex Search 100M 行上限 = enterprise BI RAG 典型规模**——不是千亿/万亿规模, 而是 "把内部企业数据 (10M ~ 100M docs) 拿来做 RAG" 的 sweet spot.
3. **黑盒 vector index algorithm**——Snowflake 不公开 HNSW/IVF 选择, 与 Pinecone 公开 pod-based 架构 / Databricks 公开 HNSW + BM25 + RRF 形成对比. 这是 **closed-SaaS 与 open-system 在 transparency 上的 axis difference**.
4. **Snowflake Arctic Embed family**——`arctic-embed-m-v1.5` / `arctic-embed-l-v2.0` 是 Snowflake 自研 embedding model, OSS HF available. 与 NV-Embed / E5-Mistral / BGE-M3 是 [embedding model paradigm](../concepts/modern-embedding-paradigms.md) 选项之一, wiki 未单独 page 但 Cortex Search 默认使用是重要 production deployment.

## Open Questions

- **底层 vector index 算法**: HNSW? IVF + PQ? 数据湖原生暗示 Parquet-block 友好 layout (e.g. SPIRE-style hierarchical), 但 100M 上限暗示 single-node HNSW more likely. Snowflake 未公开.
- **BM25 vs 其他 lexical search**: Snowflake 用"keyword search" 含糊措辞, 不是确定 BM25——可能是 Cortex 自研 lexical 实现.
- **Snowflake Arctic Embed 与 NV-Embed / BGE-M3 性能 head-to-head**: arctic-embed-m-v1.5 在 MTEB 上排名? OSS leaderboard 数据点不足.
- **100M 上限的 architectural origin**: 是 serving infra 单 node memory bound 还是 materialized table 单文件大小限制?
- **incremental refresh 延迟**: Source table 更新 → index 同步的 lag, 影响 freshness-sensitive RAG 选型.
