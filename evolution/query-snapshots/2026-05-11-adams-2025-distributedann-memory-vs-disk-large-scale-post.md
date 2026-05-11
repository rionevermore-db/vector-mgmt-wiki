---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-11
phase: post
ingest-context: adams-2025-distributedann
wiki-pages-total: 69
cited-pages: [systems/distributedann.md, systems/spann.md, systems/diskann.md, systems/spfresh.md, systems/turbopuffer.md, systems/starling.md, topics/disk-vs-memory-ann.md]
cited-count: 7
---

# Post-snapshot (adams-2025-distributedann): memory-vs-disk-large-scale

## TL;DR (delta from turbopuffer-docs post)

**DistributedANN 引入 wiki 内全新"distributed KV store as shared disk"路径**——之前 disk path 全是 local SSD (DiskANN/SPANN/Starling/FreshDiskANN/SPFresh paper) + object storage (Turbopuffer)。DistributedANN 用 **distributed KV store 作 shared-disk 抽象**——key-value backend 是 source of truth, in-memory caches 服务 hot data, **数据 layout 由 application 控制** (compressed vectors duplicated into graph nodes is a deliberate layout choice). Bing 已 production-validated, 4 axis breakdown:
- Local NVMe (DiskANN/SPANN/Starling): low latency, single-host bound
- Object storage primary (Turbopuffer): cheap, cold latency存在
- **Distributed KV store (DistributedANN)**: medium latency (26ms p50), 6× throughput vs partition

## Answer

### 与之前 ingest 的演进

| | turbopuffer-docs post | **adams-2025-distributedann post (NEW)** |
|---|---|---|
| 4 disk tier (memory / local NVMe / object storage / brute-force) | unchanged | **+ 5th tier: distributed KV store as shared disk** |
| Bing production disk path | SPANN-style inverted file on SSD | **DistributedANN: distributed KV store + node-co-located scoring** |
| Object storage primary vs distributed KV | only Turbopuffer (object storage) | **+ DistributedANN (KV store) — 两类 shared backend 哲学** |

### DistributedANN distributed KV store 路径（NEW）

[per adams-2025-distributedann §2]

> DISTRIBUTEDANN begins with the abstraction of the **key-value store as a large shared disk**, and then makes modifications to the data and compute placement choices of DISKANN indices in order to make it practical to serve them in this setting.

**为什么 distributed KV store, 不是 object storage**:
- Object storage roundtrip ~100ms (cold) — 太慢 for 26ms p50 target
- Distributed KV store 在 datacenter network 单 hop ~ms 级 + data is hot in distributed cache
- KV store 支持 fine-grained key-based access (vs object storage 是 file-based)
- 但 KV store 是**底层基础设施依赖**——需 Cosmos DB 或类似 internal system

**3 关键改造让 distributed KV store + DiskANN viable**:
1. **Compressed vectors duplicated into graph nodes** (10× space amp, 1 IO/hop)
2. **In-memory head index** (sharded, 2.5B vectors, beam search starting points)
3. **Near-data computation** (node scoring on each KV host, ~6× bandwidth savings)

### 完整 memory vs disk vs distributed-backend 全景表（updated 2026-05-11 post adams-2025）

