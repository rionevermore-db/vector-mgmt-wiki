---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-11
phase: post
ingest-context: adams-2025-distributedann
wiki-pages-total: 69
cited-pages: [systems/distributedann.md, systems/spann.md, systems/turbopuffer.md, systems/vespa.md, systems/milvus.md, systems/pinecone.md]
cited-count: 6
---

# Post-snapshot (adams-2025-distributedann): index-architecture-global-vs-routed

## TL;DR (delta from turbopuffer-docs post)

**DistributedANN 让 (a) 全局单一索引在 wiki 内首次获得 production scale 实证**——之前 (a) 全局索引仅适合 ≤百亿 + 内存装得下场景 (HNSW Milvus single-segment), production 上限 ~1B。**DistributedANN 通过"single DiskANN graph + distributed KV store as shared disk" 把 (a) 单一索引 push 到 50B per slice production scale at Bing**——(a) 不再仅仅是"small-scale 简单 single graph"，而是 **logical-single-graph + physical-distributed 的混合形态**. **关键 NEW**: Bing **head-to-head 选 (a) 而非 (c)** based on Table 1 quantified data — single graph 在 6× throughput + 7.8pp recall + 7.5× less IO 上 outweigh (c) 的 latency 优势.

## Answer

### 与之前 ingest 的演进

| | turbopuffer-docs post | **adams-2025-distributedann post (NEW)** |
|---|---|---|
| (a) 全局索引 production scale | ≤百亿 (HNSW Milvus single-segment / Weaviate single-class) | **+ 50B per slice (DistributedANN @ Bing — logical-single-graph + physical-distributed)** |
| (c) 层次路由 production case | Pinecone slab + Milvus segment + SPANN @ Bing + SPANN @ Vespa + Turbopuffer namespace | **+ SPANN @ Bing 已被 DistributedANN 替换** (status historicized) |
| (d) 无索引 + tenant 分区 | Vespa Streaming | 不变 |
| (e) namespace-as-primitive | Turbopuffer | 不变 |
| **(a) 在 production main 主流** | 仅 ≤1B small scale | **NEW: Bing 100B+ production main is (a) logical single graph** |

### DistributedANN: (a) 全局单一索引在 production scale 实证（NEW）

[per adams-2025-distributedann §1-2]

**关键论点**：DistributedANN 把 (a) global-single-index 从 small-scale 推到 hundreds-of-billions production——通过 **logical/physical 分离**：
- **Logical**: 一个 DiskANN graph over 50B vectors (single index)
- **Physical**: graph nodes 分布在 1000+ KV store hosts (distributed)
- 抽象: KV store as shared disk

**Sublinear scaling 理论** (§1):
- Inside a single ANN index, query cost scales with **log|X|** (empirically measured)
- Across many partitions (fixed partition size), the cost will scale with the number of partitions
- A system with P partitions has search complexity **P log(|X|/P)** — much worse

50B 数据下:
- Single graph (DistributedANN): log(50B) ≈ 36 graph nodes traversed
- 203 partitions × log(247M) ≈ 203 × 28 = 5684 nodes if all partitions queried
- Top-40 partitions of 203 × 28 = 1120 nodes traversed
- **Single graph 比 partitioned 减 30× node visits**

### (a) vs (c) Bing head-to-head 选择（NEW）

[per adams-2025-distributedann Table 1]

| Architecture | Recall@5 | Latency p50 | Throughput | SSD | Memory | Bing choice |
|---|---|---|---|---|---|---|
| **(a) Single graph distributed (DistributedANN)** | **90.8%** | 26 ms | **>100K QPS** | 780 TiB | 42 TiB | **✓ chosen** |
| (c) Clustered partitioning (SPANN-style) | 83.0% | **16 ms** | ~15K QPS | **270 TiB** | **18 TiB** | × replaced |

**Bing rationale for choosing (a)**:
1. **6× throughput** at same hardware footprint
2. **+7.8pp recall@5 / +4.5pp recall@200** — quality 直接影响 search ranking
3. **7.5× less IO per query** — SSD IO 是瓶颈
4. **Sublinear scaling**: single graph log|X| 比 P × log(|X|/P) 增长慢
5. **Reliability**: node-level graceful degradation vs partition-level dramatic drop
6. **Load balancing**: random sharded KV store 比 semantic partition load 更均匀

**(a) 接受的 cost**:
- Latency p50 16→26 ms (1.6× slower)
- Storage 2.9× more SSD + 2.3× more memory
- Network BW 4.7× more per query
- 需要 distributed KV store infrastructure as底层

### 五种架构 (a)-(e) 全景对比表（updated 2026-05-11 post adams-2025）

| 架构 | 适用规模 | Partition 粒度 | Layer 数 | Production case |
|---|---|---|---|---|
| **(a) 全局单一索引** | **≤百亿 (memory-resident) OR ≥百亿 (DistributedANN logical-single + distributed-physical)** | n/a (single index) | 1-2 | HNSW Milvus / Weaviate single-class + **DistributedANN @ Bing 50B per slice** |
| (c) 层次路由 | 百亿-万亿 | content cluster group OR slab OR segment OR centroid | 2-5 | **Vespa SPANN (5), Pinecone slab (3), Milvus segment (3), SPANN @ Bing historical** |
| (d) 无索引 + tenant 分区 | per-tenant ≤百万 docs | per-tenant disk partition | 2 | Vespa Streaming |
| (e) namespace-as-primitive | per-namespace ≤500M docs / 全局任意大 | per-namespace S3 prefix (100M+) | 2 | Turbopuffer SPFresh |

### 千亿规模主流变化（NEW）

Tier 5 (≥100B) 之前：
- (c) 路由架构是 production 主流（Pinecone slab / Vespa SPANN / Milvus segment / Bing SPANN）
- (a) 不可行（single graph 装不下单机）

Tier 5 之后（DistributedANN 后）：
- **(a) DistributedANN single-graph distributed 是 Bing production 当前主流**
- (c) Vespa OSS SPANN 仍 active production path
- 两者**并存且 axis 相反**：(a) sublinear scaling + 6× throughput vs (c) faster latency + lower storage

→ **Bing 选 (a)** 因 throughput + recall 优先；**Vespa SPANN 仍选 (c)** 因 OSS path + latency 优先 workload.

### Single-graph distributed 的工程门槛（NEW）

[per adams-2025-distributedann §2]

3 关键 design 让 (a) viable at production scale:
1. **Compressed vectors duplicated into graph nodes** — 10× storage 换 1 IO/hop
2. **In-memory head index** — top-layers sharded ANN 作 beam search 起点
3. **Near-data computation** — node scoring service on each KV host (~6× bandwidth savings)

→ 没有 distributed KV store 基础设施的 vendor 无法直接做 (a) 路径；这是 Bing internal infrastructure 优势.

### 已知盲区

- **(a) DistributedANN OSS 实现**: 未公开; 复现需类似 KV store
- **(a) DistributedANN 在 OSS vector DBMS 内的等价物**: Milvus / Qdrant / Weaviate / Vespa 当前均无 single-graph-distributed
- **(a) vs (c) 在不同 workload 的 crossover**: Bing 接受 latency penalty; latency-critical workload 是否 (c) 仍更好? Vespa 选择暗示 yes
- **(a) multi-slice routing**: Bing hundreds of billions = multiple 50B slices; 跨 slice query 协调论文不深入

## Cited Pages

- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/pinecone.md](../../systems/pinecone.md)
