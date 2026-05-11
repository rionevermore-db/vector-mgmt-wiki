---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-11
phase: post
ingest-context: weaviate-docs
wiki-pages-total: 66
cited-pages: [systems/weaviate.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (weaviate-docs): memory-vs-disk-large-scale

## TL;DR (delta from qdrant-docs post)

**Weaviate 提供 LSM-based 主存储 + memmap 选项**——与 Qdrant per-component tier 类似但不及那么细粒度独立配置。**核心 NEW**：Weaviate 显式集成 **完整 inverted index 套件**（BlockMaxWAND BM25 + Roaring bitmaps for set + bit-sliced range bitmaps），这是 wiki 内首个 vector DBMS **完整 inverted index storage stack**——不只 vector + payload，是真正的"primary DB" 存储层。**百亿+ memory-vs-disk landscape 主流不变**：千亿+ 仍需 DiskANN/SPANN/Starling/FreshDiskANN/SPFresh 路径；Weaviate 在中等规模适合 hybrid (vector + BM25 + filter) workload + LSM-friendly。

## Answer

### 与之前 ingest 的演进

| | qdrant-docs post | **weaviate-docs post (NEW)** |
|---|---|---|
| Per-component storage tier | + Qdrant 4 component independent | **+ Weaviate LSM + memmap + RocksDB 类似** |
| Inverted index 完整套件 | n/a (传统 DB index 不在 vector DBMS 关注) | **+ Weaviate BlockMaxWAND + Roaring + bit-sliced bitmaps** |
| 大规模 disk-resident | DiskANN / SPANN / Starling / FreshDiskANN / SPFresh + Qdrant memmap | **不变**（Weaviate 中等规模） |

### Weaviate 完整 inverted index 套件（NEW）

[per sources/docs/weaviate/llms.txt §Architecture]

> Objects and inverted indexes rely on a flexible **LSM store**. Set-style filters use **Roaring bitmaps**; range filters use **bit-sliced range bitmaps** (requires `index_range_filters=True` on the property — without it, range queries fall back to a full scan). BM25 indexes use **BlockMaxWAND**.

| Index 组件 | 实现 | 用途 |
|---|---|---|
| Object + property storage | **LSM store** | 主数据存储 |
| BM25 full-text | **BlockMaxWAND** (Block-Max WAND) | 文本 + hybrid search |
| Set / equality filter | **Roaring bitmaps** | filter by category/tag/tenant |
| Range filter (price/timestamp) | **Bit-sliced range bitmaps** | opt-in via `index_range_filters=True` |
| Vector | **HNSW + RQ8** + ACORN + HFresh preview | similarity |

→ **wiki 内首个 vector DBMS 完整 inverted index 套件**——之前 vector DBMS 仅 vector + payload filter；Weaviate 把传统 DBMS 完整 inverted index 集成。这是 "primary DB not just vector store" 的核心 storage evidence。

### LSM-friendly 大规模实证（NEW）

[per sources/docs/weaviate/llms.txt §Best Practices]

> Default sharding/replication works for **99% of use cases** with cloud auto-scaling.

LSM-based storage 在大规模下的特性：
- 写入友好（追加 + 周期 compaction）
- Range query 通过 bit-sliced bitmaps 高效 (opt-in)
- 与 vector index (HNSW) 解耦——可独立 tune

但 docs **不公开千亿规模 LSM benchmark**——主流千亿仍是 graph-on-SSD 路径 (DiskANN) 或 cluster-on-SSD 路径 (SPANN)。

### 路线对比表（updated with Weaviate）

| 路线 | 数据驻留 | inverted index | 量化 | 大规模实证 |
|---|---|---|---|---|
| 全内存 graph | DRAM | n/a | ✗ | 1B OOM |
| DiskANN | DRAM (PQ) + SSD | n/a | PQ | 1B SIFT |
| SPANN | DRAM (centroids) + SSD | n/a | ✗ | 1B+ Bing |
| Starling segment | DRAM (nav) + SSD | n/a | PQ | BIGANN 1B 31 segs |
| FreshDiskANN streaming | DRAM (TempIndex) + SSD (LTI) | n/a | PQ | 800M SIFT week-long |
| SPFresh streaming | DRAM (cluster head) + SSD | n/a | ✗ | 1B SIFT 100 days |
| Qdrant per-component | per-component (4 选项 independent) | 简单 payload index | per-vector | 中等规模 |
| **Weaviate (NEW)** | **LSM + memmap + RQ8** | **完整套件 (BlockMaxWAND + Roaring + bit-sliced)** | **RQ8 default** | **中等规模** + hybrid heavy |
| CAGRA GPU | GPU HBM | n/a | future | DEEP-100M |

→ Weaviate 在"完整 inverted index"维度独特——但不挑战千亿单机 disk-resident path。

### 千亿规模下 Weaviate 角色（推断）

[per systems/weaviate.md "Scale 边界"]

- **千亿 + read-heavy + hybrid search heavy** → Weaviate 中等规模 sweet spot 不够；推荐 Milvus / Pinecone
- **千亿 + RAG-heavy** → Weaviate Query Agent stack 是 unique value 但 production benchmark 缺
- **千亿 + cost-sensitive** → SPFresh / SPANN 主导
- **千亿 + 极致 recall** → DiskANN / FreshDiskANN 主导
- **中等规模 (1B-10B) + hybrid + filter + agent stack** → **Weaviate sweet spot**

### 已知盲区

- **Weaviate LSM 千亿规模 production**：完全空白
- **Weaviate inverted index suite 在 hybrid heavy workload 实证**：docs 未公开 benchmark
- **Weaviate vs Qdrant per-component storage tier head-to-head**：完全空白
- **HFresh storage tier integration**：preview，未公开
- **Weaviate + GPU**：不支持，CPU only

## Cited Pages

- [systems/weaviate.md](../../systems/weaviate.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
