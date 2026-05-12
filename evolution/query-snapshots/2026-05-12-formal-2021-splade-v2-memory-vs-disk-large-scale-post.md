---
query-key: memory-vs-disk-large-scale
query: "当向量规模超出单机内存时（百亿级以上），有哪些工业方案？SPANN、DiskANN、混合内存-磁盘索引各自的核心思路和取舍是什么？"
date: 2026-05-12
phase: post
ingest-context: formal-2021-splade-v2
wiki-pages-total: 75
cited-pages: [concepts/splade-sparse-retrieval.md, topics/sparse-dense-hybrid-retrieval.md, topics/disk-vs-memory-ann.md]
cited-count: 3
---

# Post-snapshot (formal-2021-splade-v2): memory-vs-disk-large-scale

## TL;DR (delta from kusupati-2022-matryoshka post)

**SPLADE 引入 sparse-side disk path 全新 axis**——之前 wiki 内 disk path (SPANN/DiskANN/Starling/FreshDiskANN/SPFresh/Turbopuffer/DistributedANN) 全部 **dense vector on-disk** patterns. SPLADE 提供 **sparse inverted index on-disk** — 经典 IR 基础设施 (Anserini / Pyserini / Lucene / Elasticsearch / OpenSearch / BlockMaxWAND) 50+ 年成熟. **关键 NEW**: sparse-side disk path **与 dense-side disk path 正交**, 性能 characteristic 差异显著: sparse inverted index posting list 是 sequential read (SSD-friendly), dense ANN graph traversal 是 random read (SSD-stress); 二者在 hybrid pipeline 各自优化 disk tier.

## Answer

### 与之前 ingest 的演进

| | kusupati-2022-matryoshka post | **formal-2021-splade-v2 post (NEW)** |
|---|---|---|
| 5 disk tier | dense-side only | **不变, dense-side; + sparse-side 全新 axis** |
| Sparse-side disk path | 未涵盖 | **NEW: inverted index 50 年 IR 基础设施 (Lucene/Anserini/Pyserini/BlockMaxWAND)** |
| Sparse vs dense disk read pattern | 隐含 | **NEW: sparse sequential read SSD-friendly; dense random read SSD-stress** |

### Sparse-side disk path infrastructure（NEW production reality）

[per formal-2021-splade-v2 + IR 50 年历史]

**Sparse inverted index disk infrastructure**:
- **Lucene** (Apache, 1999+): Elasticsearch / OpenSearch / Solr 底层
- **BlockMaxWAND** (Ding & Suel 2011, wiki 待 ingest): BM25 query optimization
- **Anserini / Pyserini** (Lin 2019+): IR research baseline framework
- **Vespa BlockMaxWAND**: production sparse path (Yahoo! 2003 lineage)

→ Sparse path 50+ 年成熟, **直接服务 SPLADE/BM25 production at giga-scale**. wiki 内之前 dense-disk-focus 完全 ignore 这一巨大成熟生态.

### Sparse vs dense disk read pattern 对比（NEW insight）

[per formal-2021-splade-v2 §3 + DiskANN paper + IR infrastructure]

| Pattern | Sparse (inverted index) | Dense (HNSW / DiskANN graph) |
|---|---|---|
| Storage layout | per-term posting list (doc IDs + impacts) | per-doc vector + neighbor list |
| Query I/O pattern | **sequential read** (posting list traversal) | **random read** (graph traversal, jump to next node) |
| SSD-friendliness | **high** (sequential read 适合 SSD) | low (random read 4KB page miss-heavy) |
| Memory cache 需求 | low (posting list ID compressed) | mid-high (graph adjacency in DRAM) |
| Posting list 压缩 | mature (VBE / FoR / SIMD-BP128 / EliasFano) | mid (Block Shuffling / Starling 优化) |
| Production case mature | 50+ 年 | 5-10 年 |

→ **Sparse path 在 disk tier 自然占优** — IR 50 年 inverted index 优化把 sparse-on-SSD 已经做到极致.

### Hybrid disk path 全景表（updated 2026-05-12 post formal-2021-splade-v2）

| 方案 | Dense path disk | Sparse path disk | 双 path 哲学 |
|---|---|---|---|
| Vespa (BM25 + SPANN) | SPANN centroid + posting on SSD | BlockMaxWAND inverted index on SSD | **first-class both paths native** |
| Weaviate (BM25 + HNSW) | HNSW + RQ8 SSD | BlockMaxWAND BM25 SSD | first-class hybrid via `hybrid()` API |
| Turbopuffer (BM25 + SPFresh) | SPFresh object storage + NVMe cache | BM25 inverted index object storage + NVMe cache | application RRF, 单 namespace 双 path |
| Pinecone Hybrid | dense slab adaptive | sparse vector index | server-side fusion (黑盒) |
| Milvus | DISKANN / HNSW disk-resident | sparse_inverted_index on SSD | multi-field, application fusion |
| **DistributedANN** | distributed KV store dense graph | paper 不涵盖 sparse path | dense only (paper) |

### 决策表（updated 2026-05-12 post formal-2021-splade-v2）

| Workload | 推荐方案 |
|---|---|
| **Hybrid retrieval + cost-sensitive + multi-tenant + cold-latency-tolerant** | **Turbopuffer SPFresh dense + BM25 inverted index sparse + multi_query application RRF** |
| Hybrid shared corpus + 复杂 ranking + ML rerank | Vespa BM25 + SPANN + SPLADE + 4-phase ranking |
| Hybrid shared corpus + AI-native primary | Weaviate BlockMaxWAND + HNSW + `hybrid()` API |
| Pure dense large-scale | DistributedANN (paper 不涉及 sparse) |
| 中等规模 hybrid + 多 index_type | Milvus DISKANN + sparse_inverted |

### 已知盲区

- **SPLADE 在 100B+ scale inverted index 实测**: 论文 MS MARCO 8.8M, giga-scale unknown
- **Hybrid disk path read pattern interleaving**: dense random + sparse sequential 同 SSD 是否互相干扰? 不公开
- **Vespa rank-profile production case at large-scale hybrid**: case study zero
- **Turbopuffer object storage 上 sparse inverted index 实测**: docs 不深入

## Cited Pages

- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
