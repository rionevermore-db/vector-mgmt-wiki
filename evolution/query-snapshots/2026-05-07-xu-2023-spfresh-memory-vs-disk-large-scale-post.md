---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-07
phase: post
ingest-context: xu-2023-spfresh
wiki-pages-total: 33
cited-pages: [systems/spfresh.md, systems/spann.md, systems/diskann.md, systems/milvus.md, concepts/lire.md, concepts/woodpecker.md, topics/disk-vs-memory-ann.md, topics/in-place-vs-out-of-place-updates.md, benchmarks/spfresh-vs-diskann-spann-update.md]
cited-count: 9
---

# Post-snapshot (xu-2023-spfresh): memory-vs-disk-large-scale

## TL;DR (delta from milvus-docs post)

**新增第 5 条 SSD 路线 + 第 4 个 trade-off 维度**：[SPFresh](../../systems/spfresh.md) = SPANN + LIRE in-place update + raw SPDK SSD。在 [SPANN](../../systems/spann.md) 已经达到的 1B / ~1ms / ~32 GB DRAM 之上，**消除周期 rebuild**，把资源峰值从 "rebuild 时 1100 GB + 32 cores × 2 天" 降到 "持续 10 GB + 2 cores"。**第 4 维度（update strategy）**正式加入 disk-vs-memory 选型公式。

## Answer

### 路线对比表完整版（updated with SPFresh）

[per topics/disk-vs-memory-ann.md "工业方案对比"]

| 路线 | 数据驻留 | 1B SIFT recall @ <5ms | 单机内存预算 | 动态数据 | 代表 |
|---|---|---|---|---|---|
| 全内存 graph | DRAM | 1B 直接 OOM | TB 级 | ✗ | HNSW / NSG |
| 量化压缩 + 全内存 | DRAM | ~62% | 数十 GB | ✗ | Faiss IVFPQ |
| 多机分片 | 分布式 DRAM | ~98% | N × 数十 GB | ✗ | NSG @ Taobao |
| GPU brute force | HBM | 高 | 单卡 ~32 GB | ✗ | Faiss-GPU |
| 磁盘 + 量化导航 + SSD re-rank | DRAM (PQ) + SSD | 98.68% | 64 GB | **streamingMerge rebuild**（1100 GB peak） | DiskANN |
| 磁盘 + IVF + SSD 全精度 | DRAM (centroids) + SSD | >90% @ ~1ms | ~32 GB | 周期 rebuild | SPANN |
| 分布式 mmap + 极致压缩 | mmap 分布式 | — | 20 服务器 | ✗ | Meta 1.5T Faiss |
| DBMS LSM segment | DRAM + S3 | — | ~64 GB/节点 | ✓ (LSM merge) | Milvus 1.x |
| DBMS cloud-native + zero-disk WAL | etcd + S3 | — | stateless | ✓ + Streaming Node | Milvus 2.x ([Woodpecker](../../concepts/woodpecker.md)) |
| **磁盘 + IVF + SSD 全精度 + In-place 增量** | **DRAM (centroids + version map) + raw NVMe (SPDK)** | **>0.86 @ ~5ms** | **持续 10 GB**（无 rebuild peak） | **✓ ([LIRE](../../concepts/lire.md))** | **[SPFresh](../../systems/spfresh.md)** |

### 第 4 维度：update strategy（NEW）

之前 disk-vs-memory 三维度：(1) recall, (2) latency, (3) memory cost。
**SPFresh ingest 后新增第 4 维度：update strategy**。

| 系统 | Recall | Latency | Memory | **Update strategy** |
|---|---|---|---|---|
| Faiss IVFPQ | ~62% | <0.1 ms (GPU) | 数十 GB | freeze + retrain |
| DiskANN | 98.68% | <5 ms | 64 GB peak（rebuild 时 1100 GB） | streamingMerge rebuild |
| SPANN | >90% | ~1 ms | ~32 GB peak（rebuild 时多倍） | 周期 rebuild |
| Milvus | 取 segment | — | LSM merge peak | LSM tiered merge |
| **SPFresh** | **>0.86** | **~5 ms** | **持续 10 GB** | **In-place 持续** |

### SPFresh 在 disk-vs-memory 路线中的关键位置（NEW）

[per systems/spfresh.md, topics/in-place-vs-out-of-place-updates.md]：

**SPFresh 对 wiki 现有路线的关系**：
1. **建在 SPANN 之上**：保留 IVF + 全精度 SSD posting + closure clustering + query-aware pruning
2. **打破 DiskANN 全局 rebuild 范式**：实证 cluster-based 路线可以 in-place
3. **未触及 graph-based**：[DiskANN](../../systems/diskann.md) Vamana graph 的 in-place 仍开放
4. **未触及 quantization 路线**：LIRE 假设全精度，PQ 路径未验证 in-place 兼容

### 三层"磁盘 vs 内存"反转的最新一层

[per topics/disk-vs-memory-ann.md "v2.6.x 把 WAL 也搬到 object storage"]：

之前已有：
1. vector data DRAM→SSD（DiskANN/SPANN）
2. write path memory→segment（Milvus LSM）
3. WAL broker disk→object storage（Woodpecker）

**SPFresh 在第 1 层基础上加：raw SSD via SPDK，绕过 OS 文件系统 + KV store**——把"磁盘"语义进一步降级到 raw block append-only。极致 IOPS（饱和 NVMe 400K guarantee）。

> **wiki 解读**：磁盘 vs 内存的层级反转还有空间——直接操作 SSD raw block 是"再降一级"。但代价是 SPDK 开发复杂度 + 自实现 snapshot/recovery。

### 与之前两轮 ingest 的演进

| | wang-2021 post | milvus-docs post | **xu-2023-spfresh post (NEW)** |
|---|---|---|---|
| SSD 路线数量 | 4 | 4 | **5**（+ SPFresh） |
| 主要维度 | recall + latency + memory | + update strategy（弱） | **+ update strategy（强）** |
| Update 解 | 仅 Milvus LSM 部分 | + Milvus Streaming Node | **+ SPFresh in-place** |
| Graph in-place | 未解 | 未解 | **明确仍未解** |

### 已知盲区

- **SPFresh + 多 SSD/分布式**：单机 IOPS bound，多机扩展未实证
- **Pinecone pod-based**：仍未覆盖
- **CXL / 持久内存**：未覆盖
- **FreshDiskANN [Singh 2021]**：graph + streaming update，wiki 未 ingest
- **Aurora-style storage compute fusion + ANN**：未覆盖

## Cited Pages

- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/lire.md](../../concepts/lire.md)
- [concepts/woodpecker.md](../../concepts/woodpecker.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
- [benchmarks/spfresh-vs-diskann-spann-update.md](../../benchmarks/spfresh-vs-diskann-spann-update.md)
