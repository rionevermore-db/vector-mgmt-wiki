---
query-key: memory-vs-disk-large-scale
query: "向量超出单机内存（百亿+）时有哪些工业方案？SPANN / DiskANN / 混合内存-磁盘各自核心思路与取舍？"
date: 2026-05-07
phase: post
ingest-context: milvus-docs
wiki-pages-total: 29
cited-pages: [systems/diskann.md, systems/spann.md, systems/milvus.md, concepts/woodpecker.md, concepts/vamana.md, topics/disk-vs-memory-ann.md, benchmarks/diskann-sift1b.md, benchmarks/spann-vs-diskann-billion.md]
cited-count: 8
---

# Post-snapshot (milvus-docs): memory-vs-disk-large-scale

## TL;DR (delta from wang-2021-milvus post)

**第三种"磁盘 vs 内存"反转**：[per topics/disk-vs-memory-ann.md "v2.6.x 把 WAL 也搬到 object storage"] Milvus v2.6 自研 [Woodpecker](../../concepts/woodpecker.md) 把 **WAL 也下放 object storage**——彻底无本地磁盘。三层反转累积：(1) vector data DRAM→SSD（DiskANN/SPANN）；(2) write path memory→segment（Milvus LSM）；(3) **WAL broker disk→S3**（Woodpecker）。每一层都是"用更慢但更便宜/可靠的存储替代更快的"。

## Answer

### 三层"磁盘 vs 内存"反转完整图景（NEW）

[per topics/disk-vs-memory-ann.md "v2.6.x 把 WAL 也搬到 object storage"]：

| 层级 | 反转 | 代表 |
|---|---|---|
| **第一层：vector data** | DRAM → SSD | [DiskANN](../../systems/diskann.md) (Vamana graph + SSD), [SPANN](../../systems/spann.md) (IVF + SSD posting) |
| **第二层：write path** | in-memory → 周期 flush 到 segment | Milvus LSM segment + S3/HDFS |
| **第三层（NEW）：WAL** | broker local disk → object storage | [Woodpecker](../../concepts/woodpecker.md) zero-disk |

每一层目标相同：**stateless compute + 共享存储 + 弹性扩缩**。

### Woodpecker 的具体设计 [per concepts/woodpecker.md]

```
┌──────────────────────────────────┐
│ Client                           │
├──────────────────────────────────┤
│ LogStore                         │
│ ├─ 高速 write buffering          │
│ └─ 异步 upload to storage        │
├──────────────────────────────────┤
│ Storage backend (S3 / GCS / etc) │
│ + Etcd (metadata)                │
└──────────────────────────────────┘
```

两种部署：
- **MemoryBuffer**：embedded client 周期 flush，**200-500 ms 写延迟**
- **QuorumBuffer**：3-replica quorum，**single-digit ms 延迟**，强一致

性能对比：
- WP S3 mode：**750 MB/s**（5.8× over Kafka 130 MB/s, 7× over Pulsar 107 MB/s）
- WP Local mode：**450 MB/s + 1.8 ms 延迟**

### Milvus v2.6.x 路线在路线图中的新位置（NEW）

之前 wiki 现有路线（wang-2021-milvus post）：

| 路线 | 数据驻留 | 1B SIFT recall @ <5ms | 单机内存预算 | 动态数据 |
|---|---|---|---|---|
| 全内存 graph | DRAM | 1B 直接 OOM | TB 级 | ✗ |
| 量化压缩 + 全内存 | DRAM | ~62% | 数十 GB | ✗ |
| 多机分片 | 分布式 DRAM | ~98% | N × 数十 GB | ✗ |
| GPU brute force | HBM | 高 | 单卡 ~32 GB | ✗ |
| 磁盘 + 量化导航 + SSD re-rank | DRAM (PQ) + SSD | 98.68% | 64 GB | ✗ |
| 磁盘 + IVF + SSD posting | DRAM (centroids) + SSD | >90% @ ~1 ms | ~32 GB | ✗ |
| 分布式 mmap + 极致压缩 | mmap 分布式 | — | 20 服务器 | ✗ |
| DBMS shared-storage + LSM (Milvus 1.x) | DRAM + S3 segments | 论文未实测 | ~64 GB/节点 | ✓ |

**新增 v2.6.x cloud-native 路线**：

| 路线 | 数据驻留 | 关键创新 | 动态数据 |
|---|---|---|---|
| **DBMS cloud-native + zero-disk WAL (Milvus 2.x)** | etcd (meta) + S3 (data + WAL) + 节点 stateless | **Woodpecker zero-disk** + Streaming Node + DiskANN 集成 | ✓ |

**与 1.x 的关键差异**：
- 1.x：单 writer + Pulsar/Kafka broker（broker 需 local disk）
- 2.x：每 shard 一 Streaming Node（exactly-one binding）+ Woodpecker 直写 S3
- **彻底 stateless compute**——所有 Worker Node 故障可在新 K8s pod 上从 etcd + S3 + Woodpecker 完整恢复

### v2.6.x 把 [DiskANN](../../systems/diskann.md) 集成为索引选项（NEW）

[per systems/milvus.md "v2.6.x 索引家族"] Milvus v2.6.x 把 DiskANN 作为索引类型暴露——用户在 collection 上 `index_type=DISKANN` 即享受单机 64 GB RAM + SSD 跑 1B + 98.68% recall 的能力。

这意味着 **v2.6.x Milvus = DBMS 层 + DiskANN/SPANN 算法核心**。不再是 SIGMOD 1.x 时代的"DBMS + Faiss-only"。

### Open / 仍未覆盖

- **SPANN 是否也被 Milvus 集成**：v2.6.x 文档列 DISKANN 但未提 SPANN，可能 Microsoft 没有 SPTAG 公开 binding
- **Pinecone pod-based 架构**：仍未覆盖
- **CXL / 持久内存**：仍未覆盖
- **Woodpecker 跨 region / 多云延迟特性**：[per concepts/woodpecker.md Open Q]
- **2.6.x cloud-native 实测在 1B+ recall/latency**：wiki 实测仍只有 1.x SIGMOD 数字

## Cited Pages

- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/woodpecker.md](../../concepts/woodpecker.md)
- [concepts/vamana.md](../../concepts/vamana.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [benchmarks/diskann-sift1b.md](../../benchmarks/diskann-sift1b.md)
- [benchmarks/spann-vs-diskann-billion.md](../../benchmarks/spann-vs-diskann-billion.md)
