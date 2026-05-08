---
title: In-Place vs Out-of-Place Updates（向量索引更新策略）
type: topic
sources: [xu-2023-spfresh, chen-2021-spann, subramanya-2019-diskann, wang-2021-milvus, douze-2024-faiss-library]
related: [../systems/spfresh.md, ../systems/spann.md, ../systems/diskann.md, ../systems/milvus.md, ../systems/faiss.md, ../systems/pinecone.md, ../systems/analyticdb-v.md, ../systems/pase.md, ../concepts/lire.md, ../concepts/hnsw.md, ../concepts/nsg.md, ../concepts/product-quantization.md, ../concepts/pinecone-serverless-slabs.md, ../concepts/delta-consistency.md, ../concepts/vgpq.md]
created: 2026-05-07
updated: 2026-05-07
---

# In-Place vs Out-of-Place Updates

**TL;DR**: ANN 索引的 update 策略是 system-level 选择，与 index type 部分独立。**Out-of-place**（DiskANN streamingMerge / Faiss codebook freeze + global rebuild / Milvus LSM merge / Vearch tombstone + 周期 rebuild）以**周期成本换日常稳定**；**In-place**（[SPFresh](../systems/spfresh.md) [LIRE](../concepts/lire.md) / Vearch 部分支持）以**持续低成本换无 rebuild 高峰**。In-place 在 graph-based 索引上**目前未解**——只有 cluster-based（[SPANN](../systems/spann.md)-style）有 SPFresh LIRE 这样的成功案例。

## 问题陈述

ANN 索引的两类 update 模型：

- **Out-of-place**：用户更新累积到 secondary delta index → 周期"全局 rebuild"合并到 main index → 旧 index 析构
- **In-place**：用户更新直接修改主 index 数据结构 → 后台维护索引质量（split/merge/reassign 等）

**核心 trade-off**：
- Out-of-place：日常稳定，但周期 rebuild 资源峰值极高（1B 量级 1100 GB DRAM + 32 cores × 2 天 [per benchmarks/spfresh-vs-diskann-spann-update.md]），rebuild 期间 latency 跳水
- In-place：无 rebuild 高峰，但 index 数据结构持续承担更新成本，需要复杂 concurrency control + 形式化质量证明（如 [LIRE](../concepts/lire.md) NPA 收敛证明）

## 工业方案对比

