---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-11
phase: post
ingest-context: vespa-docs
wiki-pages-total: 67
cited-pages: [systems/spann.md, systems/diskann.md, systems/vespa.md, systems/milvus.md, systems/pinecone.md, systems/qdrant.md, concepts/pinecone-pod-based.md, topics/disk-vs-memory-ann.md]
cited-count: 8
---

# Post-snapshot (vespa-docs): giga-scale-sharding

## TL;DR (delta from weaviate-docs post)

**Vespa 提供两个新的 giga-scale 视角**：(1) [SPANN](../../systems/spann.md) **第二个 production deployment**——Vespa OSS 实现 SPANN 作为 billion-scale option（之前 wiki 仅 Microsoft Bing 闭源 production）；(2) **Stateless container / Stateful content cluster 分离**——giga-scale sharding 的拓扑级抽象，content cluster 分 group + node，每 group 全 dataset、group 间负载均衡，container 无状态可水平扩展接 query。**关键 NEW** 之 (3)：Vespa **Streaming Search** 是 giga-scale 的**反方向方案**——不建 index、纯 brute-force per-user partition、45 bytes/doc 内存使用、 billion docs/node——对 multi-tenant + per-tenant 数据量小（个人 email / chat history）的 giga-scale workload **比 SPANN 还更经济**。

## Answer

### 与之前 ingest 的演进

| | weaviate-docs post | **vespa-docs post (NEW)** |
|---|---|---|
| SPANN production case | Microsoft Bing (闭源 only) | **+ Vespa OSS (2nd) — frontier closed** |
| Giga-scale 拓扑抽象 | Milvus segment + coord; Pinecone slab | **+ Vespa stateless container / stateful content cluster (group + node)** |
| 千亿 scale non-graph 方案 | DiskANN + SPANN (path "build big graph but page from disk") | **+ Vespa Streaming Search (no-graph, no-index brute-force per-user)** |

### Vespa SPANN production（NEW）

[per sources/docs/vespa/llms-full.txt §Billion Scale Vector Search]

> Vespa supports SPANN-style indexing for billion-scale collections, with both the centroid index in memory and the posting lists on disk.

→ **wiki 内第二个 SPANN production deployment**——之前唯一是 Microsoft Bing（闭源），现在 OSS Vespa 是公开可验证的 SPANN 实现。SPANN production frontier 至此**正式闭合**（≥2 个 deployment）。

**16 节点 + 1TB RAM 假设下的 Vespa SPANN allocation**：
- Memory budget per node: 1 TB
- Centroids（in-memory）：~100M centroids × 768 × 4 bytes = ~300 GB → fits
- Posting lists（on-disk）：~1T vectors × 768 × 4 bytes / 16 nodes = ~3 TB/node disk
- Query: container 路由到 centroids → 选 top-K closest centroids → 各 content node 读对应 posting lists

### Vespa Streaming Search（NEW 反方向）

[per sources/docs/vespa/llms-full.txt §Streaming Search]

> **45 bytes per document**, can handle **billions of documents per node** — when each query touches only a small subset (e.g. one user's emails).

→ **完全不建 index**。每 query 走完整 disk scan 但仅扫某 user 的 partition——典型 multi-tenant 场景（personal email/chat/notes）。**16 节点 + 1TB RAM 假设下 Streaming Search**：
- 每 doc 45 bytes：1T docs × 45 / 16 = ~3 TB/node disk
- 每 query 仅扫 1 user 的 partition (~K docs)，**无需 P99 加速结构**
- 内存仅用作 disk cache + metadata，RAM 浪费在 ANN index 反而是浪费

→ "千亿规模下 P99 < 50ms" 在 Streaming Search 假设里是 **per-user partition 大小 × 45 bytes / disk bandwidth** ——只要 per-user 是 100K-1M docs scale，**brute-force 比 SPANN 还简洁可靠**。

### 千亿/万亿决策表（updated 2026-05-11 post vespa-docs）

| Workload | 推荐方案 |
|---|---|
| Multi-tenant + per-tenant 数据小（个人助手 / RAG over personal data） | **Vespa Streaming Search** |
| Shared corpus 千亿规模 + 复杂 ranking pipeline + ML rerank | **Vespa SPANN + 4-phase ranking** |
| 千亿规模 + 简洁 OSS Rust 部署 | Qdrant + DiskANN-style on-disk HNSW (但 docs 不详) |
| 千亿规模 + 多 index_type 选择空间 | Milvus DISKANN / SPARSE_INVERTED_INDEX |
| Billion-scale + 闭源 SaaS 简单 | Pinecone slab |
| 单一 graph 索引 + RAM 富余 | HNSW (memory-only) |

### Vespa stateless / stateful 分离的工程价值

Vespa 16 节点典型部署可拆为：
- **2-4 stateless container nodes**：处理 query parsing, YQL planning, embedding inference, global-phase ONNX rerank——**水平扩展独立于数据**
- **12-14 stateful content nodes**：持有 index + 数据，按 group 分片，group 内每 node 持有 partition 子集

→ 这种拆分是 web search engine heritage（Yahoo! 2003 起 query path / data path 分离）——Milvus 在 v2.6.x cloud-native 重写后才类似（streaming node / query node / data node），Vespa **20 年前就是这套**。

### 已知盲区

- **Vespa SPANN 实测 billion-scale latency 公开数字**：docs 描述机制但无具体表
- **Streaming Search vs SPANN crossover threshold**：per-user partition 大小阈值 docs 未给
- **16 节点 + 1TB RAM 假设下 Vespa 实测**：未公开
- **Vespa group 拓扑下 weekly full refresh cost**：千亿规模 weekly 重建 application package deploy 时间 unknown

## Cited Pages

- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/pinecone.md](../../systems/pinecone.md)
- [systems/qdrant.md](../../systems/qdrant.md)
- [concepts/pinecone-pod-based.md](../../concepts/pinecone-pod-based.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
