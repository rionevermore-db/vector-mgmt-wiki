---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-12
phase: post
ingest-context: santhanam-2022-colbertv2
wiki-pages-total: 79
cited-pages: [concepts/colbertv2.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (santhanam-2022-colbertv2): memory-vs-disk-large-scale

## TL;DR (delta from lancedb-docs post)

**ColBERTv2 的 disk path = PLAID-style token-level inverted file**——不是 vector-specific disk path (SPANN/DiskANN/Starling/etc.), 不是 object storage primary, 不是 distributed KV. 是 **multi-vector inverted index on SSD**——token centroids in DRAM + per-centroid token list on SSD. **关键 NEW**: 这是 wiki 内**第 8 类 disk philosophy**: multi-vector inverted file (token-level posting list).

## Answer

### Disk philosophy 8 类全景（updated 2026-05-12 post colbertv2）

| # | 方案 | 代表 |
|---|---|---|
| 1 | Memory-only (HNSW) | OSS DBMS default |
| 2 | NVMe vector-specific (SPANN/DiskANN/Starling) | Microsoft Bing 历史 + Milvus DISKANN |
| 3 | Object storage primary | Turbopuffer + Chroma Cloud |
| 4 | Brute-force per-tenant disk scan | Vespa Streaming |
| 5 | Distributed KV store | DistributedANN @ Bing |
| 6 | General-RDBMS heap | pgvector |
| 7 | Format-first OSS columnar | LanceDB Lance format |
| **8** | **Multi-vector inverted file (NEW)** | **ColBERTv2 PLAID-style** |

### ColBERTv2 PLAID 与其他 disk path 哲学不同

[per santhanam-2022-colbertv2 §3.3]

- (1-2) HNSW / SPANN / DiskANN: single-vector graph or centroid + SSD posting
- (3) Object storage primary: full vector on S3 + SSD cache
- (5) Distributed KV: single graph nodes on KV store
- **(8) ColBERTv2 PLAID**: token-level centroids (IVF-style) + per-centroid token posting list (residual compressed)

→ **Multi-vector inverted file** is unique 8th disk philosophy——继承 IR 50 年 inverted index 哲学 but applied to per-token multi-vector retrieval.

### 已知盲区

- **PLAID-style inverted file 大规模 production case**: paper MS MARCO only
- **ColBERTv2 + 其他 disk path (SPANN style centroid + posting)**: 是否可叠加? Open
- **Multi-vector inverted file vs single-vector ANN performance**: 不同 axis 不可直接对比

## Cited Pages

- [concepts/colbertv2.md](../../concepts/colbertv2.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
