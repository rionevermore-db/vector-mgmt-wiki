---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下索引架构选 (a) 全局单一索引 还是 (c) 层次路由结构？"
date: 2026-05-07
phase: post
ingest-context: milvus-docs
wiki-pages-total: 29
cited-pages: [queries/index-architecture-global-vs-routed.md, topics/disk-vs-memory-ann.md, systems/milvus.md, systems/diskann.md, systems/spann.md, concepts/woodpecker.md]
cited-count: 6
---

# Post-snapshot (milvus-docs): index-architecture-global-vs-routed

## TL;DR (delta from wang-2021-milvus post)

**Milvus v2.6.x 把 (c) 层次路由的内部细节进一步暴露**：vchannel（虚拟通道）→ pchannel（物理通道）→ Streaming Node（exactly-one binding）的三级映射 + Query Delegator fan-out 模型。**Streaming Node + Query Node + Data Node 三角色分离**让"路由 + 计算"职责清晰：Streaming Node 路由 + 实时查询，Query Node 历史查询，Data Node 离线 compaction/索引。

## Answer

### v2.6.x (c) 层次路由的具体形态（NEW）

[per sources/docs/milvus/site/en/reference/architecture/data_processing.md, streaming_service.md]：

**Shard / Channel 路由层级**：
```
Collection
  ├─ shard₀ ←──── vchannel₀ ←──── pchannel_a ←──── Streaming Node X (exactly-one)
  ├─ shard₁ ←──── vchannel₁ ←──── pchannel_b ←──── Streaming Node Y
  └─ ...
       ↓
   Query Delegator on each Streaming Node
       ↓ fan-out
   Query Node 1, Query Node 2, ... (load sealed segments from S3)
```

**关键约束**（v2.6.x docs）：
- 单 collection 最多 16 shard [per sources/docs/milvus/site/en/about/limitations.md]
- 每 vchannel 映射到 exactly-one Streaming Node（fencing 保证）
- Query Delegator 在 Streaming Node 上协调单 shard fan-out

### Search 路径细节（NEW，v2.6.x）

[per sources/docs/milvus/site/en/reference/architecture/architecture_overview.md "Example Data Flow: Search Operation"]：

1. Client → Load Balancer → 可用 Proxy
2. Proxy 用 routing cache 决定目标节点；cache 失效时联系 Coordinator
3. Proxy → 相关 Streaming Nodes
4. **每 Streaming Node 本地搜 growing data + 协调 Query Nodes 搜 sealed segments**
5. Query Nodes 从 Object Storage load sealed segments，segment 级 search
6. **三级 reduce**：QN reduce 跨 segments → SN reduce 跨 QN → Proxy reduce 跨 SN → Client

> **wiki 解读**：v2.6.x 路由不仅是"coarse quantizer 找 IVF bucket"——它是物理 shard 路由 + 角色分工 + 三级 reduce 的复合结构。比 1.x SIGMOD 论文的描述深入很多。

### Trade-off 表（updated with v2.6.x）

| 维度 | (a) 全局 graph | (c) Meta/SPANN 路由 | (c) Milvus 1.x segment | (c) **Milvus 2.x** (NEW) |
|---|---|---|---|---|
| 路由层 | 无 | coarse quantizer | segment | shard → vchannel → pchannel → Streaming Node + Query Delegator + Query Node |
| 写入节点 | n/a | n/a | single writer | **每 shard 一 Streaming Node** |
| 读节点 | 单一 graph 全节点 | 中央 fan-out（Meta） | reader instance | Streaming Node（实时）+ Query Node（历史，多副本） |
| WAL | n/a | n/a | Pulsar/Kafka broker | **[Woodpecker](../../concepts/woodpecker.md) zero-disk** |
| Recall | 全精度 | 量化失真 | 取 segment 内 index | 取 segment 内 index（含 DISKANN 全精度 re-rank） |
| 增量 | HNSW add 不 delete | rebuild | LSM | **CDC + LSM + function field** |
| Filter | ✗ | ✗ | 5 策略 partition-based | + 多租户 4 层隔离 |

### Milvus 1.x → 2.x 路由层质变（NEW）

| | 1.x | 2.x |
|---|---|---|
| Writer 模型 | single writer | 每 shard 一 Streaming Node |
| Routing 层级 | collection / segment | + vchannel → pchannel → Streaming Node |
| Reader 角色 | reader instance（多副本） | Streaming Node + Query Node 分离 |
| Compute 与 storage | Compute 与 storage 共享 | Streaming Node 实时 + Query Node 历史 + Data Node 离线 三角色分离 |
| WAL | Pulsar/Kafka broker | Woodpecker direct-to-S3 |

### 已知盲区

- **Pinecone pod-based**：仍未覆盖；与 Milvus shared-storage 是另一种 (c) DBMS 路线
- **多 region / 多云路由**：v2.6.x 是 cloud-native 但 cross-region routing 文档未深入
- **DiskANN/SPANN 在 Milvus 中作为索引选项的路由含义**：v2.6.x 把 DiskANN 作为 segment-level index，但 segment + DiskANN 的 routing 是否产生新模式？
- **Milvus 在万亿规模实测**：仍只有 1.x SIGMOD 数字（SIFT1B / 12 节点）

## Cited Pages

- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [concepts/woodpecker.md](../../concepts/woodpecker.md)