| 方案 | 路线 | 索引类型 | 更新粒度 | 全局 rebuild 频率 | 资源峰值 | 数据漂移适应 |
|---|---|---|---|---|---|---|
| [DiskANN](../systems/diskann.md) `streamingMerge` | **Out-of-place** | Graph (Vamana) | 周期 batch | 1B SIFT 每 ~1-2 周（实测） | 1100 GB + 32 cores × 2d 或 64GB + 16 cores × 5d [per benchmarks/spfresh-vs-diskann-spann-update.md] | rebuild 后才修复 |
| **SPANN+**（modified [SPANN](../systems/spann.md)） | In-place append-only | IVF | 单向量 | 不 rebuild（实验对照） | 极低 | **不修复**（partition skew 累积，P99.9 4ms→>10ms over 100d） |
| [Faiss](../systems/faiss.md) IVFPQ | **Out-of-place** | IVF + PQ | 周期 batch | 完全 retrain codebook | 同 train 全套 | codebook freeze 后**完全不修复** |
| [HNSW](../concepts/hnsw.md) (Faiss / nmslib) | **Out-of-place** | Graph | add ✓ delete ✗ | rebuild 时全图重建 | 同 train | 仅 add 不 delete；图退化无修复 |
| [NSG](../concepts/nsg.md) | **Out-of-place** | Graph | 不支持任何增量 | rebuild | 同 train | rebuild 后才修复 |
| Vearch | In-place | Cluster (in-memory) | 单向量 + tombstone | 周期 GC + rebuild | 中 | tombstone GC，不 rebalance |
| ADBV / 早期 [Milvus](../systems/milvus.md) | **Out-of-place** | 多种 | delta index 累积 | 周期 merge | 中（rebuild 期 search 还要 main + delta） | rebuild 后才修复 |
| Milvus v2.x LSM | **Out-of-place + LSM** | Segment-based | 单向量 → segment | 后台 tiered merge | 中 | merge 仅按 size，不按数据分布 |
| **[SPFresh](../systems/spfresh.md) LIRE** | **In-place + 主动 rebalance** | IVF（[SPANN](../systems/spann.md) base） | 单向量 + 邻近触发 | **完全不需要** | **持续 10 GB + 2 cores**（1B SIFT） | **NPA 主动维持**，P99.9 ~4ms 稳定 100 天 |
| **[Pinecone Serverless slab](../systems/pinecone.md)** | **LSM-style + adaptive indexing** | Slab in object storage | 单向量 → memtable → flush → slab merge | **slab merge** 中（非全 rebuild） | docs 未公开数字（auto） | **adaptive**：merge 时自动从 fast indexing 升级到 sophisticated indexing |
| **[Manu (Milvus 2.x)](../systems/milvus.md)** | **Stream indexing + delta consistency τ** | Segment（512 MB default） + slice（10K vec temp IVF-FLAT） | growing segment 临时 index → sealed segment 完整 index | 周期 stream indexing（非全 rebuild） | NeurIPS 2021 winner SSD index | **delta consistency τ** 让 user 调"过期容忍度"——支持 strong / eventual / 中间任意点 |
| **[AnalyticDB-V (ADBV)](../systems/analyticdb-v.md)** | **Lambda streaming + batching 双索引** | Streaming HNSW (in-memory) + Batching VGPQ (Pangu) | 新数据进 streaming HNSW；周期 async merge 到 batching VGPQ + 重建 VGPQ | **Async merge** 中（不阻塞 query） | OLAP DB 路径 + 4 plan CBO | 与 Milvus LSM / Manu stream indexing 同代思路；但 streaming/batching 用**两种不同算法** (HNSW vs VGPQ) |
| **[PASE](../systems/pase.md)** | **PG 自身 OLTP transaction + WAL** | IVFFlat / HNSW page chain in PG | 直接走 PG INSERT/UPDATE/DELETE；index 与表数据用同一 PG transaction 原子；ACID 保证 | **PG 自身 vacuum + WAL replay** | OLTP RDBMS 路径 | 不需要自定 update strategy——**PG 内核自带的 MVCC 处理**；缺点是受限于 PG 单实例能力（million-scale）|

## 三种"In-Place"的差别

[per concepts/lire.md, systems/spfresh.md]：

1. **不修复型 in-place**（SPANN+ / Vearch）：单向量插入到现有 partition + tombstone delete；不 rebalance partition。代价：partition skew 累积导致 latency 与 recall 缓慢退化
2. **修复型 in-place**（SPFresh LIRE）：插入触发 split + 邻近 NPA 检查 + reassign；持续维持 well-partitioned 性质
3. **DBMS-style segment merge**（Milvus LSM）：实际是 out-of-place 的"周期化"——后台 tiered merge 替代周期 rebuild，但 segment 内 index 仍然冻结

只有第二种（修复型 in-place）能同时保 latency 稳定 + 资源低 + 不 rebuild。**目前 wiki ingest 的 source 中仅 [SPFresh](../systems/spfresh.md) 一个**。

## 为什么 graph-based 索引没有 in-place 解

[xu-2023-spfresh §1, §2.1] 显式分析：

- Graph index 每个 vertex 维护 K 个邻居 edge；插入新 vector 需识别它的 hundreds of neighbors，**每次查询整个 high-d 空间**
- Delete 更贵：需扫整个图找指向该 vector 的 in-edge
- 高维下"shortcut" 维护成本与图大小线性
- Cluster-based 索引天然便宜：插入 / 删除只动 1 个 partition 内数据 + centroid（如果 centroid 不变）

→ Graph-based ANN（HNSW / NSG / Vamana / DiskANN）的 in-place update 仍是开放问题。

## 三个明确的"经济阈值"

[per benchmarks/spfresh-vs-diskann-spann-update.md, xu-2023-spfresh Table 1]：

