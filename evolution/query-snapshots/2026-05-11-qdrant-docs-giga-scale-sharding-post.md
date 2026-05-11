---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-11
phase: post
ingest-context: qdrant-docs
wiki-pages-total: 65
cited-pages: [systems/qdrant.md, queries/index-architecture-global-vs-routed.md]
cited-count: 2
---

# Post-snapshot (qdrant-docs): giga-scale-sharding

## TL;DR (delta from ootomo-2023-cagra post)

**Qdrant 给私有云部署添加新选项**：Qdrant **Hybrid Cloud / Private Cloud** 提供 OSS code + 商业 control plane 的 deployment 模式——直接面向 16 节点 × 1TB RAM 私有云配置。但 Qdrant **未公开千亿规模实证**——docs 仅提"scalable to billions" 但 cluster sizing recommendation 上限实测 12 shards 跨多 nodes，多数 production 在百万-亿规模。**对该配置的影响**：Qdrant 适合 "中等规模 + 简单运维 + Rust 性能"——千亿规模仍需 Milvus segment / SPANN / DiskANN 类 cluster 架构。

## Answer

### 与之前 ingest 的演进

| | ootomo-2023-cagra post | **qdrant-docs post (NEW)** |
|---|---|---|
| 千亿分片实证 | SPANN @ Bing / FreshDiskANN 1B / Starling 31 segments | **不变（Qdrant 不实证千亿）** |
| 私有云 OSS DBMS 选项 | Milvus 唯一明确 | **+ Qdrant Hybrid / Private Cloud** |
| 私有云 SaaS-control-plane 模式 | 完全空白 | **+ Qdrant Hybrid Cloud (control plane @ Qdrant + data plane @ user)** |

### Qdrant 在 16 节点 × 1TB 私有云配置的可行性（NEW）

[per sources/docs/qdrant/distributed_deployment.md + hybrid-cloud / private-cloud docs]

**Deployment SKU**：

| SKU | 描述 |
|---|---|
| OSS self-host | 全自管 |
| Qdrant Cloud | SaaS（Qdrant 全管） |
| **Hybrid Cloud** | **Control plane @ Qdrant Cloud + Data plane @ user infrastructure (k8s)** |
| Private Cloud | 全私有部署 + Qdrant 商业 control plane |
| Edge | commodity device |

**私有云配置推荐** [per qdrant docs "Choosing the right number of shards"]：

- 推荐 3+ nodes + replication (production gold standard)
- 12 shards 支持 expand 到 1/2/3/6/12 nodes 不 re-shard
- Cloud 自动 rebalance；OSS 需手动 move shards

**但 Qdrant 不实证千亿** [per qdrant docs scale claims]：
- "scalable to billions" 是 docs 用语
- 具体千亿 cluster sizing 未公开
- 12 shards × 16 nodes = 192 shards——理论上承载千亿 (10^11 / 192 ≈ 500M per shard)，但 single shard 500M HNSW 内存需 ~30-40 GB——fits in 1 TB but tight

### 千亿配置决策（updated）

| 决策 | 推荐 | Qdrant 是否合适 |
|---|---|---|
| **千亿 + read-heavy + production** | (c) 层次路由 + per-segment DiskANN/SPANN/Starling | **Qdrant 不直接** |
| 中等规模 (10亿-百亿) + Rust 性能 + 简洁运维 | **Qdrant + 12 shards × 多 node** | **✓ 可行** |
| 中等规模 + 混合云 | **Qdrant Hybrid Cloud + 12 shards** | **✓ 推荐** |
| 千亿 + Hybrid cloud | n/a | 未实证 |

→ Qdrant **不是千亿的 default**——主流千亿仍是 Milvus / DiskANN / SPANN 路径。但 Qdrant 提供 **中等规模 + 简洁 Rust + flexible SKU** 的独特位置。

### 与 Milvus 的私有云 production trade-off（NEW）

| 维度 | Milvus | **Qdrant** |
|---|---|---|
| 千亿规模实证 | ✓ (Salesforce / NVIDIA 等) | ✗（中等规模实证为主） |
| 多 index_type | 多 (HNSW/IVF*/DISKANN/CAGRA/SPARSE) | **仅 HNSW** + Sparse |
| Cloud-native disaggregated | ✓ (Manu 复杂架构) | **Rust 单 binary + shard** |
| Operational complexity | 高（4 层 disaggregated） | **低（简洁路径）** |
| Hybrid Cloud SKU | Zilliz Cloud (类似) | **Qdrant Hybrid Cloud (明示)** |
| Private Cloud | 自管 | **Qdrant Private Cloud（商业 control plane + 私有 data plane）** |

→ **Qdrant 在 SKU flexibility 上独特**——Hybrid Cloud 模式（control plane @ Qdrant + data plane @ user）填补 OSS self-host 与 SaaS 之间的 gap。

### 已知盲区

- **Qdrant 千亿规模 production benchmark**：完全空白
- **Qdrant Hybrid Cloud 实际部署案例**：docs 仅 high-level，具体客户 + 规模不公开
- **Qdrant vs Milvus head-to-head 千亿 benchmark**：双方都不公开
- **Qdrant + multi-region replication**：docs 提及但深度有限
- **Qdrant Edge 部署**：commodity device 上的 ANN 限制未深入

## Cited Pages

- [systems/qdrant.md](../../systems/qdrant.md)
- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
