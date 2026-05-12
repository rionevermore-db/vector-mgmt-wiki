---
title: Elasticsearch (kNN + dense_vector)
type: system
sources: [elasticsearch-docs]
related: [
  ./opensearch.md,
  ./vespa.md,
  ./weaviate.md,
  ../concepts/hnsw.md,
  ../concepts/blockmaxwand-bm25.md,
  ../concepts/rabitq.md,
  ../topics/sparse-dense-hybrid-retrieval.md,
]
created: 2026-05-12
updated: 2026-05-12
---

# Elasticsearch (kNN + dense_vector)

**TL;DR**: 全球部署量最大的搜索引擎自 ES 8.0 (2022) 起原生支持 ANN/HNSW + dense_vector 字段. 与 OpenSearch (AWS fork) 共同占企业 hybrid retrieval 部署 ~60%+ market. **核心架构**: HNSW 索引**per-segment 构建** (vs single global graph, Lucene-style), 通过 disjunction `score = boost1 * match_score + boost2 * knn_score` 实现 BM25 + vector hybrid. 2024 起加入 **DiskBBQ + int8_hnsw + bfloat16 + bbq_disk** 多种量化, **rescore with original float vectors** 是 production 高 recall 路径. 关键设计: 全部 vector data **必须 fit node page cache** for HNSW 高效——是 in-memory-friendly 但不是 DRAM-free.

## 架构图

```
┌─────────────────────────────────────────────────────────┐
│  Elasticsearch Index                                     │
│  (Lucene-backed, segment-based)                          │
│                                                          │
│   ┌─────────────────┐  ┌─────────────────┐ ...           │
│   │  Segment 1      │  │  Segment 2      │              │
│   │  ┌───────────┐  │  │  ┌───────────┐  │              │
│   │  │ HNSW per  │  │  │  │ HNSW per  │  │              │
│   │  │ segment   │  │  │  │ segment   │  │              │
│   │  └───────────┘  │  │  └───────────┘  │              │
│   │  ┌───────────┐  │  │  ┌───────────┐  │              │
│   │  │ BM25      │  │  │  │ BM25      │  │              │
│   │  │ (inverted │  │  │  │ inverted  │  │              │
│   │  │  index)   │  │  │  │  index    │  │              │
│   │  └───────────┘  │  │  └───────────┘  │              │
│   └─────────────────┘  └─────────────────┘              │
│                                                          │
│   Quantization options (per-field):                      │
│    • float (raw)                                         │
│    • bfloat16 (2-byte/dim)                              │
│    • int8_hnsw                                          │
│    • bbq_disk (DiskBBQ clustering)                      │
└─────────────────────────────────────────────────────────┘
                          │
                          │ Query
                          ▼
        ┌─────────────────────────────────────┐
        │ Disjunction (boolean OR):           │
        │  score = b1 * match_score           │
        │        + b2 * knn_score             │
        │ (user-tunable boost b1, b2)         │
        │                                     │
        │ Optional rescore:                   │
        │  retrieve via int8_hnsw → rescore   │
        │  top-k with original float          │
        └─────────────────────────────────────┘
```

[per sources/docs/elasticsearch/knn-search.md]

## 关键设计决策

| Decision | Trade-off |
|---|---|
| **Per-segment HNSW (vs global)** | 优势: Lucene-native, 与 BM25 inverted index 同 segment-level granularity, refresh / merge 自然集成; 劣势: **跨 segment recall 一定有 fragmentation**——同一 query 在不同 segment 召回不同 candidates, 必须 union + rerank; 单 segment HNSW size 受 segment size 上限 |
| **HNSW data 必须 fit page cache** | "All vector data must fit in the node's page cache for efficient performance" —— 硬约束, 是 **wiki 内 first explicit RAM-bound 公开 vendor 数据点**; 与 DiskANN / SPANN disk-aware vendor 对比鲜明 |
| **Hybrid via disjunction + boost** | 优势: 用户全控 score 配比 (`0.9 * match + 0.1 * knn`); 劣势: RRF-style fusion 在某些 OOD query 上更鲁棒, ES 选 weighted sum 是简单但 task-tuning-dependent |
| **量化: int8_hnsw / bfloat16 / DiskBBQ / bbq_disk** | 4 level 选项, 用户按 storage budget 选; **rescore with original float** 是高 recall 兜底 (PathFinder / SPLADE 同 spirit) |
| **DiskBBQ + visit_percentage** | bbq_disk = **first vector vendor 公开 binary-bit + cluster + visit-percent 控制 axis**——recall vs throughput 拐点 user-tunable |
| **Lucene 自适应 filter strategy** | 小 filtered set → brute force; 大 filtered set → HNSW + post-filter. 与 [PathFinder cost-based optimizer](../concepts/frontier-2025-distributed-vector-search.md) 是 vendor-level 实现 |

