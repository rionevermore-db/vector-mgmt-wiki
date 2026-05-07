---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-07
phase: post
ingest-context: wang-2021-milvus
wiki-pages-total: 28
cited-pages: [systems/diskann.md, systems/spann.md, systems/milvus.md, concepts/vamana.md, concepts/product-quantization.md, topics/disk-vs-memory-ann.md, benchmarks/diskann-sift1b.md, benchmarks/spann-vs-diskann-billion.md, benchmarks/faiss-trillion-scale.md, benchmarks/milvus-vs-prior-sift10m-deep10m.md]
cited-count: 10
---

# Post-snapshot: memory-vs-disk-large-scale

## TL;DR

Wiki 现有 **四条已 ingest 的 SSD/分布式路线**：(1) **DiskANN** = Vamana graph + PQ DRAM 导航 + SSD 全精度 re-rank；(2) **SPANN** = IVF + 全精度 SSD posting list + closure；(3) **Faiss IVFPQ** + 多机 mmap（Meta trillion-scale）；(4) **Milvus shared-storage 分布式 DBMS**——LSM segment 模型 + S3/HDFS 共享存储 + dynamic data。**新增 (4) 的关键差异**：Milvus 是唯一同时支持持续写入 + 分布式 + GPU 的方案；其他三条都假设静态数据。

## Answer

### 路线对比表（updated with Milvus）

| 路线 | 数据驻留 | 1B SIFT recall @ <5ms | 单机内存预算 | 动态数据 | 代表 |
|---|---|---|---|---|---|
| 全内存 graph | DRAM | 1B 直接 OOM | TB 级 | ✗ | HNSW / NSG |
| 量化压缩 + 全内存 | DRAM | ~62% (IVFOADC plateau) | 数十 GB | ✗ | Faiss IVFPQ |
| 多机分片 | 分布式 DRAM | ~98% | N × 数十 GB | ✗ | NSG @ Taobao 32-partition |
| GPU brute force | HBM | 高 | 单卡 ~32 GB | ✗ | Faiss-GPU |
| **磁盘 + 量化导航 + SSD re-rank** | DRAM (PQ) + SSD | **98.68%** | **64 GB** | ✗ | **DiskANN** |
| **磁盘 + IVF + 全精度 posting list** | DRAM (centroids) + SSD | **>90% @ ~1 ms** | ~32 GB | ✗ | **SPANN** |
| 分布式 mmap + 极致压缩 | mmap 分布式存储 | — | 20 服务器 | ✗ | Meta 1.5T Faiss |
| **DBMS shared-storage + LSM**（NEW） | DRAM + S3/HDFS 共享存储 | 论文未实测单机 1B 数字 | ~64 GB/节点 | **✓** | **Milvus** [per systems/milvus.md] |

### Milvus 的 DBMS-style "内存 + 磁盘"模式 [per systems/milvus.md §"动态数据"]

不像前三条 SSD 路线（DiskANN/SPANN/trillion mmap）假设数据冻结一次构建，Milvus 走 **LSM tiered merge** + **shared storage**：

- **内存 MemTable** 接受新写入
- 阈值或每秒 flush 为 immutable segment（默认 1 GB）→ 写到 local FS / S3 / HDFS
- 后台 tiered merge 合并相近大小 segment
- Index 默认仅 large segment 自动建（user 可手动）
- **Snapshot isolation**：query 看启动时刻 snapshot；后续写入创新 snapshot

[per topics/disk-vs-memory-ann.md §"DBMS 层的'内存 + 异步刷盘'模式"] 这一模式与 DiskANN/SPANN 的差异是**算法 vs DBMS 视角**：前者优化 query，后者同时优化 query + write + 持久化 + 一致性。

### 各路线核心思路（updated）

