---
title: In-Place vs Out-of-Place Updates（向量索引更新策略）
type: topic
sources: [xu-2023-spfresh, chen-2021-spann, subramanya-2019-diskann, wang-2021-milvus, douze-2024-faiss-library, singh-2021-freshdiskann, qdrant-docs]
related: [../systems/spfresh.md, ../systems/freshdiskann.md, ../systems/spann.md, ../systems/diskann.md, ../systems/milvus.md, ../systems/faiss.md, ../systems/pinecone.md, ../systems/analyticdb-v.md, ../systems/pase.md, ../systems/starling.md, ../systems/qdrant.md, ../concepts/lire.md, ../concepts/freshvamana.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/vamana.md, ../concepts/product-quantization.md, ../concepts/pinecone-serverless-slabs.md, ../concepts/delta-consistency.md, ../concepts/vgpq.md]
created: 2026-05-07
updated: 2026-05-11 (Qdrant Collection Aliases)
---

# In-Place vs Out-of-Place Updates

**TL;DR**: ANN 索引的 update 策略是 system-level 选择，与 index type 部分独立。**Out-of-place**（DiskANN streamingMerge / Faiss codebook freeze + global rebuild / Milvus LSM merge）以**周期成本换日常稳定**；**In-place**（[SPFresh](../systems/spfresh.md) [LIRE](../concepts/lire.md) cluster-path / [FreshDiskANN](../systems/freshdiskann.md) [FreshVamana](../concepts/freshvamana.md) graph-path）以**持续低成本换无 rebuild 高峰**。**关键状态变更（2026-05-09）**：之前 wiki 视 graph-path in-place update 为"开放问题"——FreshDiskANN（arXiv 2021，但 wiki 直到 2026-05-09 才 ingest）证明 **α-RNG property 是 graph fresh-ANNS 必要条件**——HNSW/NSG 因隐式 α=1 仍未解，Vamana 因显式 α 参数得以延伸为 FreshVamana。**两条 in-place 路径并存**：cluster-path（SPFresh，~4 GB RAM for 1B）vs graph-path（FreshDiskANN，~128 GB RAM for 1B）——内存预算决定路径选择。

## 问题陈述

ANN 索引的两类 update 模型：

- **Out-of-place**：用户更新累积到 secondary delta index → 周期"全局 rebuild"合并到 main index → 旧 index 析构
- **In-place**：用户更新直接修改主 index 数据结构 → 后台维护索引质量（split/merge/reassign 或 graph edge 维护）

**核心 trade-off**：
- Out-of-place：日常稳定，但周期 rebuild 资源峰值极高（1B 量级 1100 GB DRAM + 32 cores × 2 天 [per benchmarks/spfresh-vs-diskann-spann-update.md]），rebuild 期间 latency 跳水
- In-place：无 rebuild 高峰，但 index 数据结构持续承担更新成本，需要复杂 concurrency control + 形式化质量证明（[LIRE](../concepts/lire.md) NPA 收敛证明 / [FreshVamana](../concepts/freshvamana.md) α-RNG property）

## 工业方案对比