## Scale 边界

[per sources/docs/elasticsearch/knn-search.md]

- **Per-node HNSW data must fit page cache** → 单节点 vector 数据规模 ~ 节点 RAM 减系统/JVM 开销
- 索引构建 compute-intensive
- 水平扩展: shards 跨多节点, 但每 shard 内 segment-based HNSW

> 实际部署: 10 节点 ES cluster + 64 GB RAM/node ≈ 10亿 768d float vectors 的 quantize 后 (int8_hnsw 4× 压缩 → ~750 GB total)

## 与同类系统对比

| 与 ES 对比 | 共同 | 差异 |
|---|---|---|
| [OpenSearch](./opensearch.md) | 同源 (AWS fork 自 ES 7.10), HNSW + BM25 | ES 8+ 用 Elastic License (有限制), OpenSearch Apache 2.0; ES 仅 Lucene engine, OS 支持 Lucene + NMSLib + Faiss 多 engine |
| [Vespa](./vespa.md) | 商业搜索引擎 + vector + BM25 hybrid | Vespa 原生 hierarchical (Yahoo 1990s 起), 4-phase ranking; ES segment-based Lucene heritage, hybrid disjunction-based |
| [Weaviate](./weaviate.md) | OSS hybrid retrieval + BM25 | Weaviate 是 vector-first OSS; ES 是 full-text-first + vector-added |
| [Snowflake Cortex Search](./snowflake-cortex-search.md) | Hybrid + 数据仓内 | Cortex 是数据湖原生 (Snowflake 表上), ES 是独立搜索 cluster |

## 生产案例

> [推测, wiki 未覆盖]: ES 全球部署巨大 (DB-Engines ranking 长期前 10), 几乎所有 Fortune 500 在某 application 用 ES. 自 8.0 dense_vector + ELSER neural sparse 上线后, **大量企业 RAG 默认尝试 ES**——但具体客户案例需 Elastic 官方 case studies 查找.

## 关键 wiki 影响

1. **HNSW per-segment 模式**: wiki 内之前 HNSW 描述默认 single-global-graph. ES 是首个**显式 per-segment** 路径——与 Lucene heritage 必然 coupling, 是 ES 独特设计.
2. **page cache 必须 fit vector data**: wiki 内 first **explicit RAM-bound** 公开 vendor 数据点——production scale ceiling 公式: `vector_count × dim × byte/vec ≤ node_RAM × n_nodes - overhead`.
3. **Disjunction-based hybrid (vs RRF)**: ES 用 weighted sum, [Databricks](./databricks-vector-search.md) 用 RRF——production hybrid 2 大 fusion strategy 都有 vendor 验证.
4. **Quantization layer fully exposed**: int8 / bfloat16 / DiskBBQ / bbq_disk 4 选项, 是 wiki 内 first 完全暴露 quantization stack 的 vendor.
5. **Lucene adaptive filter**: filtered count vs num_candidates 比较选 brute-force vs HNSW——production filter 优化 pattern, 与 [PathFinder optimizer](../concepts/frontier-2025-distributed-vector-search.md) 是 algorithm-level 同源.

## Open Questions

- **ELSER (Elastic Sparse Encoder) 详细**: ES 自研 neural sparse, 类 SPLADE 但 Elastic 专用——wiki 未覆盖, 与 [SPLADE family](../concepts/splade-family-baselines.md) 形成对照需 future ingest.
- **per-segment HNSW 与 query recall fragmentation**: 跨 segment merge 是否丢失 candidates? doc 未明示.
- **DiskBBQ vs RaBitQ 关系**: DiskBBQ 是 ES 自研还是基于 [RaBitQ](../concepts/rabitq.md)? Source code 需查.
- **HNSW M / efConstruction 默认值**: Elastic 暴露 user-tunable 但 default 是? Production tuning 需要.
- **跨 shard 全局 top-K merge 策略**: 多 shard 各自 top-K, coordinator merge 时是否有 recall loss? 大 shard count 时是否退化?
