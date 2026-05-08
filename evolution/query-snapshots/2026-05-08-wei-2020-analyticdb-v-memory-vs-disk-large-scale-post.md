---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-08
phase: post
ingest-context: wei-2020-analyticdb-v
wiki-pages-total: 42
cited-pages: [systems/analyticdb-v.md, concepts/vgpq.md, systems/milvus.md, systems/diskann.md, systems/spann.md, topics/disk-vs-memory-ann.md, benchmarks/analyticdb-v-vs-twostep.md]
cited-count: 7
---

# Post-snapshot (wei-2020-analyticdb-v): memory-vs-disk-large-scale

## TL;DR (delta from guo-2022-manu post)

**新增第 7 条路线**：[ADBV](../../systems/analyticdb-v.md) Lambda framework——streaming HNSW (DRAM) + batching VGPQ (Pangu 分布式存储)。**13B records / 30 TB production**——是 wiki 已 ingest 部署中**第二大**（仅次于 Meta 1.5T Faiss）。设计哲学：streaming layer 用 in-memory graph；batching layer 用分布式存储 + quantization——把"内存 vs 磁盘"分到不同算法层。

## Answer

### 路线对比表完整版（updated with ADBV）

| 路线 | 数据驻留 | 1B SIFT recall @ <5ms | 单机/集群预算 | 动态数据 |
|---|---|---|---|---|
| 全内存 graph | DRAM | 1B 直接 OOM | TB 级 | ✗ |
| 量化压缩 + 全内存 | DRAM | ~62% | 数十 GB | ✗ |
| 多机分片 | 分布式 DRAM | ~98% | N × 数十 GB | ✗ |
| GPU brute force | HBM | 高 | 单卡 ~32 GB | ✗ |
| 磁盘 + 量化导航 + SSD re-rank | DRAM (PQ) + SSD | 98.68% | 64 GB peak | streamingMerge |
| 磁盘 + IVF + SSD 全精度 | DRAM (centroids) + SSD | >90% @ ~1 ms | ~32 GB peak | 周期 rebuild |
| 分布式 mmap + 极致压缩 | mmap 分布式 | — | 20 服务器 | ✗ |
| DBMS LSM segment / cloud-native | DRAM + S3 | — | ~64 GB/节点 | LSM merge |
| 磁盘 + IVF + 全精度 + In-place 增量 | DRAM + raw NVMe | >0.86 @ ~5 ms | 持续 10 GB | LIRE in-place |
| SaaS + slab + adaptive indexing | Memtable + cache + slab on object storage | docs 未公开 | docs 未公开 | slab merge |
| DBMS + SSD-aware hierarchical k-means + LSH replication | DRAM (centers) + SSD (4KB block + 4-8× replication) | NeurIPS 2021 winner | docs 未明示 | DBMS 全套 |
| **OLAP DB + Pangu 分布式存储 + Lambda streaming/batching**（NEW） | **DRAM (HNSW streaming) + Pangu (VGPQ batching)** | **13B records / 30 TB production** | **70 节点 Alibaba Cloud** | **Lambda async merge + 4 plan CBO** |

### ADBV 与其他 (大数据 vs 内存) 路线的对比（NEW）

[per systems/analyticdb-v.md "与 wiki 已有系统的对比"]

| 系统 | 起点 | streaming/batching 分工 |
|---|---|---|
| Faiss | Library | n/a（用户胶合） |
| DiskANN | 算法系统 | 单一 Vamana graph + PQ DRAM + SSD 全精度 |
| SPANN | 算法系统 | 单一 IVF + closure + SSD 全精度 |
| SPFresh | 算法系统 + in-place | 单一 IVF + LIRE 持续维护 |
| Milvus 1.x | DBMS | 单一 segment + LSM merge |
| Milvus 2.x / Manu | DBMS cloud-native | 单一 segment + Streaming Node + LSM |
| Pinecone | SaaS | 单一 namespace + slab merge + adaptive |
| **AnalyticDB-V (ADBV)** | **OLAP DB** | **Streaming HNSW + Batching VGPQ 双算法**（独有） |

→ ADBV 是 wiki 唯一**streaming/batching 用两种不同算法**的系统：streaming 用 graph (HNSW)，batching 用 quantization (VGPQ)。其他系统都是单一算法在不同 segment / slab / posting 内运行。

### Pangu 分布式存储的位置（NEW）

[per systems/analyticdb-v.md "架构图"]

ADBV 不直接用 SSD 或 S3——用 **Pangu**（Alibaba 自家分布式存储）：
- 与 S3 / GCS 同代但闭源
- 所有 baseline VGPQ index 在 Pangu 上
- WAL 也在 Pangu

→ **私有云用户复刻 ADBV 需要替换 Pangu 为 S3 / MinIO / HDFS**——这是 ADBV 的开源天花板。

### 与之前 ingest 的演进

| | guo-2022-manu post | **wei-2020-analyticdb-v post (NEW)** |
|---|---|---|
| 路线数 | 11 | **12** |
| Streaming/batching 双算法 | 未出现 | **首次出现 ADBV 模式** |
| 大规模 production benchmark | Manu 100M | **ADBV 13B / 30 TB** |
| OLAP DB 路径 | 隐含 | **首次显式** |

### 已知盲区

- **ADBV 的 SSD/S3-replacement 实验**：闭源 Pangu 对私有云不可用
- **ADBV vs Manu / Milvus 实测对比**：双方都不公开
- **VGPQ 在万亿规模**：未实测

## Cited Pages

- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
- [concepts/vgpq.md](../../concepts/vgpq.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [benchmarks/analyticdb-v-vs-twostep.md](../../benchmarks/analyticdb-v-vs-twostep.md)
