---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-11
phase: post
ingest-context: vespa-docs
wiki-pages-total: 67
cited-pages: [systems/spann.md, systems/vespa.md, systems/milvus.md, systems/pinecone.md, concepts/pinecone-pod-based.md, topics/disk-vs-memory-ann.md]
cited-count: 6
---

# Post-snapshot (vespa-docs): index-architecture-global-vs-routed

## TL;DR (delta from weaviate-docs post)

**Vespa 提供 wiki 内一个**特殊路由架构变种**：**stateless container / stateful content cluster 双层路由 + 内嵌 SPANN centroid posting**——这是 wiki 内首次出现"**两级 routing**"明示：第一级 container → content node (group selection)，第二级 SPANN centroid → posting list (data selection)。**关键 NEW**：Vespa **Streaming Search 是第三种正交架构**——不是 (a) 全局索引也不是 (c) 层次路由，而是 **(d) "无索引 + tenant-based 物理分区"**——所有 doc 按 tenant 分到 disk partition，query 直 scan 该 partition。这扩展了原 (a)/(c) 二分。

## Answer

### 与之前 ingest 的演进

| | weaviate-docs post | **vespa-docs post (NEW)** |
|---|---|---|
| (a) 全局索引 production case | 仅小规模 HNSW (Milvus single-segment, Weaviate single-class) | **不变** |
| (c) 路由架构 production case | Pinecone pod-based + slab; Milvus segment; SPANN (Bing) | **+ Vespa SPANN + Vespa group routing 双层** |
| 双层 routing 明示 | Pinecone slab 隐式 | **Vespa: container → content cluster group → SPANN posting (3 层)** |
| (d) 无索引 + 物理分区 (新) | 未提 | **Vespa Streaming Search — tenant-based disk partition** |

### Vespa "三层 routing" 拓扑（NEW）

[per sources/docs/vespa/llms-full.txt §Approximate Nn Hnsw §Billion Scale]

Vespa 千亿规模典型架构：

```
Query →
  Layer 1: Stateless container (query parsing, YQL planning, embedding inference)
    → Layer 2: Content cluster (group + node selection — load balancing across groups)
      → Layer 3 (if SPANN): centroid index → posting list selection
        → disk read of posting list → distance compute → return top-K
        → Layer 4 (ranking): first-phase + second-phase per-shard rerank
    → Layer 5: Container global-phase rerank (ONNX cross-encoder)
```

**5 层 fanout/merge**——之前 wiki 内系统最多 3 层（SPANN centroid + Milvus segment routing 是相同概念）。Vespa 把 routing 拆得最细：query path / data path 严格分离（Layer 1 stateless vs Layer 2 stateful）+ 路由内嵌 SPANN centroid (Layer 3) + 多阶段 ranking (Layer 4-5)。

### Vespa Streaming Search 是 (d) 新 category

之前架构二分 (a) global vs (c) routed 都假设需要 ANN index。Streaming Search 引入 (d) **无 ANN, tenant-based 物理分区, brute-force scan**：

```
Query (with tenant=user_42) →
  Container: tenant 路由 (consistent hash on user_42)
    → Content node holding user_42 partition
      → Disk scan partition (45 B/doc metadata + raw vector)
      → Distance compute on all docs in partition (~100K-1M)
      → Return top-K
```

→ **物理分区 = 索引**：no separate index structure。每 tenant 的数据按 tenant_id 物理隔离到 disk partition。**适用场景**：personal AI assistant、企业内 per-user namespace、单 user query 数据量小 (≤1M docs)。

### 架构选型决策表（updated 2026-05-11 post vespa-docs）

| 架构 | 适用规模 | 适用 workload | Production case |
|---|---|---|---|
| (a) 全局单一索引 | ≤百亿 + 内存装得下 | shared corpus + 简单 query | HNSW + Milvus single-segment / Weaviate single-class |
| (c) 层次路由 | 百亿-万亿 | shared corpus + 复杂 query / ranking | **Vespa SPANN + group routing**, Pinecone slab, Milvus segment, SPANN (Bing) |
| **(d) 无索引 + tenant 分区** | per-tenant ≤百万 docs, 总 corpus 可亿/万亿 | multi-tenant + per-tenant query 隔离 | **Vespa Streaming Search** (独占) |

### 千亿规模主流仍是 (c)，但 Vespa 拆得最细

(c) 层次路由仍是 shared-corpus 千亿规模主流。Vespa 是 wiki 内**对路由拆得最细**的系统（5 层 fanout/merge）——其他系统都是 2-3 层。这种细分对应 web search engine heritage：Yahoo! 2003 起就要分离 query path / data path / ranking pipeline。

**未来 wiki 需要更进一步明示的**：
- (c) 内部 sub-variant：centroid routing (SPANN) vs segment routing (Milvus) vs slab routing (Pinecone) 的算法差异
- (d) tenant partition 的 cross-tenant query 处理（Vespa Streaming Search 跨 tenant 怎么 fanout？）

### 已知盲区

- **Vespa 5 层 fanout 的 P99 latency budget 分配**：实测每层占多少 ms？docs 未量化
- **(c) sub-variants head-to-head**：SPANN centroid vs Milvus segment vs Pinecone slab 算法对比
- **(d) Streaming Search 跨 tenant fanout**：when query 涉及多 user，是 N 个 sequential disk scan 还是并发？docs 不详
- **Vespa Streaming + SPANN 同集群混部**：是否一 application package 内可同时部署，docs 未明示

## Cited Pages

- [systems/spann.md](../../systems/spann.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/pinecone.md](../../systems/pinecone.md)
- [concepts/pinecone-pod-based.md](../../concepts/pinecone-pod-based.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