| 方案 | 路线 | 索引类型 | 更新粒度 | 全局 rebuild 频率 | 资源峰值 | 数据漂移适应 |
|---|---|---|---|---|---|---|
| [DiskANN](../systems/diskann.md) `streamingMerge` | **Out-of-place** | Graph (Vamana) | 周期 batch | 1B SIFT 每 ~1-2 周（实测） | 1100 GB + 32 cores × 2d 或 64GB + 16 cores × 5d | rebuild 后才修复 |
| **SPANN+**（modified [SPANN](../systems/spann.md)） | In-place append-only | IVF | 单向量 | 不 rebuild（实验对照） | 极低 | **不修复**（partition skew 累积，P99.9 4ms→>10ms over 100d） |
| [Faiss](../systems/faiss.md) IVFPQ | **Out-of-place** | IVF + PQ | 周期 batch | 完全 retrain codebook | 同 train 全套 | codebook freeze 后**完全不修复** |
| [HNSW](../concepts/hnsw.md) (Faiss / nmslib) | **Out-of-place** | Graph | add ✓ delete ✗ | rebuild 时全图重建 | 同 train | 仅 add 不 delete；图退化无修复（α=1 implicit） |
| [NSG](../concepts/nsg.md) | **Out-of-place** | Graph | 不支持任何增量 | rebuild | 同 train | rebuild 后才修复（α=1 implicit） |
| Vearch | In-place | Cluster (in-memory) | 单向量 + tombstone | 周期 GC + rebuild | 中 | tombstone GC，不 rebalance |
| ADBV / 早期 [Milvus](../systems/milvus.md) | **Out-of-place** | 多种 | delta index 累积 | 周期 merge | 中（rebuild 期 search 还要 main + delta） | rebuild 后才修复 |
| Milvus v2.x LSM | **Out-of-place + LSM** | Segment-based | 单向量 → segment | 后台 tiered merge | 中 | merge 仅按 size，不按数据分布 |
| **[SPFresh](../systems/spfresh.md) LIRE** | **In-place + 主动 rebalance（cluster path）** | IVF（[SPANN](../systems/spann.md) base） | 单向量 + 邻近触发 | **完全不需要** | **持续 10 GB + 2 cores**（1B SIFT） | **NPA 主动维持**，P99.9 ~4ms 稳定 100 天 |
| **[FreshDiskANN](../systems/freshdiskann.md) FreshVamana** | **In-place + StreamingMerge（graph path）** | Graph ([Vamana](../concepts/vamana.md) base, α=1.2) | 单向量 + 周期 merge | **不需要全 rebuild**（StreamingMerge 仅处理 change set） | **持续 ~128 GB**（1B SIFT），merge 时 5.25× faster than full rebuild | **α-RNG 主动维持**，50 cycles × 5%/10%/50% recall 稳定 |
| **[Pinecone Serverless slab](../systems/pinecone.md)** | **LSM-style + adaptive indexing** | Slab in object storage | 单向量 → memtable → flush → slab merge | **slab merge** 中（非全 rebuild） | docs 未公开数字（auto） | **adaptive**：merge 时自动从 fast indexing 升级到 sophisticated indexing |
| **[Manu (Milvus 2.x)](../systems/milvus.md)** | **Stream indexing + delta consistency τ** | Segment（512 MB default） + slice（10K vec temp IVF-FLAT） | growing segment 临时 index → sealed segment 完整 index | 周期 stream indexing（非全 rebuild） | NeurIPS 2021 winner SSD index | **delta consistency τ** 让 user 调"过期容忍度" |
| **[AnalyticDB-V (ADBV)](../systems/analyticdb-v.md)** | **Lambda streaming + batching 双索引** | Streaming HNSW (in-memory) + Batching VGPQ (Pangu) | 新数据进 streaming HNSW；周期 async merge 到 batching VGPQ + 重建 VGPQ | **Async merge** 中（不阻塞 query） | OLAP DB 路径 + 4 plan CBO | streaming/batching 用**两种不同算法** (HNSW vs VGPQ) |
| **[PASE](../systems/pase.md)** | **PG 自身 OLTP transaction + WAL** | IVFFlat / HNSW page chain in PG | 直接走 PG INSERT/UPDATE/DELETE；ACID 保证 | **PG 自身 vacuum + WAL replay** | OLTP RDBMS 路径 | **PG 内核自带的 MVCC 处理**；缺点是受限于 PG 单实例（million-scale） |
| **[Starling](../systems/starling.md)** | **Static disk index + 周期 merge（FreshDiskANN-derived）** | Graph (any base + block shuffling) | §7 Discussion 提"static disk + dynamic in-memory + 周期 merge" | 周期 block shuffling 重跑 | segment-level（2GB RAM + 10GB disk per segment） | 论文未深入实证 update 模式 |