| 决策点 | 触发 |
|---|---|
| Out-of-place rebuild 经济线 | rebuild 资源（1100 GB + 32 cores × 2 天）≈ 全集群 capacity 时不可行；64 GB + 16 cores × 5 天降级也阻塞 update 5 天 |
| In-place skew 经济线 | SPANN+ 100 days 后 P99.9 4→10ms；不 rebalance 就退化 |
| In-place rebalance 经济线 | SPFresh LIRE 仅 0.4% insertion 触发 rebalance，持续低成本 |

## 关键洞察：update 是 system-level 而非 algorithm-level 决策

[per systems/diskann.md, systems/spann.md, systems/spfresh.md]：

DiskANN 的 Vamana 算法本身**不预设** update 模型——它只定义索引结构（α-controlled graph）。**out-of-place 是 DiskANN 系统层选的，不是 Vamana 算法本征要求**。

类似：SPANN 的 IVF + closure clustering 也不预设 update 模型。SPFresh 直接复用 SPANN 的 IVF + clustering，加 LIRE 协议**仅在 system 层让 IVF 变成 in-place**。

→ Update 策略是 **system layer 选择**，可独立于 algorithm 层。这意味着原则上 [Faiss](../systems/faiss.md) IVFPQ + LIRE-like 协议、Milvus segment 内 IVF 用 LIRE 等组合都可行——目前都没人做。

## 与现有 wiki Open Q 的关系

本 topic 直接关闭多个 Open Q：

- [systems/faiss.md §Open Questions]: "Faiss 当前 IVF / PQ 的 codebook 一旦 train 就冻结，long-running 索引如何 graceful 重新训练？"
  → SPFresh 给出 cluster-based 答案：centroid 保持 NPA 性质 + 增量 split/merge
- [systems/diskann.md §Open Questions]: "更新与删除：与 NSG 一样不支持增量"
  → 仍未解（graph 路线）
- [systems/spann.md §Open Questions]: "数据漂移下的退化：closure clustering 训练好后簇分配冻结，新加点需要重 cluster"
  → SPFresh 直接是该问题的答案
- [concepts/hnsw.md §Open Questions]: "如何支持元素删除/更新而不退化图质量？"
  → 仍未解
- [concepts/nsg.md §Open Questions]: "不支持增量更新（论文 §5 明确承认是未来工作）"
  → 仍未解

## 与 embedding-update-handling 的区分

**注意区别**：本 topic 讨论"**同 embedding 空间内的 vector update**"——insert / delete / modify 单个向量。**不**包括"**embedding model 升级**"（BERT→SBERT）情况——那是 vector space 整体迁移，所有索引必须重建，与 in-place vs out-of-place 正交。

详见 wiki 当前未覆盖 embedding-lifecycle 工程实践（双索引切换、increment patch、共享空间训练等）。

## Open Questions

- **Graph-based in-place update**：HNSW / Vamana / NSG 的 in-place 仍开放；FreshDiskANN [Singh 2021] 是后继工作（wiki 未 ingest）
- **MIPS 任务的 in-place update**：[topics/mips-vs-l2-nn.md](./mips-vs-l2-nn.md) 与 [LIRE](../concepts/lire.md) 的交叉未覆盖
- **混合架构**：cluster-based in-place + graph-based 静态的层叠？
- **跨 shard / 分布式 in-place**：[xu-2023-spfresh §6] 明示 future work；[Milvus](../systems/milvus.md) 多机 + LIRE 是开放方向
- **Embedding 量化系数随 update 漂移**：PQ codebook 一开始训好，update 进来后量化误差累积——LIRE 不解决量化层问题，仅 partition assignment

## Cited Pages

- [systems/spfresh.md](../systems/spfresh.md)
- [systems/spann.md](../systems/spann.md)
- [systems/diskann.md](../systems/diskann.md)
- [systems/milvus.md](../systems/milvus.md)
- [systems/faiss.md](../systems/faiss.md)
- [concepts/lire.md](../concepts/lire.md)
- [concepts/hnsw.md](../concepts/hnsw.md)
- [concepts/nsg.md](../concepts/nsg.md)
