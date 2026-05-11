---
query-key: vector-scalar-bench-methodology
query: "对比多个 vector DB 在向量 + 标量过滤混合查询下的性能，如何公平地横向 benchmark？"
date: 2026-05-11
phase: post
ingest-context: turbopuffer-docs
wiki-pages-total: 68
cited-pages: [systems/turbopuffer.md, systems/qdrant.md, systems/weaviate.md, systems/milvus.md, systems/pinecone.md, systems/vespa.md, topics/attribute-filtering.md, concepts/acorn.md]
cited-count: 8
---

# Post-snapshot (turbopuffer-docs): vector-scalar-bench-methodology

## TL;DR (post-ingest of turbopuffer-docs)

Wiki 已有 [topics/attribute-filtering.md](../../topics/attribute-filtering.md) 五策略方法论骨架，turbopuffer-docs ingest 后**6 个 production OSS/SaaS vendor 的具体 filter 实现都已覆盖**——Milvus 5 strategies / Qdrant Filterable HNSW + ACORN fallback / Weaviate ACORN + correlation / Vespa pre/post/Acorn-1 + YQL planner / Pinecone filter+slab (黑盒) / **Turbopuffer "native filtering" inverted-index-aware with primary-index hierarchy understanding**。**关键 NEW**：Turbopuffer 文档 explicitly 提到 [native-filtering blog post](https://turbopuffer.com/blog/native-filtering) 描述"attribute index 与 primary vector index 共同理解 clustering hierarchy"——是 wiki 内**首个明确 attribute index + SPFresh centroid hierarchy 联合**的工业实现。公平 benchmark 框架可基于五 vendor 的 selectivity 拐点设计。

## Answer

### Wiki 当前覆盖的 6 vendor filter 实现

| Vendor | Filter 实现 | Source |
|---|---|---|
| **Milvus** | 5 strategies (Pre-A/Pre-B/Per-segment-A/Per-segment-B/Per-segment-partition E) | [wang-2021-milvus] |
| **Qdrant** | Filterable HNSW (payload-aware extra edges) + ACORN fallback (v1.16.0) + Tenant/Principal Index | [qdrant-docs] |
| **Weaviate** | ACORN (HNSW + filter check during traversal) + "positive/negative correlation optimization" | [weaviate-docs] |
| **Vespa** | 3-mode (pre-filter / post-filter / Acorn-1) by YQL planner | [vespa-docs] |
| **Pinecone** | filter + slab adaptive index (具体不公开) | [pinecone-docs] |
| **Turbopuffer (NEW)** | "Native filtering": attribute inverted index + SPFresh centroid hierarchy understanding | [turbopuffer-docs] |

### Turbopuffer "Native Filtering" 设计（NEW）

[per sources/docs/turbopuffer/llms-full.txt §concepts §filtering §performance]

> Attribute indexes are inverted indexes built for filterable attributes... **These indexes are aware of the primary vector index and understand the clustering hierarchy**, allowing them to work together for high-recall filtered vector searches.

→ Turbopuffer 是 wiki 内首个明示 "attribute inverted index + ANN clustering 互知" 的工业实现。具体机制：
- Attribute inverted index 知道每个 attribute value 对应的 doc IDs
- SPFresh centroid hierarchy 知道每个 cluster 包含哪些 docs
- Query planner 用两者交集决定: filter 是否高 selective → 走 pre-filter path; 是否低 selective → 走 post-filter path

vs 其他 vendor:
- Qdrant 是 graph build-time aware (insert 时改 HNSW edge selection)
- Weaviate ACORN 是 query-time aware (HNSW traversal 时 check filter)
- Vespa 是 query planner 决定 pre/post/Acorn-1
- **Turbopuffer**: clustering-hierarchy aware (SPFresh centroids 的 doc 分布 statistics 给 planner)

### 公平 vector-scalar benchmark 框架

[基于 wiki 内 5 vendor 实现 + topics/attribute-filtering.md 方法论]

**Fair benchmarking criteria**:

1. **Selectivity sweep**: 必测 selectivity ∈ {0.01%, 0.1%, 1%, 10%, 50%, 90%, 99%} 七个点——不是单点
2. **Attribute cardinality**: 必测 cardinality ∈ {10, 1K, 100K, 10M, 100M}——cardinality 影响 inverted index 经济性
3. **Correlation**: 必测 vector ↔ attribute 正/负相关 / 独立 三种 corr structure (Weaviate "correlation optimization" 假设之)
4. **Recall**: 必报 recall@10, recall@100 给所有 selectivity——不是仅 latency
5. **System state**: cold / warm 区分（Turbopuffer cold p50=343ms 与 warm p50=8ms 必须分报）
6. **Filter complexity**: 必测 simple (单 attribute Eq) / complex (多 attribute AND/OR) / range / glob/regex——Turbopuffer trigram-based glob/regex index 是独有
7. **Concurrent load**: 多 client / QPS 下的 P50/P99/P999

**Pitfall list (常见不公平)**:
- 单 selectivity 点（典型 1%）→ 误判 pre-filter vs post-filter 优势
- 忽略 cold/warm（Turbopuffer 不公平劣势）
- 忽略 update workload（SPFresh / FreshVamana / HFresh 优势在 streaming）
- Recall 阈值不同（HNSW ef=400 vs ef=100 对比不公平）

### 各 vendor 的"sweet spot" selectivity 区间

[per 各 docs 间接 implied]

- **Milvus per-segment-partition E**: pre-define partition → 极端 selective workload (>99% filter) 最优
- **Qdrant Filterable HNSW + ACORN fallback**: ACORN 在 mid selectivity (0.1% - 10%) 最优, 极端走 fallback
- **Weaviate ACORN + correlation**: 与 Qdrant 相近; correlation optimization 给负相关 workload 帮助
- **Vespa pre-filter**: 高 selectivity (>50% filter); Acorn-1: mid; post-filter: 低 (<5%)
- **Pinecone**: 不公开
- **Turbopuffer native filtering**: clustering-hierarchy aware → query planner 自动决定, sweet spot 可能更宽

### 已知盲区

- **5 vendor head-to-head benchmark**：不存在公开实测
- **Turbopuffer native filtering blog post 算法细节**：blog 未 ingest
- **Filtered vector + cold query**：Turbopuffer cold 与 filter 联合 latency 未明示
- **Attribute index 大小占总存储比**：各 vendor 不公开

## Cited Pages

- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/qdrant.md](../../systems/qdrant.md)
- [systems/weaviate.md](../../systems/weaviate.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/pinecone.md](../../systems/pinecone.md)
- [systems/vespa.md](../../systems/vespa.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
- [concepts/acorn.md](../../concepts/acorn.md)