## In-place 的两条路径（NEW 2026-05-09）

[per concepts/lire.md, concepts/freshvamana.md]

In-place fresh-ANNS 现在有**两条工业证实路径**——按 index 数据结构分：

### Cluster Path: [SPFresh](../systems/spfresh.md) + [LIRE](../concepts/lire.md)

[xu-2023-spfresh, SOSP 2023, Microsoft Research]

- 索引基础: SPANN inverted file
- 关键性质: **NPA (Nearest Partition Assignment)** 必要条件
- 增量机制: 5 操作（split / merge / reassign / split-recursive / split-cascading）
- 触发频率: 仅 **0.4% inserts** 触发 rebalance
- 内存预算: ~4 GB for 1B SIFT
- 实证: 100 days × 1% daily update, P99.9 ~4 ms 稳定

### Graph Path: [FreshDiskANN](../systems/freshdiskann.md) + [FreshVamana](../concepts/freshvamana.md)

[singh-2021-freshdiskann, arXiv 2021, Microsoft Research India + CMU]

- 索引基础: Vamana graph
- 关键性质: **α-RNG property**（α > 1 RobustPrune）必要条件
- 增量机制: FreshVamana Insert + Lazy Delete + Batch Consolidation
- 触发频率: 持续 inserts/deletes，merge 在 TempIndex 大小达阈值时触发
- 内存预算: ~128 GB for 1B SIFT
- 实证: 50 cycles × 5%/10%/50% change, recall 95%+ 稳定

### 对比

| | SPFresh / LIRE (cluster path) | **FreshDiskANN / FreshVamana (graph path)** |
|---|---|---|
| 论文年份 | 2023 SOSP | **2021 arXiv** (preprint) |
| 索引 base | SPANN inverted file | Vamana graph |
| 必要条件 | 2 NPA conditions | α-RNG property (α > 1) |
| 增量算法 | 5 operations | Insert + Lazy Delete + Batch Consolidate |
| 内存预算（1B） | **~4 GB** | ~128 GB |
| Update cost | 极低 (per-cluster locality) | 中 (StreamingMerge sequential SSD pass) |
| 实证 stability | 100 days × 1% daily | 50 cycles × 5%/10%/50% |
| Recall | 95%+ | 95%+ |
| 团队 | Microsoft Research（Yuming Xu et al.） | Microsoft Research India + CMU（Singh + Subramanya et al.） |

→ **内存预算决定路径选择**：4 GB vs 128 GB 是 30× 差距；cluster path 更经济，graph path 更高 recall + 更快 search。

## 三种"In-Place"的差别

[per concepts/lire.md, systems/spfresh.md, concepts/freshvamana.md]

1. **不修复型 in-place**（SPANN+ / Vearch）：单向量插入到现有 partition + tombstone delete；不 rebalance partition。代价：partition skew 累积导致 latency 与 recall 缓慢退化
2. **修复型 in-place（cluster path）**：SPFresh LIRE。插入触发 split + 邻近 NPA 检查 + reassign；持续维持 well-partitioned 性质
3. **修复型 in-place（graph path，NEW 2026-05-09）**：FreshDiskANN FreshVamana。插入用 α=1.2 RobustPrune；周期 batch consolidate deletes；StreamingMerge to LTI on SSD
4. **DBMS-style segment merge**（Milvus LSM）：实际是 out-of-place 的"周期化"——后台 tiered merge 替代周期 rebuild，但 segment 内 index 仍然冻结

只有 (2) (3) 能同时保 latency 稳定 + 资源较低 + 不全 rebuild。

## 为什么 HNSW / NSG 仍没有 in-place 解（updated）

[per concepts/freshvamana.md "α-RNG Property" + singh-2021-freshdiskann §3.3 Fig 1]