**DiskANN 核心** [per systems/diskann.md]：
- Vamana graph + α-controlled → 小 diameter（hop 数比 HNSW/NSG 少 2-3×）
- PQ codes 在 DRAM 做导航；全精度向量 + graph edges 在 SSD（同扇区 piggyback）
- Beam search W=4-8 批量 SSD I/O
- 全精度 re-rank "免费"（读邻居顺手取全精度坐标）
- 1B SIFT 单机 64 GB RAM，1-recall@1 = 98.68% @ <5 ms

**SPANN 核心** [per systems/spann.md]：
- 不用 PQ，全程全精度——内存只 centroids（占 ~16% N），SSD 放完整向量
- HBC + closure clustering + query-aware dynamic pruning + RNG rule
- 1B SIFT / DEEP / SPACEV @ 90% recall ~1 ms，单机 ~32 GB RAM
- Bing 几千亿规模生产部署

**Milvus 核心**（NEW）[per systems/milvus.md]：
- **建在 Faiss 之上的 DBMS 层**：保留 IVF/PQ/HNSW 内核，补齐 dynamic + distributed + GPU + filter
- **Cache-aware partition**：query 切块让 query+heap 落进 L3，1.5×-2.7× 速度
- **Runtime SIMD hooking**：同 binary 在不同 CPU 自动选最优 SIMD
- **SQ8H hybrid CPU/GPU 索引**：GPU 装不下数据时优于 pure GPU
- **Shared-storage 分布式**：单 writer + 多 reader + S3/HDFS；K8s 自动 restart

### DiskANN vs SPANN 关键差异（保留）

| | DiskANN | SPANN |
|---|---|---|
| 算法路线 | Graph (Vamana) + SSD | Inverted file + SSD |
| 是否用 PQ | 是（导航） | 否（全精度） |
| SSD 访问模式 | 多次小读 | 少量大读 |
| 90% recall 延迟 | ~3-4 ms | ~1 ms |

### Trillion-scale 跨档：mmap + 极致压缩 [per benchmarks/faiss-trillion-scale.md]

Meta 1.5T × 144-d：PCAR72,SQ6 = 54 字节/向量 → 83 TiB；HNSW 10M coarse；20 服务器 mmap；单查询 ~1s。

### Milvus 与其他三条路线的取舍

**适用 Milvus 的场景**：
- 需要持续写入（电商商品、用户行为日志）
- 需要 attribute filter / multi-vector query 等高级查询
- 需要分布式弹性扩缩容
- DBMS 操作语义（snapshot isolation、ACID 在 segment 级）

**不适用 Milvus 的场景**：
- 单机最低延迟（<1 ms）+ 静态数据 → SPANN 更优
- 单机 64 GB RAM 极小预算 → DiskANN 更优
- 只读 + 万亿规模 → Meta Faiss mmap 路线

### 已知盲区

- **Milvus 2.0+ cloud-native 重写**：本论文 1.x 架构；2.0+ 重新分层（log broker + DataNode + QueryNode）尚未 ingest
- **Pinecone pod-based 架构**：商业云向量 DB 主流形态
- **CXL / 持久内存中间层**：DRAM 与 SSD 之间新存储层，未覆盖
- **网络存储下的 ANN**：所有 disk-resident 分析假设本地 NVMe；S3 直读延迟特性 wiki 仅在 Milvus shared-storage 略提
- **DiskANN/SPANN + DBMS 包装**：是否有研究把它们用 LSM segment 化？wiki 未覆盖

## Cited Pages

- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/vamana.md](../../concepts/vamana.md)
- [concepts/product-quantization.md](../../concepts/product-quantization.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [benchmarks/diskann-sift1b.md](../../benchmarks/diskann-sift1b.md)
- [benchmarks/spann-vs-diskann-billion.md](../../benchmarks/spann-vs-diskann-billion.md)
- [benchmarks/faiss-trillion-scale.md](../../benchmarks/faiss-trillion-scale.md)
- [benchmarks/milvus-vs-prior-sift10m-deep10m.md](../../benchmarks/milvus-vs-prior-sift10m-deep10m.md)
