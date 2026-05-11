---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-11
phase: post
ingest-context: turbopuffer-docs
wiki-pages-total: 68
cited-pages: [systems/spann.md, systems/spfresh.md, systems/diskann.md, systems/turbopuffer.md, systems/vespa.md, systems/starling.md, topics/disk-vs-memory-ann.md, concepts/lire.md]
cited-count: 8
---

# Post-snapshot (turbopuffer-docs): memory-vs-disk-large-scale

## TL;DR (delta from vespa-docs post)

**Turbopuffer 把"memory vs disk"维度扩展到 "memory vs disk vs object storage"——wiki 内首次出现 object storage 作 primary storage 的 production system**。**关键 NEW**：之前所有方案（HNSW in-mem / SPANN centroid-mem+SSD-posting / DiskANN graph-mem-compressed+SSD-vector / Vespa Streaming disk-only）都假设 **local SSD 是底层**；Turbopuffer 第一次把 **S3 object storage 作 source of truth**——延迟 ~100ms/roundtrip 但 cost ~10× cheaper than NVMe-resident。SPFresh 选择不是偶然——是**唯一**适配 object-storage-primary 的 ANN（roundtrip-minimal centroid-based vs roundtrip-heavy graph-based）。

## Answer

### 与之前 ingest 的演进

| | vespa-docs post | **turbopuffer-docs post (NEW)** |
|---|---|---|
| Memory tier | HNSW + quantization | 不变 |
| Memory-only | HNSW production (4 OSS) | 不变 |
| SSD tier (mixed mem + SSD) | SPANN / DiskANN / Starling / FreshDiskANN / SPFresh research | **+ SPFresh production via Turbopuffer (OSS-known)** |
| Disk-only (no index) | Vespa Streaming | 不变 |
| **Object storage tier** | wiki 未列 | **+ Turbopuffer (first OSS-known)** |

### Turbopuffer 引入 Object Storage 作 primary tier（NEW）

[per sources/docs/turbopuffer/llms-full.txt §architecture §concepts §tradeoffs]

之前 wiki "memory vs disk" 维度只考虑 **local NVMe**——所有 disk-resident ANN (DiskANN / SPANN / Starling / FreshDiskANN / SPFresh paper) 假设 SSD/NVMe + SPDK 直接控制. Object storage 当 backup 或 archive.

Turbopuffer **倒置**：
- S3/GCS object storage = **source of truth**
- NVMe = **cache only**
- RAM = **hot cache only**
- All compute stateless

**经济性账目**:
- NVMe: ~$0.1/GB/month (provisioned)
- S3 standard: ~$0.023/GB/month (4× cheaper)
- S3 + intelligent tiering: ~$0.01/GB/month (10× cheaper)
- → Turbopuffer "10× cheaper than peer" claim 来自此

**Latency 代价**:
- NVMe: ~10-50µs per read
- S3 first byte: ~50-100ms per roundtrip
- → Cold query p50=343ms, p90=444ms 1M docs (Turbopuffer measured)
- → Warm query (NVMe hit) p50=8ms (peer DBMS 同级)

### 完整 memory vs disk vs object storage 全景表（updated 2026-05-11 post turbopuffer-docs）

| 方案 | 核心思路 | RAM | SSD/NVMe | Object Storage | Production 实证 |
|---|---|---|---|---|---|
| HNSW (memory) | 全图全数据全内存 | high | n/a | n/a | Milvus / Qdrant / Weaviate / Pinecone / Vespa OSS |
| HNSW + quantization | 同 + 4-32× compress | mid | n/a | n/a | Qdrant / Weaviate / Vespa |
| **SPANN** | centroid in-mem + posting on-disk | low (centroids) | high (posting lists) | n/a | Microsoft Bing + Vespa OSS |
| **DiskANN** | graph in-mem (compressed PQ) + raw vector on-disk | mid (compressed graph) | high | n/a | Microsoft + Milvus DISKANN |
| **Starling** | DiskANN 优化 segment layout | mid | high | n/a | Zilliz/Milvus future |
| **FreshDiskANN** | DiskANN + streaming insert/delete | mid | high | n/a | research / Milvus partial |
| **SPFresh (paper)** | SPANN + LIRE in-place update + SPDK raw SSD | low | high | n/a | research |
| **Vespa Streaming** | no index, brute-force per-user partition | very low (45 B/doc metadata) | scan | n/a | Vespa OSS |
| **Turbopuffer SPFresh** | **SPFresh + LSM + WAL all on object storage** | low (cache) | low (NVMe cache) | **high (primary)** | **Turbopuffer SaaS (frontier closed)** |

### 决策表（updated 2026-05-11 post turbopuffer-docs）

| Workload | 推荐方案 |
|---|---|
| **Cost-sensitive + multi-tenant + cold-latency-tolerant + 0 ops** | **Turbopuffer (object storage primary)** |
| Multi-tenant + per-tenant 数据小 | Vespa Streaming Search |
| Shared corpus 百亿规模 + 复杂 ranking pipeline | Vespa SPANN + 4-phase ranking |
| Shared corpus 百亿规模 + Milvus 生态 | Milvus DISKANN |
| Streaming insert + 百亿 disk-resident | FreshDiskANN / SPFresh path |
| 闭源 SaaS managed simplicity | Pinecone |
| Memory-only HNSW 顶到边 | + quantization (BQ / RQ / PQ) 延后 disk 切换 |

### Turbopuffer 选 SPFresh 的算法-存储匹配

[per Turbopuffer architecture docs argument]

为什么 graph-based (HNSW/DiskANN) 不适合 object storage primary:
- HNSW traversal 单 query ~log(N) 跳, 每跳读邻居 → 多 roundtrip × ~100ms
- DiskANN 同样需多个 SSD page reads
- Object storage primary 下: log(N) × 100ms 太慢

Centroid-based (SPFresh/SPANN-family) 适合:
- 1 read centroid index (~100ms)
- 1 batch fetch posting lists (~100ms)
- Total: ~3-4 roundtrips total ≈ 343ms cold

→ **Storage 假设 force 算法选择**。Turbopuffer 用 SPFresh 不是偏好，是 object storage primary 的逻辑结论。

### 已知盲区

- **Turbopuffer SPFresh 与 paper 实测对比**：闭源 fork 程度、LIRE NPA 在 object storage 下是否仍成立 unknown
- **Object storage 之外（B2/R2/MinIO/Azure Blob）的 SPFresh 实测**：仅 S3/GCS production
- **NVMe-resident SPFresh production case**：除 Turbopuffer，其他用 SPFresh 的 production case？无
- **多 disk path head-to-head**: Starling / FreshDiskANN / SPFresh / DiskANN 实测对比仍 zero
- **Turbopuffer cold query 上限**: p99 / p999 cold scenario docs 不给

## Cited Pages

- [systems/spann.md](../../systems/spann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/starling.md](../../systems/starling.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [concepts/lire.md](../../concepts/lire.md)
