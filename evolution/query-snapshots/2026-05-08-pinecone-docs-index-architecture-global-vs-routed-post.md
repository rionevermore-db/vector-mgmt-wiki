---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下索引架构选 (a) 全局单一索引 还是 (c) 层次路由结构？"
date: 2026-05-08
phase: post
ingest-context: pinecone-docs
wiki-pages-total: 36
cited-pages: [queries/index-architecture-global-vs-routed.md, systems/pinecone.md, concepts/pinecone-pod-based.md, concepts/pinecone-serverless-slabs.md, systems/milvus.md, systems/spfresh.md, systems/spann.md]
cited-count: 7
---

# Post-snapshot (pinecone-docs): index-architecture-global-vs-routed

## TL;DR (delta from xu-2023-spfresh post)

**直接关闭 query archive 明确 flag 盲区**："Pinecone pod-based 架构未覆盖 / Milvus segment 未覆盖" 现已全部 ingest。Pinecone 是 wiki 内**唯一商业 SaaS** (c) 路由形态——同时支持 pod-based 静态 routing（legacy）和 serverless slab + adaptive routing（current）。**namespace 是核心 isolation 单元**——每 query 仅扫一个 namespace，强制 multi-tenant 隔离。

## Answer

### Pinecone 作为 (c) 层次路由 SaaS 形态（NEW，关闭 query archive flag）

[per queries/index-architecture-global-vs-routed.md "已知盲区"]：

> "Pinecone pod-based 架构：商业云向量 DB 主流形态，wiki 无对应 page"

→ **本次 ingest 关闭此 flag**。

### Pinecone 两代 (c) 路由形态

[per concepts/pinecone-pod-based.md + pinecone-serverless-slabs.md]

**第一代 Pod-based**：
- 用户预选 pod 类型 + 数量 + replica → 路由到固定 pod
- Replica 自动跨 zone（多 zone 冗余）
- 单 region；scaling 手动
- 与 Faiss IVF + 自实现路由层相似

**第二代 Serverless slab**：
- 写读路径完全解耦（独立 scale）
- Query routers 决定 slab 命中（路由层透明）
- 多 region 支持
- 索引 slab 在 object storage（与 SPANN posting list / Milvus segment 同源）
- **Adaptive indexing**——slab 大小决定索引算法

### (c) 形态完整对比表（updated）

| (c) 系统 | 路由层 | 路由 mutability | 索引算法 lifecycle |
|---|---|---|---|
| Faiss IVF + HNSW coarse | HNSW centroids | freeze + 周期 retrain | 静态 |
| Meta 1.5T Faiss mmap | 10M centroids HNSW | freeze | 静态 |
| SPANN | SPTAG centroids | 周期 rebuild centroid | 静态 |
| Milvus 1.x segment | segment-level coarse | LSM merge | segment 内静态 |
| Milvus 2.x（vchannel） | shard 路由 + Streaming Node | shard 不 fluid，segment 内 LSM | 静态 |
| SPFresh | SPTAG centroids + 持续 split/merge | **In-place LIRE**（routing-as-mutable） | 静态 |
| **Pinecone Pod-based** | **pre-configured pod** | freeze | 静态 |
| **Pinecone Serverless slab**（NEW） | **slab + namespace** | **slab merge + adaptive** | **adaptive across lifecycle** |

→ Pinecone Serverless 是 wiki 现有 source 中**首个 routing + index 都演化**的系统——slab 既是路由单元也是 index 单元，两者同时随 lifecycle 演化。

### Namespace 作为 (c) 路由的 isolation 单元（NEW）

[per systems/pinecone.md "数据模型" + sources/docs/pinecone/llms-full.txt §Implement multitenancy]

Pinecone 强制每 query 仅扫一个 namespace：
- 路由层先按 namespace 分流
- 每 namespace 独立 slab + memtable 集合
- 跨 namespace 查询**不支持**（必须应用层合并）

→ namespace 是天然的 multi-tenant 隔离 + (c) 路由分桶单元。

**与 Milvus partition_key 对比**：
- Milvus：partition_key 路由到 partition，但**同 collection 内可跨 partition 查**（option）
- Pinecone：namespace 是 hard boundary，**不可跨**

### 对 query archive 的回应

[queries/index-architecture-global-vs-routed.md] 当时归档说"Pinecone pod-based 未覆盖"。**本次 ingest 不仅关闭此 flag，还引入 Pinecone serverless slab 新形态**——比 query archive 写时预期的更深。

但 **archive 本身不修改**（per CLAUDE.md "queries/ 是冻结的快照"）——本 post-snapshot 即为 update 答案。

### 与之前 ingest post 的演进

| | wang-2021 post | milvus-docs post | xu-2023-spfresh post | **pinecone-docs post (NEW)** |
|---|---|---|---|---|
| (c) 系统数 | 3（Meta/SPANN/Milvus 1.x） | + Milvus 2.x | + SPFresh | **+ Pinecone（pod + serverless）** |
| 路由 mutability | 主要 freeze | 部分 LSM | LIRE 实证 | **+ slab adaptive** |
| 商业 SaaS 视角 | 缺失 | 缺失 | 缺失 | **填补** |
| 闭源黑盒 trade-off | 未讨论 | 未讨论 | 未讨论 | **首次显式** |

### 已知盲区（仍未覆盖）

- **跨 region (c) 路由细节**：Pinecone docs 提 multi-region 但 failover / consistency 未深入
- **Pinecone DRN multi-namespace 支持**：当前单 namespace only，未来支持但路由细节 docs 未给
- **Pinecone vs Milvus segment routing 实测**：双方都不公开
- **2026 SIGMOD 跨 model 整合**：talk demo

## Cited Pages

- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
- [systems/pinecone.md](../../systems/pinecone.md)
- [concepts/pinecone-pod-based.md](../../concepts/pinecone-pod-based.md)
- [concepts/pinecone-serverless-slabs.md](../../concepts/pinecone-serverless-slabs.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/spann.md](../../systems/spann.md)
