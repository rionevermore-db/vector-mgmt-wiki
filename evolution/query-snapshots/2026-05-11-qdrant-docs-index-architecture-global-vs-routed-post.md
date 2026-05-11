---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下，索引架构 (a) 全局单一索引 vs (c) 层次路由结构的 trade-off？"
date: 2026-05-11
phase: post
ingest-context: qdrant-docs
wiki-pages-total: 65
cited-pages: [systems/qdrant.md, queries/index-architecture-global-vs-routed.md]
cited-count: 2
---

# Post-snapshot (qdrant-docs): index-architecture-global-vs-routed

## TL;DR (delta from ootomo-2023-cagra post)

**Qdrant 是 (c) 层次路由结构的典型 OSS 实现**——Collection → Shard (Raft consensus + consistent hashing + user-defined) → Segment → HNSW index。**3 层结构**比 Milvus 4 层 (Manu disaggregated) 简洁。**对该 query 的影响**：(c) 路由结构现在有清晰双路径——**4b Milvus-style 复杂 disaggregated** vs **4d Qdrant-style Rust shard 简洁路径**。前者 千亿可行 + 多 workload；后者 production sweet spot 中等规模 + 简洁运维。

## Answer

### 与之前 ingest 的演进

| | ootomo-2023-cagra post | **qdrant-docs post (NEW)** |
|---|---|---|
| (c) 路由结构 OSS DBMS 实证 | Milvus segment (4 层) | **+ Qdrant Shard (3 层简洁)** |
| Raft consensus 在 vector DBMS | indirect (Milvus mention) | **+ Qdrant 明示 Raft** |
| User-defined sharding | n/a | **+ Qdrant v1.7.0 (multitenancy 友好)** |

### Qdrant (c) 路由结构层级（NEW）

[per sources/docs/qdrant/distributed_deployment.md + manage-data/]

```
Layer 1: Cluster (Raft consensus on topology + metadata)
   ↓
Layer 2: Collection (named set of points; 统一 distance metric + dim)
   ↓
Layer 3: Shard (consistent hashing or user-defined)
   - shard_number fixed at creation (no auto re-shard in OSS)
   - replication_factor + write_consistency_factor 配置
   - Raft for shard topology
   ↓
Layer 4: Segment (appendable + non-appendable)
   - independent vector + payload + index per segment
   - LSM-style optimizer merges segments
```

→ Qdrant **不像 Milvus 4 层 (proxy + coordinator + query node + worker)**——而是**Cluster + Collection + Shard + Segment 4 层**但**单 binary instance**：每 Qdrant node 既是 query node 又是 storage node，简化运维。

### (c) 路由结构的双路径（NEW）

| 维度 | 4b Milvus-style (disaggregated) | **4d Qdrant-style (Rust shard)** |
|---|---|---|
| 层级 | Proxy / Coordinator / Query Node / Worker / Storage | **Cluster / Collection / Shard / Segment**（单 binary） |
| 部署形态 | k8s 多 service | **单 binary 多 instance** |
| Cluster scaling | Coordinator-managed auto | **Manual sharding (OSS) / Cloud auto rebalance** |
| Consensus | etcd / Coordinator | **Raft (built-in)** |
| Resharding | 复杂 segment 重组 | **Cloud only (v1.13.0+); OSS 需新 collection** |
| Routing 复杂度 | 高（cluster topology + load balance） | **中（per-shard 内 query 已 simple）** |
| Sharding mode | 自动 hash | **automatic + user-defined (v1.7.0+ for multitenancy)** |
| Operational philosophy | "Cloud-native 完整 stack" | **"Rust 单 binary + Raft 简洁"** |

### Qdrant User-Defined Sharding 的独特价值（NEW）

[per sources/docs/qdrant/distributed_deployment.md "User-defined sharding"]

> *Available as of v1.7.0* — Each point is uploaded to a specific shard, so that operations can hit only the shard or shards they need.

→ **Multitenancy 友好**：每 tenant 分配独立 shard，避免 cross-shard query overhead。这与 Milvus partition / Pinecone namespace 同代但更细粒度（shard-level vs partition-level）。

### 在 16 节点 × 1TB 私有云的具体决策（updated）

[per queries/index-architecture-global-vs-routed.md + 推断]

**Qdrant 在此配置下角色**：
- (c) 层次路由：Collection × 12+ shards × 16 nodes
- 每 shard ~500M-1B vectors（千亿 / 192 shards） — single shard HNSW ~30-40 GB RAM fits
- Hot multitenancy: **user-defined sharding** 把 hot tenant 独立 shard
- Memory budget: shards 共享 1 TB RAM；可用 memmap on-disk vector
- Replication factor 2-3 for fault tolerance

但 caveat：**Qdrant 不实证千亿**——理论容量 OK，production benchmark 未公开。

### 已知盲区

- **Qdrant 千亿 (c) 路由实证**：完全空白
- **Qdrant OSS 手动 sharding 在大集群下的 operational burden**：cloud auto rebalance 缓解但 OSS 仍手动
- **Qdrant + cross-shard query coordinator latency**：docs 提"automatic query other nodes"但具体协议未深入
- **Qdrant Raft consensus 在 large cluster (50+ nodes) 性能**：未公开实测
- **HCPS + Qdrant Filterable HNSW + (c) routing**：理论可行未实证

## Cited Pages

- [systems/qdrant.md](../../systems/qdrant.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
