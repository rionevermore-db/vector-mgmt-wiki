---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-07
phase: post
ingest-context: milvus-docs
wiki-pages-total: 29
cited-pages: [topics/index-selection.md, topics/disk-vs-memory-ann.md, systems/milvus.md, systems/diskann.md, systems/spann.md, concepts/woodpecker.md, benchmarks/faiss-trillion-scale.md]
cited-count: 7
---

# Post-snapshot (milvus-docs): scale-tier-shifts

## TL;DR (delta from wang-2021-milvus post)

**第 4 个质变点（DBMS 形态）进一步细化**：v2.6.x cloud-native 重写让 DBMS 内部又分裂出**子质变点**——(4a) 单机/单 shard → 多 shard Streaming Node binding、(4b) broker WAL → zero-disk WAL（[Woodpecker](../../concepts/woodpecker.md)）、(4c) 单算法核心 (Faiss) → 多算法核心 (Faiss + DiskANN + ScaNN + RAFT)。这些子质变是**水平扩展能力**与**算法选项**的双重深化。

## Answer

### 五维质变点（updated from wang-2021 post）

| 维度 | 质变点 | 备注 |
|---|---|---|
| **1. 索引必要性** | ~10M | <10M 用 FLAT |
| **2. 单机 RAM 触顶** | ~1B | DRAM → SSD/多机分片 |
| **3. 单机 SSD 触顶** | ~百亿+ | SSD → 分布式 mmap |
| **4. 形态：library → DBMS** | 任何 N + 引入 dynamic/分布式/filter | 与规模正交的"工程维度质变" |
| **4a. DBMS 内部：1.x → 2.x cloud-native (NEW)** | 任何 N + 需要 stateless compute / 弹性扩缩 | Streaming Node + Woodpecker；与是否单点 writer 相关 |
| **4b. DBMS 内部：单算法 → 多算法核心 (NEW)** | 任何 N + 需要异构索引（GPU + DiskANN + ScaNN 同 cluster） | v2.6.x 把 DiskANN/ScaNN/CAGRA 全部集成为索引选项 |

### v2.6.x 暴露的 DBMS 子质变点（NEW）

[per systems/milvus.md "v2.6.x 关键架构改变"]：

**4a. 单 writer → 多 Streaming Node**（写入扩展性质变）

- 1.x：single writer 实例（论文 §5.3 明示 "single writer is sufficient since Milvus is read-heavy"）
- 2.x：每 shard 一 Streaming Node（exactly-one binding），最多 16 shard / collection
- 含义：write throughput 上限从"单 writer 处理能力"提升到"16 streaming node 并行处理能力"
- 触发条件：write-heavy + 单 writer 触顶（吞吐 / latency 限制）

**4b. broker WAL → zero-disk WAL**（运维形态质变）

[per concepts/woodpecker.md]：
- 1.x：Pulsar / Kafka 外部 broker（broker 节点 + local disk + replica 协调）
- 2.x：[Woodpecker](../../concepts/woodpecker.md) 直写 S3（无 broker 节点）
- 含义：从"运维 broker 集群" → "运维 etcd + S3"，运维成本质变
- 触发条件：cloud-native 部署需求 / broker 运维成本超阈值

**4c. 单算法核心 → 多算法核心**（算法异构质变）

- 1.x：Faiss 内核（IVF_FLAT/SQ8/PQ + HNSW + RNSG）
- 2.x：Faiss + DiskANN + ScaNN + NVIDIA RAFT (CAGRA) + 自家 sparse index
- 含义：单 cluster 内可同时承载 dense + sparse + GPU + SSD 多种索引
- 触发条件：workload 异构（部分 collection 用 GPU、部分用 SSD、部分 MIPS）

### 与 wang-2021 post 的差异

| | wang-2021 post | milvus-docs post |
|---|---|---|
| 质变维度 | 4 个（规模 3 + DBMS 形态 1） | **6 个**（4 + DBMS 内部 2 子维度） |
| DBMS 形态质变内容 | 静态 → 动态 + 单维度 → 多维度查询 | + cloud-native 重写 + 算法异构 |
| Cloud-native 角色 | 隐含在 "shared-storage" 中 | 显式分出来：stateless compute + zero-disk WAL + 三 worker 角色 |

### 仍然不算质变的（参数微调）

- 同一 IVF 框架下 K_IVF / nprobe 参数调
- HNSW 内 M / efConstruction 参数调
- Milvus segment 大小 / closure replicas
- Streaming Node 数（在 16 shard 上限内）

### 已知盲区

- **Pinecone pod-based 形态质变**：与 Milvus shared-storage 是两条 (c) DBMS 路线，仍未覆盖
- **2026 SIGMOD 跨 embedding model 整合**：embedding 升级触发的"事件型质变"
- **Read-heavy vs write-heavy 路线分化**：v2.6.x Streaming Node 模型让 write-heavy 可行，但具体阈值文档未拆
- **Multi-cloud / 跨 region 质变**：v2.6.x 是 cloud-native 但 cross-cloud 部署文档未深入

## Cited Pages

- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [concepts/woodpecker.md](../../concepts/woodpecker.md)
- [benchmarks/faiss-trillion-scale.md](../../benchmarks/faiss-trillion-scale.md)