之前 wiki 解释（[xu-2023-spfresh §1, §2.1]）说"graph 高维 in-place 太贵"——这是**部分正确但不完整**。FreshDiskANN [Singh 2021] 揭示真因：

- **HNSW 隐式 α=1**——aggressive RobustPrune 创建极稀疏 graph
- **NSG 隐式 α=1**——同样问题
- **Vamana 显式 α 参数**——可设 α > 1（实测 α=1.2 work）

→ **α=1 是 fresh-ANNS 失败的根因**——不是 graph 算法本身的问题，而是 pruning 哲学问题。

理论上 HNSW / NSG 加入 α-augmented RobustPrune 都可以变成 fresh-ready（Microsoft Research 论文 §3.3 实测：HNSW Delete Policy A/B 失败的根因是 α=1）——但 hnswlib / nmslib / NSG-original 工业实现都没改。

| 算法 | α 参数 | Fresh-ANNS 状态 |
|---|---|---|
| HNSW | implicit α=1 | ✗（理论上加 α=1.2 可解，未做） |
| NSG | implicit α=1 | ✗（同上） |
| Vamana | explicit α | ✓ via FreshVamana α=1.2 |

## 三个明确的"经济阈值"

[per benchmarks/spfresh-vs-diskann-spann-update.md, benchmarks/freshdiskann-streaming-sift800m.md]

| 决策点 | 触发 |
|---|---|
| Out-of-place rebuild 经济线 | DiskANN rebuild 资源（1100 GB + 32 cores × 2 天）≈ 全集群 capacity 时不可行 |
| In-place skew 经济线 | SPANN+ 100 days 后 P99.9 4→10ms；不 rebalance 就退化 |
| In-place rebalance 经济线（cluster path） | SPFresh LIRE 仅 0.4% insertion 触发 rebalance，~4 GB RAM 持续低成本 |
| In-place merge 经济线（graph path） | FreshDiskANN StreamingMerge 5.25× faster than full DiskANN rebuild，~128 GB RAM 持续 |

## 关键洞察：update 是 system-level 而非 algorithm-level 决策

[per systems/diskann.md, systems/spann.md, systems/spfresh.md, systems/freshdiskann.md]

DiskANN 的 Vamana 算法本身**不预设** update 模型。**out-of-place 是 DiskANN 系统层选的**；FreshDiskANN 用同样 Vamana base 重选 in-place（FreshVamana + StreamingMerge）。

类似：SPANN 的 IVF + closure clustering 也不预设 update 模型。SPFresh 直接复用 SPANN 的 IVF + clustering，加 LIRE 协议**仅在 system 层让 IVF 变成 in-place**。

→ Update 策略是 **system layer 选择**，可独立于 algorithm 层。这意味着原则上 [Faiss](../systems/faiss.md) IVFPQ + LIRE-like 协议、Milvus segment 内 IVF 用 LIRE、HNSW + α-augmented update 等组合都可行——目前都没人做。

## 与现有 wiki Open Q 的关系（updated 2026-05-09）

本 topic 直接关闭多个 Open Q：

- ~~[systems/diskann.md §Open Questions]: "更新与删除：与 NSG 一样不支持增量"~~ → **graph 路径已解 via [FreshDiskANN](../systems/freshdiskann.md) + [FreshVamana](../concepts/freshvamana.md)**
- ~~[systems/faiss.md §Open Questions]: "FreshDiskANN is in development"~~ → **已 ingest，wiki 已覆盖**
- [systems/spann.md §Open Questions]: "数据漂移下的退化" → **SPFresh 已答（cluster path）**
- [concepts/hnsw.md §Open Questions]: "如何支持元素删除/更新而不退化图质量？" → **理论已答（α-augmented RobustPrune），工业未集成 HNSW 等价 patch**
- [concepts/nsg.md §Open Questions]: "不支持增量更新" → 同上
- [concepts/vamana.md §Open Questions]: "FreshDiskANN 是后继工作" → **已 ingest**

