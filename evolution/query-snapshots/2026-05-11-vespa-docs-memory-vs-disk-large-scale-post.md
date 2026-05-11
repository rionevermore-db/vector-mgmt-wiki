---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-11
phase: post
ingest-context: vespa-docs
wiki-pages-total: 67
cited-pages: [systems/spann.md, systems/diskann.md, systems/vespa.md, systems/starling.md, systems/freshdiskann.md, systems/spfresh.md, topics/disk-vs-memory-ann.md]
cited-count: 7
---

# Post-snapshot (vespa-docs): memory-vs-disk-large-scale

## TL;DR (delta from weaviate-docs post)

**Vespa 闭合两个 frontier**：(1) **SPANN OSS production** — 之前 wiki 内 SPANN 唯一 production 是 Microsoft Bing 闭源, Vespa Apache-2.0 OSS 是**第二个独立 production deployment**, SPANN 不再仅"论文 + Bing 一份 ground truth"; (2) **完全无 index brute-force disk scan tier (Streaming Search)** — 45 bytes/doc 内存 + disk-only scan, billion docs/node — 是 wiki 内**首个"反 ANN"哲学的 production 系统**, 与 SPANN/DiskANN 的"建索引以减搜索"路线**正交**。

## Answer

### 与之前 ingest 的演进

| | weaviate-docs post | **vespa-docs post (NEW)** |
|---|---|---|
| SPANN production case | Microsoft Bing only (闭源) | **+ Vespa OSS (2nd, Apache-2.0) — production frontier closed** |
| 反 ANN 哲学 production | wiki 未提 | **+ Vespa Streaming Search (no index, brute-force per-user partition)** |
| Disk-vector 路径数 | 3 (SPANN + DiskANN + Starling) | **4 (+ Streaming Search 反方向)** |

### Vespa SPANN（NEW）

[per sources/docs/vespa/llms-full.txt §Billion Scale Vector Search]

> SPANN-style indexing for billion-scale collections, with both the centroid index in memory and the posting lists on disk.

→ Vespa OSS 是 wiki 内**第二个 SPANN production deployment**（独立于 Microsoft Bing）。SPANN 论文 [chen-2021-spann] 提出的 "RAM centroids + SSD posting lists" 架构现在有**两个**独立 production 实证，标志 SPANN 从"论文 + 一个内部"变为"production-validated default for billion-scale"。

### Vespa Streaming Search 完全反方向（NEW）

[per sources/docs/vespa/llms-full.txt §Streaming Search]

> 45 bytes per document — billion documents per node — when each query touches only a small subset

**核心哲学反转**：
- ANN 路线（HNSW / SPANN / DiskANN）：build index → reduce search space → fast query
- Streaming Search：**不 build index**, query 直接 disk scan 但仅扫**很小子集**（per-user partition）

**为什么 disk-only 行得通**：
- 每 doc 仅 45 bytes 元数据 + raw vector 在 disk (NVMe)
- Per-user partition 通常 100K-1M docs
- 单 query 扫该 partition：disk bandwidth × partition size / 45 = 几 ms
- 无 index build / update / memory 开销

**适用 workload**：personal AI assistant、企业内每用户 namespace 隔离、个人 email/chat/notes 索引。**不适用**：global shared corpus 单 query 需触及全部数据。

### 内存-磁盘方案全景表（updated 2026-05-11 post vespa-docs）

| 方案 | 核心思路 | 内存 | 磁盘 | Production 实证 |
|---|---|---|---|---|
| HNSW (memory) | 全图全数据全内存 | high | n/a | Milvus / Qdrant / Weaviate / Pinecone / **Vespa** OSS 全部 |
| **SPANN** | centroid in-mem + posting on-disk | low (only centroids) | high (posting lists) | **Microsoft Bing + Vespa OSS** (2 独立) |
| **DiskANN** | graph in-mem (compressed PQ) + raw vector on-disk | mid (compressed graph) | high | Microsoft 内部 + Milvus DISKANN |
| Starling | DiskANN 优化 segment layout | mid | high | Zilliz / Milvus future |
| FreshDiskANN | DiskANN + streaming insert/delete | mid | high | research / 部分集成 Milvus |
| SPFresh | DiskANN-style + 增量 in-place update | mid | high | research |
| **Vespa Streaming Search** | **无 index, brute-force per-user partition** | very low (45 B/doc metadata) | scan-based | **Vespa OSS** 独占 |

### 决策表（updated 2026-05-11 post vespa-docs）

| Workload | 推荐方案 |
|---|---|
| Shared corpus 百亿规模 + 复杂 ranking pipeline | **Vespa SPANN + 4-phase ranking** |
| Shared corpus 百亿规模 + Milvus 生态 | Milvus DISKANN (DiskANN 实现) |
| Multi-tenant + per-tenant 数据小 | **Vespa Streaming Search**（跳过 ANN tier） |
| Streaming insert + 百亿 disk-resident | FreshDiskANN / SPFresh path |
| 闭源 SaaS 简单 | Pinecone slab |
| Memory-only HNSW 顶到边 | + quantization (RaBitQ / BQ / PQ) 延后 disk 切换 |

### 已知盲区

- **Vespa SPANN 与 Microsoft Bing 实现差异**：centroid update / posting list reorganization 是否相同
- **Streaming Search 与 SPANN crossover** per-tenant 阈值
- **disk-resident HNSW (Vespa hnsw_paged ?) vs SPANN**：两者皆 Vespa 内提供, 实测对比 wiki zero
- **多个 disk path (Starling / FreshDiskANN / SPFresh / SPANN) head-to-head**: 仍 zero

## Cited Pages

- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/starling.md](../../systems/starling.md)
- [systems/freshdiskann.md](../../systems/freshdiskann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
