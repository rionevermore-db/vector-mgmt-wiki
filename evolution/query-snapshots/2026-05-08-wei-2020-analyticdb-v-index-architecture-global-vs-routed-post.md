---
query-key: index-architecture-global-vs-routed
query: "千亿/万亿规模下索引架构选 (a) 全局单一索引 还是 (c) 层次路由结构？"
date: 2026-05-08
phase: post
ingest-context: wei-2020-analyticdb-v
wiki-pages-total: 42
cited-pages: [queries/index-architecture-global-vs-routed.md, systems/analyticdb-v.md, concepts/vgpq.md, systems/milvus.md, topics/disk-vs-memory-ann.md]
cited-count: 5
---

# Post-snapshot (wei-2020-analyticdb-v): index-architecture-global-vs-routed

## TL;DR (delta from guo-2022-manu post)

**ADBV 是另一种 (c) 路由形态**——OLAP-style cluster-based partitioning：数据在 ingest 时按向量 cluster centroid 分区（不是 hash/range），query 仅 dispatch 到 closest N 分区（user query hint）。**10×-100× throughput 提升 on 1000+ 节点 large-scale**。这与 [Milvus segment](../../systems/milvus.md) / [Pinecone namespace](../../systems/pinecone.md) / [SPANN posting](../../systems/spann.md) 的"系统层路由"不同——ADBV 是**关系 OLAP 的"按 vector 列分区"**——partition 决策直接基于向量几何。

## Answer

### ADBV cluster-based partitioning 形态（NEW）

[per systems/analyticdb-v.md "Clustering-based partitioning"]

```
传统关系 DB partition: hash(id) / range(date) / list(category)
ADBV 加 partition: ClusterBasedPartition(feature_vector)
   → 数据 ingest 时 k-means(feature) → 分到 closest centroid
   → Query 时 user 给 hint：dispatch 到 closest N 分区
```

**关键差异**：
- 传统关系 partition 维度是**已知的标量列**
- ADBV cluster-based partition 维度是**多维 vector geometry**

→ Partition 与"何处放数据 + 何处查"绑定为同一决策（与 vector geometry 对齐）。

### (c) 路由形态完整对比（updated）

| (c) 系统 | 路由层 | 路由 mutability | partition 决策依据 |
|---|---|---|---|
| Faiss IVF + HNSW coarse | HNSW centroids | freeze | 训完冻结 |
| Meta 1.5T Faiss mmap | 10M centroids HNSW | freeze | 训完冻结 |
| SPANN | SPTAG centroids | 周期 rebuild | 训完冻结 |
| Milvus 1.x segment | segment-level coarse | LSM merge | segment 内 index 选 |
| Milvus 2.x（vchannel） | shard 路由 + Streaming Node | shard 不 fluid | hash(primary key) |
| SPFresh | SPTAG centroids + 持续 split/merge | **In-place LIRE**（routing-as-mutable） | persistent NPA |
| Pinecone Pod-based | pre-configured pod | freeze | hash |
| Pinecone Serverless slab | slab + namespace | **slab merge + adaptive** | namespace + slab merge |
| **AnalyticDB-V** | **ClusterBasedPartition (vector centroid)** | **ingest-time + 周期 re-cluster** | **vector geometry** |

→ ADBV 是**首个 partition 决策直接基于 vector geometry** 的系统（其他都按 metadata / hash / 系统层 segment）。

### ADBV 实测的 (c) 路由提升（NEW）

[per benchmarks/analyticdb-v-vs-twostep.md "结果 2"]

SIFT1B_512p（512 cluster-based partitions）：
- 不剪枝（扫全 512 partition）：baseline
- 剪到 closest 3 partition：**95% recall, 100×+ throughput**
- 剪到 closest 5：97% recall
- 剪到 closest 10：99% recall

→ 千亿规模下 cluster-based partitioning 是工程关键——直接决定多机 dispatch 数。

### ADBV 与 SPANN cluster-based partitioning 的对比（NEW）

| | SPANN | ADBV |
|---|---|---|
| Partition 单元 | posting list | partition (节点级) |
| Partition 决策 | balanced clustering 训 centroids | k-means 在 ingest 时分 |
| Partition 跨节点 | 单节点上 SSD posting | **跨节点（每节点多 partition）** |
| Query 时 dispatch 范围 | 单节点内 K posting | **节点级 N 分区**（user query hint） |
| 边界处理 | closure clustering 复制边界向量 | **直接接受 boundary loss**（recall 用 N 调整） |
| 100× throughput 实测 | n/a | **✓ on 1000+ node** |

### 关闭 query archive 的"Pinecone pod-based 未覆盖"flag 后的 (c) 路由完整性

[per queries/index-architecture-global-vs-routed.md "已知盲区"]

之前 query archive 列的盲区已全部 ingest：
- ✅ Pinecone pod-based 架构（pinecone-docs ingest）
- ✅ Milvus segment 模型（wang-2021-milvus + milvus-docs + guo-2022-manu ingest）
- ✅ DiskANN graph + SSD（subramanya-2019-diskann ingest，更早）
- ✅ SPANN IVF + SSD（chen-2021-spann ingest，更早）
- **✅ ADBV cluster-based partitioning**（本次 ingest，OLAP-extended (c) 路由形态）

→ wiki (c) 路由覆盖**首次完整**：算法系统 + DBMS + SaaS + OLAP-extended 四种形态都 ingest。

### 与之前 ingest 的演进

| | guo-2022-manu post | **wei-2020-analyticdb-v post (NEW)** |
|---|---|---|
| (c) 形态数 | 4 | **5**（+ ADBV） |
| Vector-geometry partition | 未出现 | **ADBV 首次** |
| (c) 100×+ throughput 实测 | Manu 2-10× scaling | **ADBV cluster-pruning 100×+** |
| OLAP-extended (c) | 未出现 | **首次出现** |

### 已知盲区

- **ADBV cluster-based partition 在 update 下的 rebalance**：论文 §3.3 末尾建议 re-cluster 但未深入触发条件 / 成本
- **千亿规模 ADBV cluster-based partitioning 实测**：仅 SIFT1B / Deep1B 测试；13B production 但 partition 数与 latency 未细测
- **多 region (c) 路由**：所有 wiki source 都未深入 cross-region (c) 路由

## Cited Pages

- [queries/index-architecture-global-vs-routed.md](../../queries/index-architecture-global-vs-routed.md)
- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
- [concepts/vgpq.md](../../concepts/vgpq.md)
- [systems/milvus.md](../../systems/milvus.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
