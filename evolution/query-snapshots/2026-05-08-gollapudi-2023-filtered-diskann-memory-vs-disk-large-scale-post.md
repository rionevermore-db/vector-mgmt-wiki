---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-08
phase: post
ingest-context: gollapudi-2023-filtered-diskann
wiki-pages-total: 46
cited-pages: [systems/diskann.md, concepts/filtered-vamana.md, topics/disk-vs-memory-ann.md, benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md]
cited-count: 4
---

# Post-snapshot (gollapudi-2023-filtered-diskann): memory-vs-disk-large-scale

## TL;DR (delta from yang-2020-pase post)

**Filtered-DiskANN 在 SSD 维度延伸 DiskANN**——把 [FilteredVamana](../../concepts/filtered-vamana.md) 直接放进 DiskANN PQ-DRAM-graph-SSD 框架，**28M DANN dataset 实测 thousands QPS @ 90%+ recall on inexpensive SSDs**。这意味着 filter-aware build 与 SSD-resident 不冲突——可同时享受 filter 加速 + SSD 大容量。但**实测最大 28M**——百亿+ 推断可行未实证。

## Answer

### Filtered-DiskANN 在 SSD 路径的位置（NEW）

[per benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md "SSD 模式" + topics/disk-vs-memory-ann.md]

```
DiskANN 原架构（subramanya-2019）:
   DRAM: PQ codes（32 byte/vec）
   SSD: Vamana graph + 全精度 vectors（4 KB block）

Filtered-DiskANN 扩展（gollapudi-2023）:
   DRAM: PQ codes + filter-to-medoid map
   SSD: FilteredVamana graph（filter-aware edges）+ 全精度 vectors
```

实测：
- 28M DANN dataset, 24 threads, beam width 4
- specificity 1pc-100pc 全区间稳定
- **thousands QPS @ 90%+ recall**

### Filtered-DiskANN 与其他 SSD 路线的对比（NEW）

[per topics/disk-vs-memory-ann.md "工业方案对比"]

| 路线 | filter-aware? | 最大实测 |
|---|---|---|
| DiskANN（原 Vamana + PQ + SSD）| ✗ | 1B SIFT |
| SPANN（IVF + closure + SSD）| ✗ | 1B+/千亿+ Bing |
| SPFresh（SPANN + LIRE in-place）| ✗ | 1B SIFT/SPACEV |
| Manu SSD-aware hierarchical k-means | ✗ | 1B (NeurIPS 2021 winner) |
| **Filtered-DiskANN** | **✓** | **28M DANN** |

→ **Filtered-DiskANN 是 SSD 路线中第一个 filter-aware 方案**。但实测规模仅 28M——比其他 SSD 路线小一个数量级。论文未实测 1B+ filter-heavy 场景。

### 路线对比表（添加 Filtered-DiskANN 行）

| 路线 | 数据驻留 | 1B SIFT recall @ <5ms | 单机/集群预算 | filter-aware | 代表 |
|---|---|---|---|---|---|
| 全内存 graph | DRAM | 1B 直接 OOM | TB 级 | ✗ | HNSW / NSG |
| 量化 + 全内存 | DRAM | ~62% | 数十 GB | ✗ | Faiss IVFPQ |
| 多机分片 | 分布式 DRAM | ~98% | N × 数十 GB | ✗ | NSG @ Taobao |
| GPU brute force | HBM | 高 | 单卡 ~32 GB | ✗ | Faiss-GPU |
| 磁盘 + 量化导航 + SSD re-rank | DRAM (PQ) + SSD | 98.68% | 64 GB peak | ✗ | DiskANN |
| 磁盘 + IVF + SSD 全精度 | DRAM (centroids) + SSD | >90% @ ~1 ms | ~32 GB peak | ✗ | SPANN |
| 分布式 mmap + 极致压缩 | mmap 分布式 | — | 20 服务器 | ✗ | Meta 1.5T Faiss |
| DBMS LSM segment | DRAM + S3 | — | ~64 GB/节点 | partial（segment 内 filter） | Milvus |
| DBMS cloud-native + zero-disk WAL | etcd + S3 | — | stateless | partial | Milvus 2.x |
| 磁盘 + IVF + 全精度 + In-place 增量 | DRAM + raw NVMe | >0.86 @ ~5 ms | 持续 10 GB | ✗ | SPFresh |
| SaaS + slab + adaptive indexing | Memtable + cache + slab | docs 未公开 | docs 未公开 | metadata filter | Pinecone Serverless |
| DBMS + SSD-aware hierarchical k-means | DRAM (centers) + SSD (4KB block + 4-8× replication) | NeurIPS 2021 winner | docs 未明示 | partial | Manu (Milvus 2.x) |
| OLAP DB + Pangu + VGPQ batching | DRAM + Pangu (distributed) | 13B records production | 70 节点 Alibaba Cloud | **4-plan CBO（search-time）** | AnalyticDB-V |
| OLTP RDBMS + 8KB page IVFFlat / HNSW | DRAM (PG buffer) + PG storage | million-scale per instance | PG 单实例 + 应用分片 | **iterative via amgettuple（search-time）** | PASE |
| **磁盘 + Filter-aware Vamana graph + PQ DRAM**（NEW） | **DRAM (PQ + medoid map) + SSD (filter-aware graph + 全精度)** | **28M DANN @ thousands QPS** | **single machine** | **✓ build-time** | **Filtered-DiskANN** |

### Filtered-DiskANN 与 Milvus / Pinecone / AnalyticDB-V 在 attribute filter 维度的对比（NEW）

[per benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md "vs Milvus"]

```
Milvus HNSW + filter:    QPS < 300 across datasets（论文 §A.2）
Faiss IVF post-process:  1pc filter 下 fail
Faiss IVF inline:        100pc-50pc OK，1pc-25pc 退化
NHQ KGraph:              比 Filtered-DiskANN 低 1 order of magnitude at 100 recall
Filtered-DiskANN:        全 specificity 区间稳定 90%+ recall，QPS 数千+
```

→ **Filtered-DiskANN 在 filter-heavy SSD 场景比所有 wiki 已有 baseline 显著优**。

### 与之前 ingest 的演进

| | yang-2020-pase post | **gollapudi-2023-filtered-diskann post (NEW)** |
|---|---|---|
| SSD 路线数 | 7 | **8**（+ Filtered-DiskANN） |
| Filter-heavy SSD 实证 | 无 | **28M DANN thousands QPS** |
| Build-time filter-aware | ✗（所有 search-time 方法）| **首次出现**（FilteredVamana / StitchedVamana） |

### 已知盲区

- **Filtered-DiskANN 千亿规模**：仅 28M 实证
- **Filtered-DiskANN + cluster-based partition**：Filtered + ADBV / SPANN-style 组合 zero coverage
- **CXL / 持久内存 + filter-aware**：仍未覆盖
- **2026 SIGMOD 跨 model 整合**：talk 当日 demo

## Cited Pages

- [systems/diskann.md](../../systems/diskann.md)
- [concepts/filtered-vamana.md](../../concepts/filtered-vamana.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md](../../benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md)
