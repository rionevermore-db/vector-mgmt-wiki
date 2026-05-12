---
title: Databricks Vector Search
type: system
sources: [databricks-docs]
related: [
  ./snowflake-cortex-search.md,
  ./pinecone.md,
  ./lancedb.md,
  ../concepts/hnsw.md,
  ../concepts/blockmaxwand-bm25.md,
  ../topics/sparse-dense-hybrid-retrieval.md,
]
created: 2026-05-12
updated: 2026-05-12
---

# Databricks Vector Search

**TL;DR**: Databricks 数据智能平台内嵌的 vector search 服务, 索引建在 **Delta tables** 之上. **明确技术栈**: HNSW + L2 距离 + Okapi BM25 + Reciprocal Rank Fusion (RRF, `rrf_param=60`). 与 Snowflake Cortex Search 形成数据湖原生 vector search **2024-2025 双雄**, 但 Databricks **公开内部架构** (HNSW + BM25 + RRF), Snowflake 是 closed-box. 两 endpoint 类型: **Standard** (~320M vectors at 768d, 高 QPS) vs **Storage-optimized** (Public Preview, ~1B vectors, 10-20× faster indexing, +250ms latency). **核心 differentiator**: Delta Sync Index 模式 = source table 更新 → index 自动同步 (CDF-driven), 是流水线 freshness 优化的关键.

## 架构图

```
                  ┌──────────────────────────────────────┐
                  │  Delta Table (CDF Enabled)           │
                  │  ───────────────────────────────────  │
                  │   doc_id | content | metadata        │
                  │   (Source of Truth, ACID)            │
                  └───────────────┬──────────────────────┘
                                  │
                       Change Data Feed (CDF) tracks updates
                                  │
                  ┌───────────────▼──────────────────────┐
                  │  Vector Search Endpoint              │
                  │  (Workspace-scoped, max 500/ws)      │
                  ├──────────────────────────────────────┤
                  │ Index Type:                          │
                  │   ┌─────────────────────────────────┐│
                  │   │ Standard Endpoint               ││
                  │   │  • HNSW + L2 distance           ││
                  │   │  • ~320M vectors @ 768d         ││
                  │   │  • High QPS                     ││
                  │   ├─────────────────────────────────┤│
                  │   │ Storage-Optimized (Preview)     ││
                  │   │  • ~1B vectors                  ││
                  │   │  • 10-20× faster indexing       ││
                  │   │  • +250ms latency               ││
                  │   └─────────────────────────────────┘│
                  └───────────────┬──────────────────────┘
                                  │
                                  │  REST API query
                                  ▼
                  ┌──────────────────────────────────────┐
                  │ Hybrid Retrieval                     │
                  │  • Vector path: HNSW + L2            │
                  │  • Lexical path: Okapi BM25          │
                  │  • Fusion: RRF (rrf_param=60)        │
                  └──────────────────────────────────────┘
```

[per sources/docs/databricks/vector-search-overview.md]

## 数据流 / 控制流

### Index 构建
1. User: 在 Delta table 上启用 Change Data Feed (CDF) — `ALTER TABLE ... SET TBLPROPERTIES (delta.enableChangeDataFeed = true)`
2. Create Vector Search Index from Delta table — 3 种 embedding 模式:
   - **Databricks-managed embeddings**: 指定 Foundation Model APIs 或 model serving endpoint
   - **Self-managed pre-calculated embeddings**: User pre-compute, 直接 ingest
   - **Direct vector access**: 已存 vector 列直接索引
3. Endpoint 类型选择:
   - **Standard**: 高 QPS, ~320M vectors / 768d 上限
   - **Storage-optimized (Preview)**: ~1B vectors, indexing 快 10-20×, query latency +250ms
4. CDF-driven incremental sync: source Delta table 更新 → index 自动 reflect

### Query
1. Client REST 调用 `query()` 或 SQL function
2. 3 retrieval modes:
   - **Vector only**: HNSW + L2 (注: cosine 需 client-side normalize embedding)
   - **Hybrid**: Vector + BM25 keyword, RRF fusion with `rrf_param=60`
   - **Filter**: Metadata column filtering (50 columns/index max)
3. 返 top-K + Delta row metadata

## 关键设计决策

