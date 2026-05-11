---
query-key: embedding-update-handling
query: "当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？"
date: 2026-05-11
phase: post
ingest-context: adams-2025-distributedann
wiki-pages-total: 69
cited-pages: [systems/distributedann.md, systems/freshdiskann.md, systems/spfresh.md, concepts/freshvamana.md, concepts/lire.md]
cited-count: 5
---

# Post-snapshot (adams-2025-distributedann): embedding-update-handling

## TL;DR (delta from turbopuffer-docs post)

**DistributedANN paper 未提及 embedding model 升级 / streaming updates 路径**——focus 在 serving 大 static graph + 用 graph stitching for build。**关键 NEW**：Bing 当前 production architecture (DistributedANN, 50B per slice) 与 streaming sibling (FreshDiskANN/SPFresh) 是**两个独立分支**——DistributedANN focus on distributed serving, streaming axis 由其他 4 代 MSR 论文 (FreshDiskANN/SPFresh) 处理。Bing 怎么协调 distributed graph + streaming + embedding model upgrade 在公开论文中**仍是 open**。Algorithm 层跨模型 zero coverage 持续未变。

## Answer

### 与之前 ingest 的演进

| | turbopuffer-docs post | **adams-2025-distributedann post (NEW)** |
|---|---|---|
| 工程层 model migration tool | Qdrant Aliases + Vespa AppPkg + Turbopuffer copy_from | **不变** |
| Distributed graph build strategy | wiki 未明示 | **+ DistributedANN graph stitching (clustered partition + neighborhood union)** |
| 算法层跨模型 compatibility | 仍 zero coverage | **不变** |
| Bing model upgrade workflow | 未明示 | **仍未明示 (DistributedANN paper focus on serving)** |

### DistributedANN graph stitching (NEW build strategy)

[per adams-2025-distributedann §3]

> While it is possible to insert vectors into an DISTRIBUTEDANN graph by this procedure, it would require significantly more computation than building an equivalently sized partitioned graph, because the partitioned approach only needs to search in one smaller partition for each insertion. To reduce the graph construction cost, we employ a graph stitching approach similar to the one described in (Subramanya et al., 2019).

**Graph stitching**:
1. Clustered partition build (Wang 2021): 切 50B 数据为 203 partition × ~200M
2. Build DiskANN graph in each partition independently
3. Union neighbors: vector 在多 partition 出现 → 取 union of neighbor lists
4. Head index: BFS top layers of stitched union → conventional sharded in-mem ANN

**Build vs incremental insert**:
- Incremental insert into single 50B graph: too expensive (每 insert 需 search 50B graph)
- Graph stitching: 每 vector 仅 search 1 partition (200M) → much faster
- Quality 略低 ("sufficient to get good results") 但 build 时间大幅缩短

**Embedding model upgrade in DistributedANN context (推断)**:
- Re-embed 50B vectors with new model
- Re-cluster (semantic partition changes)
- Rebuild via graph stitching
- **Online migration unclear** — paper 不讨论 graph swap / blue-green for 50B-scale model upgrade

### 处理决策表（updated 2026-05-11 post adams-2025）

| 场景 | 推荐方案 |
|---|---|
| Per-tenant 独立 embedding + per-tenant rolling migration | Turbopuffer namespace + copy_from_namespace |
| 跨 embedding model 升级 + atomic switch + 集群行为整体变更 | Vespa application package |
| 跨 embedding model 升级 + collection-level 简单切换 | Qdrant Collection Aliases |
| 同 doc 多 model embedding 共存 + complex ranking | Vespa multi-tensor field |
| 同 doc 多 embedding + 简单查询 | Weaviate named vectors |
| 大规模 stale segment 标记 + 后台 reindex | Milvus / Vespa segment |
| **超大 distributed graph (>10B) + model upgrade** | **DistributedANN graph stitching rebuild (单 deployment 整体 rebuild)** |
| 跨 model semantic preserve algorithm | 仍未有 production——等 SIGMOD 2026 |

### MSR 谱系内 streaming 与 distributed 的 axis 分离（NEW insight）

[per adams-2025-distributedann + 谱系 5 论文]

- **DiskANN 2019**: static + single-node
- **SPANN 2021**: static + cluster + 分布式 (partition + replica)
- **FreshDiskANN 2021**: **streaming + single-node (graph path)**
- **SPFresh 2023**: **streaming + single-node (cluster path)**
- **DistributedANN 2025**: static + **distributed (single graph)**

→ **Bing production 当前是 DistributedANN (static + distributed)**——streaming axis 在 Bing 主力 production 实际状态 unknown. 可能：
- Periodic full rebuild via graph stitching (every X days)
- Hybrid: DistributedANN + FreshDiskANN local layer
- 论文未公开

**Embedding model upgrade in DistributedANN**:
- Periodic rebuild path is natural ("rebuild every X model release")
- Online migration path未公开

### 算法层跨模型方案——frontier 仍未关闭

不变. Talk 当天 SIGMOD 2026 live demo 仍是唯一 algorithm-level answer.

### 已知盲区

- **Bing model upgrade workflow**: 50B distributed graph 升级 model 的具体 production steps 不公开
- **DistributedANN + streaming**: §5.1 future direction 提 "HDD tiering for cold tenants" 暗示 streaming + distributed 是 open work
- **Graph stitching quality**: §3 "quality is lower than fully incremental but sufficient" 未量化 gap
- **Online vs batch rebuild**: Bing 是否 hot-swap 整个 50B distributed graph? 论文不讨论

## Cited Pages

- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/freshdiskann.md](../../systems/freshdiskann.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [concepts/freshvamana.md](../../concepts/freshvamana.md)
- [concepts/lire.md](../../concepts/lire.md)
