---
title: OpenSearch (k-NN plugin)
type: system
sources: [opensearch-docs]
related: [
  ./elasticsearch.md,
  ./faiss.md,
  ../concepts/hnsw.md,
  ../concepts/splade-sparse-retrieval.md,
  ../topics/sparse-dense-hybrid-retrieval.md,
]
created: 2026-05-12
updated: 2026-05-12
---

# OpenSearch (k-NN plugin)

**TL;DR**: AWS 2021 年 fork Elasticsearch 7.10 (Elastic 改 license 时), OSS Apache 2.0 永久. k-NN plugin **支持 3 个 engine 后端: Lucene / NMSLib / Faiss**, 是 wiki 内**唯一多 engine 灵活选择**的 vendor. **关键 differentiator vs ES**: (a) license 永久 OSS; (b) 3 engine 选择 (Faiss 提供 IVF + 多种 quantization); (c) **自研 Neural Sparse encoder** (类 SPLADE/ELSER) + BM25 + vector RRF hybrid; (d) AWS OpenSearch Service managed offering. 与 Elasticsearch 是 hybrid retrieval **双胞胎对手**, 选型主要看 license 偏好 + cloud preference.

## 架构图

```
┌─────────────────────────────────────────────────────────┐
│  OpenSearch Index (Lucene-segment-based 同 ES heritage)  │
│                                                          │
│   k-NN Plugin: choice of engine                          │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│   │ Lucene     │  │ NMSLib     │  │ Faiss      │        │
│   │ (Java)     │  │ (C++,      │  │ (C++ FAIR)│        │
│   │            │  │  deprecated│  │            │        │
│   │            │  │  2024+)    │  │            │        │
│   │  HNSW      │  │  HNSW      │  │  HNSW+IVF  │        │
│   └────────────┘  └────────────┘  └────────────┘        │
│                                                          │
│   Quantization (Faiss path 最丰富):                       │
│    • Scalar (8-bit, 4-bit)                              │
│    • Binary                                              │
│    • Product Quantization                                │
│                                                          │
│   Hybrid Pipeline:                                       │
│    • Neural Sparse (类 SPLADE)                          │
│    • BM25 (Lucene-inherited)                            │
│    • Dense vector kNN                                    │
│    • RRF Processor for fusion                            │
└─────────────────────────────────────────────────────────┘
```

[per sources/docs/opensearch/vector-search-techniques.md]

## 关键设计决策

| Decision | Trade-off |
|---|---|
| **3 engine 选择 (Lucene/NMSLib/Faiss)** | 优势: 用户按需 → Lucene Java-native 集成最深 + Faiss 量化最丰富 + NMSLib 历史延续; 劣势: 维护 cost (NMSLib 2024+ deprecated); 配置复杂度比 ES 单 engine 高 |
| **Faiss engine 带 IVF + PQ** | 优势: 大规模 (10亿+) 内存友好, IVF + PQ 经典组合; 劣势: build cost 高, query latency 比 HNSW 高 |
| **Apache 2.0 永久 OSS** | 优势: 商业可自由 fork (与 ES Elastic License 限制对比); 劣势: 社区 fragmentation 风险 |
| **Neural Sparse 自研 + BM25 + Vector** | 三 retrieval primitive 都有, RRF processor 通用 fusion; 与 [SPLADE family](../concepts/splade-family-baselines.md) 同 spirit 但 AWS-internal model |
| **Search Pipelines + Processor** | 优势: 模块化 (neural query enrichment / sparse two-phase / semantic highlighting); 劣势: pipeline 学习曲线 |
| **AWS OpenSearch Service** | Managed cloud path = default; self-hosted Apache 2.0 也 viable |

## Scale 边界

[per sources/docs/opensearch/vector-search-techniques.md]

> Scale 限制与 Elasticsearch 类似 (Lucene-segment heritage)——单节点 HNSW data fit page cache, 但 Faiss IVF 路径可 disk-resident 突破此约束.

> Real-world: AWS production OpenSearch cluster 10 亿+ vectors 部署存在 (AWS case studies 隐式提及).

## 与 Elasticsearch 对比

[per sources/docs/opensearch/vector-search-techniques.md 注: differentiator section]

| Axis | OpenSearch | Elasticsearch |
|---|---|---|
| **License** | Apache 2.0 永久 OSS | ES 7.10 起 Elastic License (限制 SaaS) |
| **Engine** | Lucene + NMSLib + Faiss | 主要 Lucene (单 engine) |
| **Quantization** | Scalar / Binary / PQ (Faiss path) | int8_hnsw / bfloat16 / DiskBBQ / bbq_disk |
| **Neural Sparse** | OpenSearch Neural Sparse | ELSER |
| **Hybrid Fusion** | RRF processor | Disjunction + weighted boost |
| **Cloud managed** | AWS OpenSearch Service | Elastic Cloud (含 AWS partnership) |
| **生态** | AWS + 社区 fork | Elastic 公司 + 长期客户 |

## 与同类系统对比

[per sources/docs/opensearch/vector-search-techniques.md + 公知]

| 与 OpenSearch 对比 | 共同 | 差异 |
|---|---|---|
| [Faiss](./faiss.md) | OpenSearch Faiss engine 内嵌 Faiss | Faiss 是 library, OpenSearch 是产品包装 Faiss + distributed + REST API |
| [Vespa](./vespa.md) | 商业搜索 + vector + BM25 hybrid | Vespa 原生 4-phase ranking, OpenSearch 用 RRF processor |
| [Weaviate](./weaviate.md) | OSS hybrid retrieval | Weaviate vector-first; OpenSearch full-text-first + vector |

## 生产案例

> [推测, wiki 未覆盖]: AWS 大量内部服务 + AWS 客户在 OpenSearch Service 上做 RAG. 公开案例多见于 AWS re:Invent talks / case studies.

## 关键 wiki 影响

1. **多 engine 灵活路径**: wiki 内 first **vendor 支持多 ANN library 选择**的 system——是 production "engine-agnostic" 路径代表.
2. **Apache 2.0 license axis**: 与 ES Elastic License 形成 license preference 分轴——选 OpenSearch 多是出于 license + AWS preference.
3. **Neural Sparse 是 wiki 内 SPLADE 系统化部署 vendor 之一**: 与 ELSER / SPLADE original 形成 [SPLADE family](../concepts/splade-family-baselines.md) 工业部署 3 例 (但 ELSER + Neural Sparse 都是 vendor-specific model, 不直接 reuse SPLADE OSS).
4. **RRF processor 模块化**: 与 [Databricks RRF=60](./databricks-vector-search.md) 共同代表 RRF fusion vendor 实现, 是 wiki 内 vendor-level RRF 应用 2 例.
5. **Faiss + IVF + PQ in production**: wiki Faiss page 主要算法描述, OpenSearch 给出 Faiss 在 production REST-API-managed 服务中应用的 reference.

## Open Questions

- **NMSLib deprecation 之后**: 历史用 NMSLib 的 index 是否平滑迁到 Lucene/Faiss? 数据迁移 cost.
- **OpenSearch Neural Sparse 与原 SPLADE 性能**: 是 fine-tune from SPLADE 还是 from scratch? AWS 未公开训练细节.
- **AWS OpenSearch Service 与 self-hosted 功能 parity**: managed cloud 是否提供完整 feature set?
- **跨 region OpenSearch 部署**: AWS multi-region OpenSearch hybrid retrieval 的 consistency / freshness 模型?
- **Faiss-engine HNSW vs Lucene-engine HNSW recall 差异**: 同 dataset 同参数, 两 engine 实测 recall 是否一致? Production tuning 关键.
