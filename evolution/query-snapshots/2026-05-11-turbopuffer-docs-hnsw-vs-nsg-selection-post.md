---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-11
phase: post
ingest-context: turbopuffer-docs
wiki-pages-total: 68
cited-pages: [concepts/hnsw.md, concepts/nsg.md, systems/turbopuffer.md, systems/spfresh.md, systems/vespa.md]
cited-count: 5
---

# Post-snapshot (turbopuffer-docs): hnsw-vs-nsg-selection

## TL;DR (delta from vespa-docs post)

**Turbopuffer 引入一个 wiki 内全新视角**：**graph-based ANN 不适合 object-storage 主存**——Turbopuffer 是 wiki 内**第一个公开声明"不选 HNSW/DiskANN 因为 graph traversal × 100ms object storage roundtrip 太慢"**的 production system。改用 [SPFresh](../../systems/spfresh.md) (centroid-based)——HNSW/NSG **的 production deployment 数仍为 4 OSS DBMS**（Milvus + Qdrant + Weaviate + Vespa），但 **storage 假设决定算法选择**这一维度首次明示。NSG production 仍零。

## Answer

### 与之前 ingest 的演进

| | vespa-docs post | **turbopuffer-docs post (NEW)** |
|---|---|---|
| HNSW production OSS DBMS | 4 (Milvus + Qdrant + Weaviate + Vespa) | **不变 4**——Turbopuffer 选 SPFresh 不选 HNSW |
| NSG production | 仍仅 Taobao 2B | **不变** |
| HNSW vs alternative ANN 选型驱动 | mostly **filter / streaming / GPU 能力** | **+ storage 假设（NVMe vs object storage）** |
| Object-storage-friendly ANN | wiki 未明示 | **Turbopuffer: SPFresh > HNSW/DiskANN 因 roundtrip 经济性** |

### Turbopuffer 明示"为什么不选 HNSW"（NEW）

[per sources/docs/turbopuffer/llms-full.txt §architecture §concepts]

> Vector indexes are based on SPFresh. **A centroid-based index works well for object storage as it minimizes roundtrips and write-amplification, compared to graph-based indexes like HNSW or DiskANN.**

→ 这是 wiki 内**第一个明示"graph-based ANN 不适合 object storage primary"**的 production system。逻辑链：
- HNSW traversal 单 query ~log(N) 跳，每跳读邻居 → 多 roundtrip
- Object storage roundtrip ~100ms（vs NVMe ~10µs）
- **Graph-based × 100ms × log(N) → cold query 太慢**
- Centroid-based (SPFresh): 1 roundtrip 读 centroid index + 1 batch fetch posting list → minimal roundtrip
- **Storage 假设决定算法选择**——HNSW 假设 RAM/NVMe low-latency；object storage primary 直接 force 选 centroid

### HNSW vs NSG vs SPFresh 选择决策表（updated）

| 工程考量 | HNSW | NSG | SPFresh |
|---|---|---|---|
| Production OSS DBMS | **4: Milvus / Qdrant / Weaviate / Vespa** | NSSG + Taobao | **+ Turbopuffer (commercial SaaS, OSS-known first)** |
| Storage 假设 | RAM / NVMe low-latency | RAM low-latency | **Object storage primary friendly** |
| Filter-aware build | ✓ via ACORN / Filterable HNSW | ✗ | LIRE protocol + namespace partition |
| Streaming insert+delete | HNSW α=1 fail → patch (HFresh / FreshVamana) | ✗ | **first-class via LIRE protocol** |
| GPU-native | ✓ via CAGRA | ✗ | ✗ |
| Cold query 友好 | ✗ (graph traversal × roundtrip) | ✗ | **✓ (centroid + batch fetch)** |
| Production scale 最高 claim | Bing-via-SPANN 1.5T (闭源) | n/a | **Turbopuffer 3.5T+ docs / 100B+ queryable (OSS-known)** |

### 选择决策（updated 2026-05-11 post turbopuffer-docs）

- **Object storage primary + cost-sensitive + multi-tenant (100M+ namespaces)** → **SPFresh via Turbopuffer**（不是 HNSW！）
- **OSS vector DBMS NVMe + low-latency** → **HNSW** (4 OSS DBMS 全部选)
- **OSS Rust + 简洁 + filter heavy** → Qdrant
- **OSS Go + AI-native primary DB + agent stack** → Weaviate
- **OSS Go + 多 index_type + cloud-native** → Milvus
- **复杂 ranking pipeline + tensor framework + ML rerank** → Vespa
- **闭源 SaaS managed simplicity** → Pinecone
- **Streaming graph** → Vamana base (FreshDiskANN) OR HNSW base (Weaviate HFresh)
- **CPU + 简单 single-vector TopK + million scale + 静态** → NSG (学术 benchmark only)

NSG 仍无 production OSS DBMS；HNSW 是 RAM/NVMe-based production 主流；**Turbopuffer 首次明示"object storage primary → centroid (SPFresh) over graph (HNSW)"** ——选择决策维度首次包含 storage 主从假设。

### 已知盲区

- **NSG-based 主流 production DBMS**：仍未出现
- **HNSW vs SPFresh head-to-head on object storage**：Turbopuffer 仅声明 SPFresh 更优，未给 head-to-head 数据
- **Turbopuffer SPFresh 与 paper 实现差异**：闭源 Rust port + object storage adaptation 具体 fork 程度不公开
- **Object storage 之外 storage（B2/R2/MinIO）下 SPFresh 是否同效**：Turbopuffer docs 仅 S3/GCS 实测

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/vespa.md](../../systems/vespa.md)