## 与 embedding-update-handling 的区分

**注意区别**：本 topic 讨论"**同 embedding 空间内的 vector update**"——insert / delete / modify 单个向量。**不**包括"**embedding model 升级**"（BERT→SBERT）情况——那是 vector space 整体迁移，所有索引必须重建，与 in-place vs out-of-place 正交。

**2026-05-11 update**：[Qdrant Collection Aliases](../systems/qdrant.md) [per sources/docs/qdrant/manage-data/collections.md "Collection aliases"] 是 **wiki 内首个明确的 production model migration tool**——通过 atomic alias swap 让新旧 embedding model 共存：

```
旧 collection `prod_v1` (旧 model) ← alias `prod` ← user queries
新 collection `prod_v2` (新 model) 后台 build + ingest re-embedded data
原子 swap: alias `prod` → `prod_v2`, 旧 collection delete/archive
```

工程便利显著（atomic switch + Qdrant Migration tool docker image），但**不解决跨 model embedding mapping 的 algorithm 问题**——新旧 model 的 vector space 仍然 incompatible，必须全量 re-embed 数据。**embedding-update-handling query 的 algorithm-level 解仍是 zero coverage**（19 个 ingest 后确认）。

详见 wiki 当前未覆盖 embedding-lifecycle 完整工程实践（双索引切换、increment patch、共享空间训练等）。

## Open Questions

- **HNSW / NSG α-augmented streaming variant**：理论上同 FreshVamana 思路（加 α > 1 RobustPrune）；hnswlib / nmslib 没人做——是 community PR 的 logical contribution
- **MIPS 任务的 in-place update**：[topics/mips-vs-l2-nn.md](./mips-vs-l2-nn.md) 与 [LIRE](../concepts/lire.md) / FreshVamana 的交叉未覆盖
- **混合架构**：cluster-based in-place + graph-based 静态的层叠？(SPANN 内 cluster 是 IVF——但每 cluster 内可用 graph)
- **跨 shard / 分布式 in-place**：[xu-2023-spfresh §6] / [singh-2021-freshdiskann §1] 都明示 future work；[Milvus](../systems/milvus.md) 多机 + LIRE / FreshDiskANN 是开放方向
- **Embedding 量化系数随 update 漂移**：PQ codebook 一开始训好，update 进来后量化误差累积——LIRE / FreshVamana 都不解决量化层问题。**[RaBitQ](../concepts/rabitq.md) 量化层带 sharp error bound 是否更适合 streaming**？理论上 yes 但未实证（RaBitQ 论文不涉及 streaming）
- **Graph path vs Cluster path 的 head-to-head benchmark**：FreshDiskANN 2021 + SPFresh 2023 时间错位，论文相互不直接对比；wiki 内 head-to-head 实证仍空白
- **Filter / multi-vector + streaming**：FilteredVamana / ACORN / NHQ 都不是 streaming-ready；FreshVamana + FilteredVamana 联合理论可行未实证
- **Block shuffling + streaming**：[Starling](../systems/starling.md) §7 提及 update 模型但未深入；FreshDiskANN + Block Shuffling 联合是 Zilliz/Milvus 团队 logical work（Starling 团队 vs FreshDiskANN 团队是不同 Microsoft 子团队）

## Cited Pages

- [systems/spfresh.md](../systems/spfresh.md)
- [systems/freshdiskann.md](../systems/freshdiskann.md)
- [systems/spann.md](../systems/spann.md)
- [systems/diskann.md](../systems/diskann.md)
- [systems/milvus.md](../systems/milvus.md)
- [systems/faiss.md](../systems/faiss.md)
- [concepts/lire.md](../concepts/lire.md)
- [concepts/freshvamana.md](../concepts/freshvamana.md)
- [concepts/hnsw.md](../concepts/hnsw.md)
- [concepts/nsg.md](../concepts/nsg.md)
- [concepts/vamana.md](../concepts/vamana.md)
