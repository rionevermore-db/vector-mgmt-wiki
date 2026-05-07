---
query-key: giga-scale-sharding
query: "千亿/万亿向量在私有云（16 × 128U + 1TB RAM 节点，768-d，read-heavy，P99<50ms，周级别全量更新）下的分片策略？"
date: 2026-05-07
phase: post
ingest-context: milvus-docs
wiki-pages-total: 29
cited-pages: [systems/milvus.md, concepts/woodpecker.md, benchmarks/milvus-vs-prior-sift10m-deep10m.md, benchmarks/faiss-trillion-scale.md, systems/spann.md, systems/diskann.md, topics/disk-vs-memory-ann.md, topics/index-selection.md, queries/index-architecture-global-vs-routed.md]
cited-count: 9
---

# Post-snapshot (milvus-docs): giga-scale-sharding

## TL;DR (delta from wang-2021-milvus post)

**新增 v2.6.x cloud-native 视角**：Milvus 2.x 把 SIGMOD 1.x "single writer + multi reader shared-storage" 重新设计为 **四层 disaggregated**——Streaming Node（每 shard 绑定 exactly-one node）+ Query Node + Data Node + [Woodpecker](../../concepts/woodpecker.md) zero-disk WAL。给定 16 × 1TB 私有云：**Streaming Node 数 = 总 shard 数**（默认每 collection ≤16 shard），可以让 16 节点正好对齐 shard 单 node 绑定。**Woodpecker S3 WAL 在 16 节点 + 共享存储部署下是 sweet spot**——避免独立 Kafka/Pulsar broker。

## Answer

### 16 节点 v2.6.x 部署的具体形态（NEW）

[per systems/milvus.md "v2.6.x: 关键架构改变"]：

| Layer | 节点分配建议（16 物理节点） |
|---|---|
| **Access Layer** | 多 Proxy 副本，stateless，可与 Streaming Node 共部署或独立 2 节点 |
| **Coordinator** | master-slave HA，2 节点（1 active + 1 hot standby） |
| **Streaming Node** | **每 shard 1 个 → 最多 16**（恰好对齐物理节点数）；承担 WAL + growing data + Query Delegator |
| **Query Node** | 多副本加载 sealed segments；可与 Streaming Node 共部署（K8s pod affinity） |
| **Data Node** | compaction + index build；可弹性扩缩，工作负载低时缩到 0 |
| **Storage** | etcd（meta，3-5 副本）+ S3/MinIO（object）+ Woodpecker（WAL on S3） |

**关键设计点**：
- 16 shard 上限 [per sources/docs/milvus/site/en/about/limitations.md]——给定 16 节点物理预算，sharding factor 锁死
- 每 shard ≤ 1024 partition、64 fields、单 collection entity 数无上限
- partition_key 隔离做 multi-tenancy，无需多 collection（避免 65k collection 上限）

### Woodpecker 在此场景的优势 [per concepts/woodpecker.md]

vs SIGMOD 1.x 的 Pulsar/Kafka broker：

| | Pulsar/Kafka 模式（1.x 推断） | Woodpecker S3 模式（2.x） |
|---|---|---|
| 节点占用 | 需 3-5 broker 节点 + local disk | **零** broker 节点 |
| 写延迟 | 30-60 ms | 166 ms (S3 batch) / single-digit ms (QuorumBuffer) |
| 写吞吐 | ~130 MB/s/broker | **750 MB/s** (S3) |
| 故障域 | broker disk + replica 协调 | S3 11-9 耐久 + etcd metadata |

给定 P99 < 50 ms，QuorumBuffer 模式（single-digit ms）可达；MemoryBuffer 模式（200-500 ms）超出预算。

### 索引选择（v2.6.x 扩展，NEW）

[per topics/index-selection.md, systems/milvus.md "v2.6.x 索引家族"]：

768-d float read-heavy 数据：
- 优先 **HNSW**（生态最广，million-billion 规模标准）
- 数据装不下 RAM 时切 **DISKANN**（Vamana + SSD 全精度 re-rank，集成 [DiskANN](../../systems/diskann.md) 算法）
- 需 MIPS 优化时切 **SCANN**（集成 Google [ScaNN](../../concepts/scann.md) anisotropic loss）
- GPU 节点可用时切 **GPU_CAGRA**（NVIDIA RAFT，比 HNSW 高 throughput）

### 周级别全量更新（v2.6.x 路径，NEW）

v2.6.x **CDC (Change Data Capture)** 工具 [per sources/docs/milvus/site/en/about/overview.md "Tools and Ecosystem"]：
- 在两个 Milvus 实例间同步增量
- Primary/standby 灾备模式
- 支持周级"双索引切换 + cutover"工作流

具体路径：
1. 旧 collection 持续服务
2. 新 collection 后台 ingest（Streaming Node 异步 flush 到 sealed segment）
3. CDC 把增量 delta 也复制到新 collection
4. 切流量（K8s service 切换）
5. 删除旧 collection（释放 segment）

### 已知盲区（仍未覆盖）

- **v2.6.x 实测 1B+ 数字**：[per benchmarks/milvus-vs-prior-sift10m-deep10m.md] 是 1.x 数据；2.x cloud-native 实测在 [Zilliz VectorDBBench](https://github.com/zilliztech/VectorDBBench) 但 wiki 未 ingest
- **768-d 实测**：当前 wiki benchmark 维度上限 144/128-d
- **Pinecone pod-based 架构**：另一种 (c) 路由 DBMS 路线，wiki 仍未覆盖
- **2026 SIGMOD Integrating Vector Databases across Embedding Models**：talk 当日 demo

## Cited Pages

- [systems/milvus.md](../../systems/milvus.md)
- [concepts/woodpecker.md](../../concepts/woodpecker.md)
- [benchmarks/milvus-vs-prior-sift10m-deep10m.md](../../benchmarks/milvus-vs-prior-sift10m-deep10m.md)
- [benchmarks/faiss-trillion-scale.md](../../benchmarks/faiss-trillion-scale.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [topics/index-selection.md](../../topics/index-selection.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