| Decision | Trade-off |
|---|---|
| **HNSW + L2 (不是 cosine)** | 优势: HNSW 是 wiki [proximity-graph](../concepts/hnsw.md) 工业 default, 透明 + 成熟; 劣势: cosine similarity 必须 client normalize embedding (与 default cosine 用户认知不同) |
| **Delta table + CDF integration** | 优势: ACID 数据源 + 自动 incremental refresh (vs Snowflake Cortex 的隐式 refresh); 劣势: 必须 source data 在 Delta format, 跨 lake/warehouse migration 不便 |
| **2-tier endpoint (Standard / Storage-optimized)** | 优势: 用户按规模选——320M 内用 Standard 低延迟, 1B 用 Storage-optimized 接受 +250ms; 劣势: 跨 tier 切换需 reindex |
| **Hybrid 默认开启 + RRF rrf_param=60** | 优势: 透明可调 (`rrf_param` 公开 + BM25 公开), 比 Snowflake Cortex 黑盒更可控; 劣势: RRF fusion 简单 (vs learned-fusion), 复杂 query 可能 suboptimal |
| **HNSW + L2 + RRF 默认配方** | 这是 wiki 内 **第一个 production vendor 完全公开 hybrid 内部架构**——Pinecone / Snowflake / Vespa 多少有黑盒部分; Databricks **透明 reproducibility**. |
| **Workspace 500 endpoint 上限** | 大企业多团队场景: 500 endpoint 通常够; 但 RAG-per-customer SaaS 场景 (例每 customer 一 endpoint) 可能不够, 需 multi-workspace |
| **50 columns/index, 50 indexes/endpoint** | 限制 multi-attribute filter scale, 影响 PathFinder-style ([concepts/frontier-2025-distributed-vector-search.md]) filter optimizer 应用空间 |

## Scale 边界

[per sources/docs/databricks/vector-search-overview.md]

| Axis | Limit |
|---|---|
| Per workspace | 500 endpoints |
| Per Standard endpoint | ~320M vectors @ 768d |
| Per Storage-optimized endpoint | ~1B vectors |
| Per index | 50 columns, 50 indexes/endpoint |
| Max embedding dim | 4096 |

> Storage-optimized 1B vectors 是 wiki 内 production-ready 公开数据点之一 (vs SPIRE 8B academic / Meta DistributedANN 1.5T internal / Pinecone 闭源细节). **Databricks Vector Search 是 production 万亿规模的 stepping stone**, 但不到 10B+.

## 与同类系统对比

| 与 Databricks 对比 | 共同 | 差异 |
|---|---|---|
| [Snowflake Cortex Search](./snowflake-cortex-search.md) | 数据湖原生 + governance + hybrid | Databricks 公开 HNSW + BM25 + RRF, Snowflake 黑盒; Databricks 1B 上限 vs Snowflake 100M; Databricks 用 Delta + CDF, Snowflake 用 Snowflake-native materialization |
| [Pinecone](./pinecone.md) | 商业 SaaS hybrid | Pinecone 独立专用 vector DB; Databricks 是 platform feature, 必须 Databricks workspace |
| [LanceDB](./lancedb.md) | format-first lakehouse 向量 | LanceDB 完全 OSS + Lance format; Databricks 闭源 + Delta format. **两者代表 lakehouse 内 vector 的开源 vs 商业路径** |
| [Pinecone Serverless](../concepts/pinecone-serverless-slabs.md) | 多租户 serving | Databricks endpoint = 单租户 (workspace-scoped); Pinecone serverless = 完全 multi-tenant |

## 生产案例

> [推测, wiki 未覆盖]: Databricks Vector Search 主要客户是 Databricks 已有 Mosaic AI / Spark / Delta Lake 客户, 在 enterprise DI + RAG 上做加层. 公开案例需进一步 vendor 文档查找.

## 关键 wiki 影响

1. **数据湖原生 vector search 范式** (与 Snowflake Cortex Search 共同代表)——vector search 作为数据平台 feature 而非 separate DB.
2. **Databricks 是 wiki 内 first production vendor 完全公开 hybrid 内部架构** (HNSW + L2 + BM25 + RRF=60). 这给 production RAG 复现 + tuning 提供 transparent 基线.
3. **Delta + CDF freshness 模式**——data ACID + index 自动同步, 是 wiki 内"streaming vector index"概念的 vendor 实现样本.
4. **2-tier endpoint design (Standard 320M vs Storage-optimized 1B)**——明确把 "high QPS small index" 和 "low QPS large index" 作 product axis, 与 Pinecone p1/p2/s1 pod axis 是同类设计但 Databricks 公开数字.
5. **RRF rrf_param=60 = wiki 内首个具体 RRF 参数 production data point**——之前 wiki 提到 RRF 但无具体常数. Databricks 验证 60 是 production-tested default.

## Open Questions

- **Storage-optimized 内部架构**: 是 SPANN-style hierarchical? DiskANN-style proximity graph + SSD? Databricks 未明示. 10-20× 索引快暗示 IVF-like 而非 graph build.
- **CDF lag 实测**: source Delta table commit → vector index visible 的实测延迟? Production RAG freshness-critical 应用关键数字.
- **HNSW 参数 (M, efConstruction)**: 是否暴露? Standard endpoint 一刀切 vs user-tunable?
- **跨 endpoint hybrid (e.g., 一个 endpoint 做 sparse, 一个做 dense)**: 是否支持 cross-endpoint fusion? RRF 仅 single-endpoint 内 hybrid?
- **与 Mosaic AI Quality Lab integration**: Mosaic AI Agent Framework + Vector Search 共同使用时, 哪些 evaluation tooling 自动 wire up?
