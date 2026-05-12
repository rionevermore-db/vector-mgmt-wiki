---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-12
phase: post
ingest-context: formal-2021-splade-v2
wiki-pages-total: 75
cited-pages: [concepts/splade-sparse-retrieval.md, topics/sparse-dense-hybrid-retrieval.md, systems/vespa.md]
cited-count: 3
---

# Post-snapshot (formal-2021-splade-v2): index-architecture-global-vs-routed

## TL;DR (delta from kusupati-2022-matryoshka post)

**SPLADE 不改变 5 架构 (a-e) 的 dense-side 选择**——但 sparse-side **天然适合 (c) 层次路由 + (a) global single inverted index 两种 architecture coexistence**: per-shard inverted index (sharded by doc ID hash) + global merge. 这与 dense path 的 sharding 选择是**两个独立 axis**, hybrid pipeline 同时使用. **关键 NEW**: production hybrid 实际 architecture = **dense path 选 (a-e) 之一 × sparse path 选 (sharded inverted index 简单方案)** × **fusion (Vespa rank-profile / Weaviate hybrid / Turbopuffer multi_query)**.

## Answer

### 与之前 ingest 的演进

| | kusupati-2022-matryoshka post | **formal-2021-splade-v2 post (NEW)** |
|---|---|---|
| 5 架构 (a)-(e) | dense-side only | **不变 (dense-side); + sparse-side 独立 architecture axis** |
| Sparse-side architecture | 未涵盖 | **NEW: sharded inverted index 简单 + 成熟; doc-ID hash partitioning** |
| Hybrid pipeline 总 architecture | dense single axis | **NEW: dense × sparse 独立 architecture, fusion 是 third axis** |

### Sparse path architecture 简明

[per formal-2021-splade-v2 + IR 50 年历史]

**Inverted index sharding 主流方案** (简单 + 成熟):
- Doc-ID hash partitioning (N 个 shard, doc 按 ID 哈希分配)
- Per-shard 独立 inverted index
- Query fanout to all shards → 每 shard 取 top-K → application 合并
- BlockMaxWAND optimization per-shard

**与 dense path sharding 5 axis (a-e) 对比**:
- Sparse path 实际上是 (c) 层次路由 的简化版 — fan out to N shards + merge, 不需要复杂 routing layer
- No semantic partitioning needed (sparse 之间无 cluster overlap concern)
- Throughput scales linearly with shard count, no asymptotic complexity issue (vs dense ANN P × log(|X|/P))

### Hybrid pipeline 完整 architecture model（NEW）

[per topics/sparse-dense-hybrid-retrieval.md + formal-2021-splade-v2]

Production hybrid retrieval architecture = 3 独立 axis:

**Axis 1: Sparse path architecture**:
- Doc-ID sharded inverted index (BM25 / SPLADE-encoded)
- Simple fanout to all shards + merge

**Axis 2: Dense path architecture (per kusupati-2022-matryoshka post + radford-2021-clip)**:
- (a) Global single index (DistributedANN-style at large scale)
- (c) Hierarchical routing (Vespa SPANN / Pinecone slab / Milvus segment)
- (d) No-index tenant partition (Vespa Streaming)
- (e) Namespace-as-primitive (Turbopuffer)

**Axis 3: Fusion architecture**:
- Server-side rank-profile expression (Vespa)
- Server-side `hybrid(α)` (Weaviate)
- Server-side fusion 黑盒 (Pinecone)
- Client-side RRF / custom (Turbopuffer / Milvus)

### Vespa 是 wiki 内 3 axis 同时 first-class native 唯一系统

[per topics/sparse-dense-hybrid-retrieval.md + systems/vespa.md]

| Axis | Vespa native | Other vendor |
|---|---|---|
| Sparse path | ✓ BlockMaxWAND BM25 + SPLADE weightedset | partial |
| Dense path | ✓ tensor + SPANN + HNSW + CAGRA | varies |
| Fusion | ✓ rank-profile arbitrary expression + 4-phase ranking | varies |

→ **Vespa 唯一同时 3 axis first-class** — wiki 内 hybrid retrieval 架构 most complete vendor.

### 决策表（updated 2026-05-12 post formal-2021-splade-v2）

| 场景 | 推荐 |
|---|---|
| 千亿 hybrid + 复杂 ranking + ML rerank | **Vespa**: BM25 + SPLADE + SPANN + 4-phase ranking |
| 多租户 hybrid SaaS | **Turbopuffer**: namespace per tenant + multi_query + RRF |
| AI-native primary DB + hybrid | **Weaviate**: `hybrid(α)` API |
| Managed simplicity + hybrid | Pinecone Sparse-Dense Hybrid Index |
| 多 index_type + hybrid | Milvus 多 vector field |
| Pure dense 千亿+ | DistributedANN (no sparse path support per paper) |

### 已知盲区

- **Sparse path × DistributedANN single-graph dense 组合**: paper 不涵盖 sparse path; 是否 Bing internal 有 sparse path? unknown
- **Multi-axis architecture optimization**: 3 axis 独立选择 → 总 6×5×4 = 120 组合; production sweet spot 选择算法 不公开
- **Sparse path × spatial × multimodal**: Vespa technically capable 4 axis (sparse + dense + spatial + multimodal); production case 不存在公开
- **Hybrid fusion 选择 ergonomics**: Vespa rank-profile flexibility vs Weaviate hybrid(α) simplicity vs Turbopuffer application-fusion 各自 trade-off 系统化 ablation 不存在

## Cited Pages

- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
- [systems/vespa.md](../../systems/vespa.md)
