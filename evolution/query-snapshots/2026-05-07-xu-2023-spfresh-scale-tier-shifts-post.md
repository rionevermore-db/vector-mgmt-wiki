---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-07
phase: post
ingest-context: xu-2023-spfresh
wiki-pages-total: 33
cited-pages: [topics/index-selection.md, topics/disk-vs-memory-ann.md, topics/in-place-vs-out-of-place-updates.md, systems/spfresh.md, systems/diskann.md, systems/spann.md, systems/milvus.md, concepts/lire.md, benchmarks/spfresh-vs-diskann-spann-update.md]
cited-count: 9
---

# Post-snapshot (xu-2023-spfresh): scale-tier-shifts

## TL;DR (delta from milvus-docs post)

**新增第 5 个质变点（update strategy 维度）**：从"out-of-place 周期 rebuild" 到 "in-place 增量"是任何规模档下都可触发的**事件型质变**。该质变与 N 正交，但**经济阈值随 N 急剧扩大**——million-scale 上 rebuild 资源可承受，billion-scale 上 1100 GB DRAM + 32 cores × 2 天属于近不可行，**触发 SPFresh-style 重设计的必要性随规模指数上升**。

## Answer

### 五个质变点（updated）

| # | 质变点 | 维度 | 触发条件 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | ~10M → 1M+ |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | ~1B（DRAM → SSD/分布式） |
| 3 | 单机 SSD 触顶 | 规模 + 存储 | ~百亿+（SSD → 分布式 mmap） |
| 4 | library → DBMS（cloud-native） | 工程形态（与 N 正交） | dynamic data + 分布式 + filter 需求 |
| **5（NEW）** | **out-of-place rebuild → in-place 增量** | **update strategy（与 N 正交但 N 决定经济性）** | **持续 update + 资源约束触顶** |

### 第 5 质变点的经济阈值（NEW）

[per benchmarks/spfresh-vs-diskann-spann-update.md, xu-2023-spfresh Table 1]：

| 规模 | Out-of-place rebuild 资源 | 是否可承受 | In-place 解 |
|---|---|---|---|
| 10M | 数 GB DRAM × 数小时 | ✓（轻松） | 价值小 |
| 100M | 数十 GB × 数小时 | ✓（容易） | 价值中等 |
| **1B** | **1100 GB DRAM + 32 cores × 2 天**（DiskANN） | **⚠️ 边界**（需大型集群） | **SPFresh 价值显著** |
| 1B（受限资源） | 64 GB + 16 cores × 5 天 | ✗（5 天阻塞 update） | **SPFresh 必须** |
| 10B+ | 推断 10× rebuild 资源 | ✗ | **未实测，但显然必须 in-place** |
| 100B+ | 不可行 | ✗ | **未实测；可能需要分布式 SPFresh-style** |

→ Rebuild 资源呈 super-linear 增长，**1B 是经济阈值**——这是 SPFresh 论文聚焦的恰好规模。

### 与 update 维度耦合的子质变（NEW）

[per topics/in-place-vs-out-of-place-updates.md]：

**5a. Cluster-based 路径上的 update 解**（已实证）：
- SPANN 静态 → SPANN+ in-place 不 rebalance（skew 累积） → **SPFresh = SPANN + LIRE**（in-place + rebalance）
- 触发条件：cluster-based 索引 + 持续 update + 资源约束

**5b. Graph-based 路径上的 update 仍未解**（开放）：
- HNSW add ✓ delete ✗
- NSG 不支持任何增量
- DiskANN streamingMerge（out-of-place）
- 触发条件：graph-based 索引 + 持续 update → 仍需 rebuild 或换索引类型

**5c. Quantization + update 耦合**（开放）：
- PQ codebook freeze + update 累积失真
- LIRE 假设全精度，融合 PQ 是否破坏 NPA 未知

### 不算质变（参数微调）

- LIRE reassign range 64 vs 128（marginal）
- 前/后台 thread 比 2:1 vs 1:1
- 同 update 模型下不同 split limit / merge threshold

### 与之前两轮 ingest 的演进

| | wang-2021 post | milvus-docs post | **xu-2023-spfresh post (NEW)** |
|---|---|---|---|
| 质变点数量 | 4 | 6（含 DBMS 内部 4a/4b） | **7**（+ 第 5 质变 + update 子维度） |
| Out-of-place vs in-place 维度 | 隐含 | 隐含 | **显式分轴** |
| 经济阈值 1B | 估算 | 估算 | **实证（DiskANN 1100 GB × 2d）** |
| Graph-based update | 未触及 | 未触及 | **明确仍开放** |

### 已知盲区

- **10B+ 规模 update 实证**：SPFresh 实测到 1B；超规模未测
- **多机 SPFresh-style**：跨 shard LIRE
- **Burst update（>1% daily）**：SPFresh 实验温和
- **Embedding 升级触发的"假"全量更新**：与 vector-level update 区分但未解决

## Cited Pages

- [topics/index-selection.md](../../topics/index-selection.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
- [systems/spfresh.md](../../systems/spfresh.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [concepts/lire.md](../../concepts/lire.md)
- [benchmarks/spfresh-vs-diskann-spann-update.md](../../benchmarks/spfresh-vs-diskann-spann-update.md)
