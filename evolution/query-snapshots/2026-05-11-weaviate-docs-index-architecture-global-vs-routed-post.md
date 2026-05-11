---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下，索引架构 (a) 全局单一索引 vs (c) 层次路由结构的 trade-off？"
date: 2026-05-11
phase: post
ingest-context: weaviate-docs
wiki-pages-total: 66
cited-pages: [systems/weaviate.md, queries/index-architecture-global-vs-routed.md]
cited-count: 2
---

# Post-snapshot (weaviate-docs): index-architecture-global-vs-routed

## TL;DR (delta from qdrant-docs post)

**Weaviate 强化 (c) 层次路由 OSS 实证**——同 Qdrant / Milvus 都是 (c) 路径。Weaviate 路由结构: Collection → Shard → Segment → HNSW + indexes。**OSS DBMS 选 (c) 已经是 unanimous default**（Milvus segment / Qdrant Shard / Weaviate Shard）；(a) global 仅 single-server 学术系统 (DiskANN 1B SIFT) 或 mmap-based extreme (Faiss 1.5T)。**核心 NEW**：Weaviate **collection-level multi-tenancy** 在 (c) 路由 leaf 提供更细粒度 isolation——比 Qdrant Tenant Index (payload-field-level) 更高粒度。

## Answer

### 与之前 ingest 的演进

| | qdrant-docs post | **weaviate-docs post (NEW)** |
|---|---|---|
| (c) OSS DBMS 实证 | Qdrant + Milvus | **+ Weaviate (collection → shard → segment → HNSW)** |
| (c) multi-tenancy 细粒度 | Qdrant Tenant Index (payload-field-level) | **+ Weaviate collection-level multi-tenancy** (更高粒度) |
| (a) global 实证 | Faiss 1.5T mmap, DiskANN 1B SIFT | 不变 |

### Weaviate (c) 路由结构（NEW）

[per sources/docs/weaviate/llms.txt §Architecture/Scaling + Multi-tenancy]

```
Layer 1: Cluster (auto-scaling Cloud / manual OSS)
   ↓
Layer 2: Collection
   - 命名 set of objects + vectors + indexes
   - Multi-tenancy config (collection-level)
   - RBAC permissions (collection-level)
   ↓
Layer 3: Shard
   - Auto-sharded by Cloud
   - Manual sharding by OSS user
   - Replica movement for fault tolerance
   ↓
Layer 4: Tenant (optional)
   - When multi_tenancy_config = enabled
   - Per-tenant isolated storage + isolated HNSW
   ↓
Layer 5: Segment + Indexes
   - LSM store + WAL
   - HNSW + RQ8 (vector)
   - BlockMaxWAND BM25 (text)
   - Roaring bitmaps (set filter)
   - Bit-sliced range bitmaps (range filter, opt-in)
```

→ **比 Qdrant Shard → Segment 更深一层 (Tenant 在 Shard 与 Segment 之间)**——Weaviate 把 multi-tenancy 抬升为 first-class architectural primitive。

### Multi-tenancy 在 (c) 路由的角色（NEW）

[per sources/docs/weaviate/llms.txt §Multi-tenancy + Best Practices]

| 系统 | Multi-tenancy 实现 | 粒度 |
|---|---|---|
| Milvus | Database / Collection / Partition / Partition-key | 多级 |
| Pinecone | Namespace | namespace level |
| Qdrant | Payload partitioning + Tenant Index (payload-field) + user-defined sharding | payload-field level |
| **Weaviate** | **Collection-level config + Tenant primitive in 路由 hierarchy** | **collection level (independent storage per tenant)** |

→ Weaviate **per-tenant 完全独立 storage + index**——比其他 OSS 系统的 multi-tenancy 更"hard isolation"。SaaS production friendly。

### OSS DBMS (c) 路由架构对比（updated）

| 系统 | (c) 路由层级 | Consensus | Sharding mode |
|---|---|---|---|
| Milvus (Manu) | Proxy + Coordinator + Query Node + Worker + Storage (4 层 disaggregated) | etcd (assumed) | auto |
| Qdrant | Cluster + Collection + Shard + Segment (单 binary 多 instance) | **Raft built-in** | automatic + user-defined |
| **Weaviate** | **Cluster + Collection + Shard + Tenant + Segment** | (未在 llms.txt 明示, 可能 Raft) | Cloud auto + OSS manual |

→ Weaviate **5 层 (with Tenant)** vs Qdrant **4 层** vs Milvus **4 层 disaggregated**——Weaviate 在 OSS DBMS 中**最深的 multi-tenancy 集成**。

### 在 16 节点 × 1TB 私有云的具体决策（updated）

- (c) 层次路由 + per-collection multi-tenancy + Weaviate Shard → **适合 SaaS 多租户应用**
- BYOC SKU 允许私有云完全自管
- 千亿规模实证 Weaviate 不公开——主流仍 Milvus / DiskANN-class

### 已知盲区

- **Weaviate consensus protocol**：llms.txt 未明示（Raft? Paxos?）；与 Qdrant Raft 对比未做
- **Weaviate per-tenant HNSW vs single shared HNSW**：多租户 storage 独立但 HNSW 共享/独立未明示
- **Weaviate sharding 千亿 + multi-tenant 千 tenant 实证**：完全空白
- **(c) 路由 + iterator + RM (VBASE) + Weaviate**：理论可行未实证

## Cited Pages

- [systems/weaviate.md](../../systems/weaviate.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