| 方案 | 核心思路 | RAM | Local NVMe | Object Storage | Distributed KV | Production 实证 |
|---|---|---|---|---|---|---|
| HNSW (memory) | 全图全数据全内存 | high | n/a | n/a | n/a | Milvus / Qdrant / Weaviate / Pinecone / Vespa OSS |
| HNSW + quantization | 同 + 4-32× compress | mid | n/a | n/a | n/a | Qdrant / Weaviate / Vespa |
| SPANN | centroid in-mem + posting on-disk | low (centroids) | high (posting lists) | n/a | n/a | Microsoft Bing **2021-2024** + Vespa OSS active |
| DiskANN | graph in-mem (compressed PQ) + raw vector on-disk | mid (compressed graph) | high | n/a | n/a | Microsoft + Milvus DISKANN |
| Starling | DiskANN 优化 segment layout | mid | high | n/a | n/a | Zilliz/Milvus future |
| FreshDiskANN | DiskANN + streaming insert/delete | mid | high | n/a | n/a | research / Milvus partial |
| SPFresh (paper) | SPANN + LIRE + SPDK raw SSD | low | high | n/a | n/a | research |
| Vespa Streaming | no index, brute-force per-user partition | very low (45 B/doc metadata) | scan | n/a | n/a | Vespa OSS |
| Turbopuffer SPFresh | SPFresh + LSM + WAL all on object storage | low (cache) | low (NVMe cache) | **high (primary)** | n/a | Turbopuffer SaaS |
| **DistributedANN (NEW)** | **DiskANN graph + duplicated compressed vectors in graph nodes + node-co-located scoring** | mid (head index + caches) | mid (per host) | n/a | **high (primary)** | **Microsoft Bing 2025 当前 production** |

### Disk 路径决策表（updated 2026-05-11 post adams-2025）

| Workload | 推荐方案 |
|---|---|
| **Single graph distributed + 6× throughput + Microsoft-like infra** | **DistributedANN (KV store as shared disk)** |
| Cost-sensitive + multi-tenant + cold-latency-tolerant | Turbopuffer (object storage primary) |
| Multi-tenant + per-tenant 数据小 | Vespa Streaming Search |
| Shared corpus 百亿规模 + 复杂 ranking pipeline | Vespa SPANN + 4-phase ranking |
| Shared corpus 百亿规模 + Milvus 生态 | Milvus DISKANN |
| Streaming insert + 百亿 disk-resident | FreshDiskANN / SPFresh path |
| 闭源 SaaS managed simplicity | Pinecone |
| Memory-only HNSW 顶到边 | + quantization (BQ / RQ / PQ) 延后 disk 切换 |

### "Shared backend" 哲学的两种实现（NEW comparison）

[Turbopuffer (object storage) vs DistributedANN (distributed KV store)]

| | Turbopuffer | DistributedANN |
|---|---|---|
| Shared backend type | **Object storage (S3/GCS)** | **Distributed KV store** |
| ANN algorithm | SPFresh (centroid-based) | DiskANN (graph-based) — Vamana |
| Latency p50 (1M docs) | 8ms warm / 343ms cold | **26ms (always — uniform)** |
| Cost claim | ~10× cheaper than NVMe-resident | "same machine footprint" vs partition (6× throughput more efficient) |
| Vendor model | Closed SaaS + BYOC | Closed (Bing internal) |
| Data layout control | LSM on object storage | **Compressed vectors duplicated into graph nodes (custom)** |
| Latency 来源 | Object storage GET roundtrip ~100ms × 3-4 | KV store 单 hop ~ms × 5 hops |
| Production scale | 3.5T+ docs / 100M+ namespaces | hundreds of billions per Bing index (50B per slice) |

→ **两种 "shared backend" 哲学的关键差异**：
- **Object storage path**: 选 cheap durable storage 作 source of truth; cold latency 不可消除; 适合 cost-sensitive multi-tenant
- **Distributed KV store path**: 选 fast distributed cache + custom layout 作 source of truth; latency uniform; 适合 throughput-critical single corpus

### 已知盲区

- **DistributedANN KV store 具体实现**: Microsoft Cosmos DB? 内部自研? 论文不公开
- **Object storage path vs distributed KV path cost**: per query / per GB / per QPS 对比未量化
- **Multi-region active-active**: Bing DistributedANN 是 inter-zone within region; 跨 region active-active 不讨论
- **Hybrid object-storage + KV-store**: 两种 shared backend 是否可叠加? wiki / paper 都未涵盖

## Cited Pages

- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/starling.md](../../systems/starling.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
