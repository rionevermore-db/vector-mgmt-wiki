---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-11
phase: post
ingest-context: adams-2025-distributedann
wiki-pages-total: 69
cited-pages: [systems/distributedann.md, systems/spann.md, systems/diskann.md, systems/turbopuffer.md, systems/vespa.md, systems/milvus.md]
cited-count: 6
---

# Post-snapshot (adams-2025-distributedann): giga-scale-sharding

## TL;DR (delta from turbopuffer-docs post)

**DistributedANN 引入 wiki 内一个全新 sharding 哲学**——**single graph across thousands of machines via distributed KV store**，与之前所有 sharding 路径（SPANN partition / Pinecone slab / Milvus segment / Turbopuffer namespace）哲学根本不同。**关键 NEW**：Bing 直接 quantified vs 之前 production architecture (SPANN-style clustered partitioning) head-to-head——**6× throughput at same machine footprint**，证明 single-graph-distributed 在最大 production scale 击败 partitioning。**Production 16-node + 1TB RAM 假设**下 DistributedANN 提供精确 SKU 参考: 256-768 GiB RAM / 5-10 TiB SSD / 32-64 cores / 40 Gbps network / 200 IOPS/GiB — Bing 同级别 host SKU. **Sublinear scaling 理论**: log(|X|) vs partition P × log(|X|/P), 50B 数据 ≈ 36 nodes traversed (DistributedANN) vs 1120 (203 partition × 28 nodes if top-40 selected).

## Answer

### 与之前 ingest 的演进

| | turbopuffer-docs post | **adams-2025-distributedann post (NEW)** |
|---|---|---|
| 千亿规模 sharding 路径 | (a) SPANN content cluster / (b) Vespa Streaming per-tenant / (c) Turbopuffer namespace-fanout | **+ (d) DistributedANN single-graph distributed via KV store** |
| Bing production architecture | 隐含 SPANN (2021) | **明示 DistributedANN (2025) — SPANN 历史化** |
| Single-vendor 最高 scale claim | Turbopuffer 3.5T+ docs (commercial SaaS) | **+ Bing hundreds of billions (DistributedANN OSS-paper-documented)** |
| Distributed graph algorithm | wiki 未涵盖 | **NEW: single Vamana graph across 1000+ machines** |

### DistributedANN single-graph-distributed 拓扑（NEW）

[per adams-2025-distributedann §1-2]

**核心论点**：
- Inside a single ANN index, query cost scales with log|X|
- Across many partitions (fixed partition size), the cost will scale with the number of partitions
- A system with P partitions has search complexity **P log(|X|/P)** — much worse than log(|X|) of single index over X

**Single-graph-distributed 实现**:
- Logical: one DiskANN graph over 50B vectors
- Physical: graph nodes distributed across thousand+ KV store hosts
- 抽象: **KV store as shared disk** ("DISTRIBUTEDANN begins with the abstraction of the key-value store as a large shared disk")
- 3 关键改造让 distributed DiskANN viable:
  1. Compressed vectors duplicated into graph nodes (10× space amp, 1 IO/hop)
  2. In-memory head index (top-layers as separately sharded ANN, beam search starting points)
  3. Near-data computation (node scoring on each KV host)

### Bing production 配置 (DistributedANN production SKU)

[per adams-2025-distributedann §4]

| Parameter | Value |
|---|---|
| Vector type | 384-d int8 |
| Slice size | 50B vectors |
| Slices per index | "hundreds of billions" total |
| Host RAM | 256-768 GiB |
| Host SSD | 5-10 TiB |
| Host IOPS | 200 IOPS/GiB |
| Host cores | 32-64 |
| Host network | 40 Gbps |
| Replicas | 3 |
| H (graph hops) | 5 |
| BW (beam width) | 128 |
| R (graph degree) | 72 (truncated from DiskANN 100) |
| Head index size | 2.5B vectors |
| Query p50 | 26 ms |
| Query p99 | 35 ms |
| QPS | >100K |
| Recall@5 | 90.8% |

### 16 节点 + 1TB RAM 假设下的 DistributedANN 推断

**Per-node 容量**:
- Memory 1 TB ≈ Bing 上限 (768 GiB) 范围
- 16 节点 × 1 TB = 16 TB total RAM —— 比 Bing 同 footprint (~42 TiB for 50B + 3 replicas) 小 2.6×
- 推算 max scale: ~10-15B vectors (768-d float, single replica)

**配置选择**:
- 不适合 50B-per-slice (Bing scale, too much hardware)
- 适合 5-15B single distributed graph
- vs SPANN partition 路径 (16 nodes × ~1B each partition): 大致同 viable, 但 DistributedANN 6× throughput

### 千亿/万亿决策表（updated 2026-05-11 post adams-2025）

| Workload | 推荐方案 |
|---|---|
| **千亿 + 单一巨大 shared corpus + 6× throughput 优先 + 闭源接受** | **DistributedANN (Bing 当前 production)** |
| **千亿 + 单一巨大 shared corpus + p99 < 20ms 强制** | SPANN-style clustered partitioning (Vespa OSS, faster latency) |
| 多租户极致 (1M+ tenant 独立) | Turbopuffer namespace-as-tenant |
| Multi-tenant + per-tenant 数据小 (personal AI) | Vespa Streaming Search |
| Shared corpus 百亿 + 复杂 ranking + ML rerank | Vespa SPANN + 4-phase ranking |
| 千亿 + 多 index_type | Milvus DISKANN |
| Billion-scale + 闭源 SaaS managed | Pinecone slab |

### Sharding 5 路径全景（updated 2026-05-11 post adams-2025）

| 路径 | 代表系统 | Production scale 上限 | Partition 数量级 |
|---|---|---|---|
| (a) Global single index | HNSW Milvus single-segment | ~1B | n/a |
| (b) Cluster + partition routing | SPANN @ Vespa, Pinecone slab, Milvus segment | **Bing 历史 hundreds of billions (SPANN)** | 数十-数千 |
| (c) No-index + tenant brute-force | Vespa Streaming | per-tenant 1M | 数百万 |
| (d) Namespace-as-primitive | Turbopuffer | 3.5T+ docs / 100M+ namespaces | 1亿+ |
| **(e) Single-graph distributed via KV store** | **DistributedANN @ Bing** | **50B per slice × multiple slices** (current production) | thousands of machines |

### 已知盲区

- **DistributedANN 16-node deployment 实测**: Bing 是 1000+ machines; 16-node 是否 viable 论文不讨论
- **Single-graph distributed 跨 region**: Bing inter-zone latency 2 ms, cross-rack BW 过载; 论文 §5.1 提 "dense cluster + fully connected network" 为 future
- **DistributedANN OSS implementations**: 当前 closed-source; 复现需类似 KV store
- **DistributedANN vs SPANN cost-per-query**: Table 1 给 hardware footprint same 但成本未量化（SSD price × 270 vs 780 TiB 是真实 cost delta）

## Cited Pages

- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/milvus.md](../../systems/milvus.md)
