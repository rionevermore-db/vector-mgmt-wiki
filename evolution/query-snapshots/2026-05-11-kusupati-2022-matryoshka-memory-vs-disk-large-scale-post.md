---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-11
phase: post
ingest-context: kusupati-2022-matryoshka
wiki-pages-total: 73
cited-pages: [concepts/matryoshka-embedding.md, topics/adaptive-retrieval-shortlist-rerank.md, systems/diskann.md, systems/distributedann.md, systems/turbopuffer.md, topics/disk-vs-memory-ann.md]
cited-count: 6
---

# Post-snapshot (kusupati-2022-matryoshka): memory-vs-disk-large-scale

## TL;DR (delta from radford-2021-clip post)

**MRL 提供 wiki 内 disk path 的全新优化轴**——之前 wiki 内 disk path (SPANN/DiskANN/Starling/FreshDiskANN/SPFresh/Turbopuffer/DistributedANN) 都是**机制层 disk-aware optimization** (centroid in-mem + posting on-disk / graph in-mem + vector on-disk); MRL 是**算法层 dim reduction**, 让 same recall 用更少 disk IO. **关键 NEW**: production state-of-the-art = **MRL prefix in memory + MRL full-dim on disk** = "MRL-aware DiskANN" 路径 (paper 不命名但 §6 future work 提到 "MRL-aware ANN index" 与 "learnable k-d tree on top of MRL"). 这是 wiki 内之前未涵盖的 disk path optimization frontier.

## Answer

### 与之前 ingest 的演进

| | radford-2021-clip post | **kusupati-2022-matryoshka post (NEW)** |
|---|---|---|
| 5 disk tier | memory / NVMe / object storage / brute-force / distributed KV | **不变** |
| Disk path 优化 axis | mechanism (centroid/graph/posting/scan) | **+ algorithm (MRL prefix dim reduction)** |
| MRL-aware disk path | n/a | **NEW: MRL prefix in mem + full on disk = next-gen DiskANN-style** |

### MRL-aware DiskANN-style pattern（NEW production candidate）

[per kusupati-2022-matryoshka §4 + DiskANN philosophy]

**传统 DiskANN**:
- DRAM: PQ compressed full-dim vectors + graph + navigation
- SSD: full vector for rerank
- **维度**: 单一 d 维 (e.g., 768 or 1024)

**MRL-aware DiskANN (推断 production candidate)**:
- DRAM: MRL **prefix 256-d HNSW/Vamana index** (1/4 storage vs full)
- SSD: MRL full 1024-d vector for exact rerank
- **同 query**: shortlist prefix HNSW → top-K → SSD fetch full → rerank
- 与 paper §4.3 Adaptive Retrieval 等价, 但**适配 DiskANN single-node scale**

**Cost reduction**:
- DRAM 占用: 4× 减少 (256/1024)
- SSD IO per query: 不变 (仍 K=200 candidates × 4096 bytes)
- DRAM-bandwidth: 减 (prefix HNSW 邻居读取数据小)
- Rerank latency: 与传统 DiskANN 一致

### Disk path 全景表（updated 2026-05-11 post kusupati-2022-matryoshka）

| 方案 | DRAM | SSD/Disk | 维度 | MRL-aware? |
|---|---|---|---|---|
| HNSW (memory only) | full | n/a | full | no |
| HNSW + post-hoc quantization | quantized | n/a | full | no |
| SPANN | centroids | posting lists | full | no, but compatible |
| DiskANN | PQ compressed graph | full vectors | full | **可升级 MRL-aware** (推断 SOTA) |
| Starling | DiskANN + segment layout | full | full | no, but compatible |
| FreshDiskANN | DiskANN + streaming | full | full | no |
| SPFresh | centroids + cluster | posting | full | no, but compatible |
| Vespa Streaming | metadata | raw vector scan | full | compatible |
| Turbopuffer SPFresh | metadata + centroids | posting lists | full | **架构兼容**: cell type 可 store prefix |
| DistributedANN | head index (in-mem ANN) | distributed KV graph | full (with OPQ 64-d) | paper 不明示 MRL adoption |
| **MRL-aware DiskANN (推断)** | **MRL prefix HNSW** | **MRL full** | **prefix + full** | **NEW production candidate** |

### MRL 与各 disk path 的正交性

[per kusupati-2022-matryoshka §6 + wiki disk path]

MRL **正交** 所有 disk path mechanism. 任何 vendor 可以:
1. 保持现有 disk path (SPANN/DiskANN/SPFresh)
2. **存 MRL-trained embedding 而非传统单 dim**
3. 应用层选 prefix dim → disk path service same query at lower cost

**Production cost reduction estimate**:
- 16 节点 × 1B docs × 768-d MRL (voyage-3) workload
- 传统 DiskANN: DRAM 占用 ~ N × d_PQ / R_compress = 1B × 768 / 6 (OPQ) = 128 GB
- MRL-aware DiskANN: DRAM ~ N × d_prefix / R_compress = 1B × 256 / 6 = 43 GB (**3× cheaper**)
- 同 recall, 同 search latency

### 决策表（updated 2026-05-11 post kusupati-2022-matryoshka）

| Workload | 推荐方案 |
|---|---|
| **MRL-trained embedding + large-scale disk + cost-priority** | **MRL-aware DiskANN-style** (推断, 当前 wiki 无 vendor native, application 端可实现) |
| Cost-sensitive + multi-tenant + cold-latency-tolerant | Turbopuffer SPFresh + MRL prefix cell type |
| Shared corpus 复杂 ranking | Vespa SPANN + matryoshka cell type + 4-phase ranking |
| 千亿 + 单图 distributed | DistributedANN (MRL adoption unknown) |
| Streaming + 百亿 disk-resident | FreshDiskANN / SPFresh + MRL prefix (推断) |
| 闭源 SaaS managed | Pinecone slab (MRL 不公开) |

### 已知盲区

- **MRL-aware DiskANN production case**: 论文 §6 future work, vendor 端 zero production
- **MRL on DistributedANN single-graph**: paper 不明示
- **MRL prefix HNSW α-RNG 性质**: prefix 上的 graph build quality 退化曲线
- **SSD-resident MRL + post-hoc binary 双轨**: 实际 production stack 不公开

## Cited Pages

- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
- [topics/adaptive-retrieval-shortlist-rerank.md](../../topics/adaptive-retrieval-shortlist-rerank.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
