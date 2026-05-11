---
title: 千亿/万亿向量 + 私有云 16 节点的分片策略
type: query
sources: [douze-2024-faiss-library, chen-2021-spann, subramanya-2019-diskann, xu-2023-spfresh, vespa-docs, gao-2024-rabitq]
related: [./index-architecture-global-vs-routed.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../topics/in-place-vs-out-of-place-updates.md, ../systems/spann.md, ../systems/diskann.md, ../systems/vespa.md, ../concepts/rabitq.md, ../benchmarks/faiss-trillion-scale.md, ../benchmarks/spann-vs-diskann-billion.md, ../benchmarks/spfresh-vs-diskann-spann-update.md]
created: 2026-05-11
updated: 2026-05-11
---

# 千亿/万亿向量 + 私有云 16 节点的分片策略

**Date**: 2026-05-11

**Question**:
当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？
Workload 假设：read-heavy，向量维度 768，原始数据周级别全量更新一次。**分级延迟目标**：千亿 P99 < 50ms，万亿 P99 < 1s（万亿一档对齐 Meta 1.5T 实测 ~1s 物理上限）。

## TL;DR

**100B 一档（P99 < 50ms）**：走 SPANN 风格"中心点驻内存 + posting list 驻 NVMe + 簇感知分片"，单机 ~6B 向量 + ~1ms 单节点检索，16 节点配合 closure + query-aware dispatch 可拿到 ≈ 30–50ms P99，**贴边可达**。**1T 一档（P99 < 1s）**：与 Meta 1.5T × 144d × 20 服务器 mmap ~1s 公开数字**同量级可达**，方案是 PCAR + SQ6 类极端压缩 + Meta 风格三阶段构建 + 16 节点 mmap。**周级全量重建**这一约束让 SPFresh/FreshDiskANN 的增量价值落空，反而正好对上 SPANN/DiskANN/Faiss IVF 的静态批构建路径。

## Answer

### 1. 物理约束先算账（768d 是关键变量）

| 项 | 100B | 1T |
|---|---|---|
| Raw float32 | 100B × 768 × 4 ≈ **307 TB** | **3.07 PB** |
| SQ8（每维 1 字节） | 76.8 TB | 768 TB |
| Meta 风格 PCAR72,SQ6 ≈ 54 B/vec [per [benchmarks/faiss-trillion-scale.md](../benchmarks/faiss-trillion-scale.md)] | **5.4 TB** ← 装进 16 TB 集群 RAM 都富余 | 54 TB ← 必须 mmap |
| 集群总 RAM | 16 TB | 16 TB |
| 集群总 vCPU | 2048 | 2048 |

→ **100B 在压缩后理论上"可以全 RAM"**（5.4 TB ≪ 16 TB）；**1T 即使压缩也必须 SSD/mmap**。这两档应当按两套打法对待，不能拿同一套蓝图硬套。

### 2. 100B 一档：推荐 SPANN 风格 + 簇感知分片

[per [systems/spann.md](../systems/spann.md) + [benchmarks/spann-vs-diskann-billion.md](../benchmarks/spann-vs-diskann-billion.md)]

**为什么不是 (a) 全局图**：全局 HNSW/NSG 在 1B+ 反复被证不可行；Taobao 2B NSG 也是 32 分区 [per [queries/index-architecture-global-vs-routed.md](./index-architecture-global-vs-routed.md), [concepts/nsg.md](../concepts/nsg.md)]。

**为什么是 SPANN 而不是 DiskANN**：在 1B SIFT/SPACEV1B/DEEP1B 三个数据集上，SPANN 在 90% recall 时**比 DiskANN 快 2×**（~1ms vs ~3-4ms）[per [benchmarks/spann-vs-diskann-billion.md](../benchmarks/spann-vs-diskann-billion.md)]；且 SPANN 内存预算只需 ~32 GB（centroids 占 ~16% N）vs DiskANN 64 GB（PQ codes）。16 节点 × 1 TB RAM 远远富余，可以把 centroids 完全装进 RAM。

**16 节点物理排布**：

| 层 | 配置 |
|---|---|
| 内存（每节点 1 TB） | 节点本地 SPTAG centroid 索引 + 100B × 16% = 16B 全局 centroids 的分片副本 + OS page cache |
| NVMe（假定数 TB/节点） | posting lists 全精度向量；每节点 ~6.25B vectors（100B/16） |
| 网络 | container/proxy 层路由 query 到候选 content node |

**分片策略**——直接抄 SPANN §4.3 实证 [per [systems/spann.md](../systems/spann.md) "分布式扩展"]：

| 分片方法 | 32 分区平均 dispatch 机器数 |
|---|---|
| Random partition | 32 全部 |
| Multi-constraint balanced clustering | 9 |
| + Closure assignment | 8 |
| **+ Query-aware pruning（完整 SPANN）** | **6.3** |
| **省 IO/计算** | **80.3%** |

→ **千万别用 hash/random sharding**。SPANN 用簇感知（HBC + closure）把"近邻数据共置"，单查询只命中约 1/5 节点。16 节点场景按比例外推（推断）单查询命中约 3-4 节点。

**P99 预算分摊（粗算）**：
- 单节点 SPANN 90% recall @ 1B ≈ 1ms；6.25B/节点理论上慢 2-3×（HBC 树深度增加）→ 2-3ms
- 16 节点 fan-out + 聚合 + 网络 RTT ≈ 5-10ms（推断私有云内）
- 总 P99 ≈ 30-50ms，**贴在目标线上**

### 3. 1T 一档：P99 < 1s 与 Meta 1.5T 实测同量级可达

[per [benchmarks/faiss-trillion-scale.md](../benchmarks/faiss-trillion-scale.md)] Meta 1.5T 部署是 wiki 内唯一"万亿 scale 公开数字"：

| 项 | Meta 实测 |
|---|---|
| 规模 | 1.5T × 144d（比 768d 低 5.3×） |
| 编码 | `PCAR72,SQ6` → 54 字节/向量 |
| Coarse quantizer | 10M-centroid HNSW |
| 构建 | 三阶段：2000 shard over IDs → 100 shard over lists → 20 服务器 mmap 83 TiB |
| 单查询（中央 1 机） | **~12s** |
| 单查询（分散到 20 机） | **~1s** |

→ Meta 在 20 服务器 mmap 架构下 1T 是 **~1s**——**正好对齐我们的 P99 < 1s 目标**。我们的 16 节点 + 768d 比 Meta 的 20 节点 + 144d 略 disadvantaged（向量大 5.3×、节点少 25%），但同量级。

**1T 推荐方案**（基于 Meta 三阶段套用）：
- 编码：PCAR + SQ6 至 ~54 B/vec，或 RaBitQ 类 D-bit 量化（96 B/vec，给出 unbiased error bound）
- Coarse quantizer：10M-centroid HNSW（同 Meta）
- 三阶段构建：2000 shard over IDs → 100 shard over lists → 16 节点 mmap
- 16 节点 mmap 总存储需 54 TB——按每节点 ~4 TB NVMe 计算可行
- 单查询 dispatch：full fan-out（不像 100B 一档能用 SPANN 簇感知裁剪）

> **风险点**：(1) 16 节点 vs Meta 20 节点，并行度 -20% 但向量 5.3× 大，单查询可能滑到 1.2-1.5s。需 query path 优化 + 缓存命中。(2) Weekly full rebuild 时间未公开计算公式——Meta 三阶段在数百节点上跑，外推到 16 节点的 build budget 是独立约束（见 §5）。

### 4. 量化层选择（768d 比 SIFT 128d 棘手）

[per [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md) + [topics/index-selection.md](../topics/index-selection.md) + [concepts/rabitq.md](../concepts/rabitq.md)]

| 编码 | 100B 节点单机存储 | 是否够装内存 | recall 上限 |
|---|---|---|---|
| SQ8（≈ 768 B/vec） | 4.8 TB/节点 | 否（>1 TB RAM） | 接近 100% |
| Meta-style PCAR_to_72,SQ6（≈ 54 B/vec） | 337 GB/节点 | 装进 RAM | 已知 Meta 用过 |
| RaBitQ（D bits = 96 B/vec）[per [concepts/rabitq.md](../concepts/rabitq.md)] | 600 GB/节点 | 装进 RAM | unbiased + sharp error bound |
| 不压缩 + SSD 全精度（SPANN 原生路径） | 19 TB/节点 disk | NVMe 容量边界 | 接近 100% |

**768d 对 SPANN 的影响**——必须注意 [per [systems/spann.md](../systems/spann.md) Open Question]：SPANN 原 paper posting list 上限是 12 KB（byte）/ 48 KB（float）；128d float → 16 vec/list；**768d float → 16 vec/list**（48 KB / 3072 B）。posting list 拉长 → centroids 数量爆炸 → 内存压力增大。**论文未在 768d 区间评估，是已知 Open Question**。

### 5. 周级全量重建：不要选错算法

[per [topics/in-place-vs-out-of-place-updates.md](../topics/in-place-vs-out-of-place-updates.md)]

**周级全量** = 静态批处理。这意味着：

- **不需要** [SPFresh](../systems/spfresh.md) 的 in-place LIRE 协议（其价值在每日 1% 增量、避免 1100 GB + 32 cores × 2 天 rebuild [per [benchmarks/spfresh-vs-diskann-spann-update.md](../benchmarks/spfresh-vs-diskann-spann-update.md)]）
- **不需要** [FreshDiskANN](../systems/freshdiskann.md) 的 StreamingMerge
- **完美匹配** SPANN / DiskANN / Faiss IVF 的静态构建——Meta 三阶段构建模型直接套用：2000 shard over IDs 并行 build → 100 shard over lists merge → 16 节点 mmap

但要点：周级全量本身就是一个 **几小时到几十小时** 的批作业。Meta 三阶段需要"数百节点 × 64 核 × 256 GB"做阶段 1 + 2 [per [benchmarks/faiss-trillion-scale.md](../benchmarks/faiss-trillion-scale.md)]。**这 16 节点能否 7 天内跑完一次 1T 重建本身是个独立约束**——wiki 未直接覆盖具体构建时间公式。

### 6. 最终决策表

| 维度 | 100B（已知 fit） | 1T（wiki 内无先例） |
|---|---|---|
| 拓扑 | SPANN routed (HBC + closure + query-aware) over 16 nodes | Meta 三阶段 mmap，但需要再压一档量化 |
| 编码 | PCAR + SQ6（54 B/vec, 装入 RAM）或 SPANN 原生全精度（SSD） | PCAR + SQ6 最低限度，再尝试 RaBitQ |
| 索引 | SPANN | IVF + HNSW coarse（Meta） |
| 节点角色 | Stateless proxy ×2-4 + Content node ×12-14（参考 Vespa 拆分 [per [systems/vespa.md](../systems/vespa.md)]） | 同上 |
| 单查询 dispatch | ≈ 4 节点（SPANN 80% 省） | full fan-out 16 节点 |
| P99 目标可达性 | **30-50ms 贴边可达**（目标 50ms） | **~1s 同量级可达**（目标 1s，对齐 Meta 1.5T 实测） |
| 重建路径 | 周级全量批构建，SPFresh/FreshDiskANN 价值=0 | 同 |

## Cited Pages

- [queries/index-architecture-global-vs-routed.md](./index-architecture-global-vs-routed.md)
- [topics/disk-vs-memory-ann.md](../topics/disk-vs-memory-ann.md)
- [topics/index-selection.md](../topics/index-selection.md)
- [topics/in-place-vs-out-of-place-updates.md](../topics/in-place-vs-out-of-place-updates.md)
- [systems/spann.md](../systems/spann.md)
- [systems/diskann.md](../systems/diskann.md)
- [systems/vespa.md](../systems/vespa.md)
- [concepts/rabitq.md](../concepts/rabitq.md)
- [benchmarks/faiss-trillion-scale.md](../benchmarks/faiss-trillion-scale.md)
- [benchmarks/spann-vs-diskann-billion.md](../benchmarks/spann-vs-diskann-billion.md)
- [benchmarks/spfresh-vs-diskann-spann-update.md](../benchmarks/spfresh-vs-diskann-spann-update.md)

## Follow-up Questions

- **768d 区间的 SPANN posting list 设计**：12/48 KB 上限是 128d 经验值，768d 应该重选哪个？
- **16 节点 weekly full rebuild 的具体时间**：Meta 三阶段在数百节点上跑，外推到 16 节点未公开
- **1T @ 1s 在 16 节点 + 768d 的精确边界**：Meta 1.5T × 144d × 20 节点 = ~1s；我们 1T × 768d × 16 节点的 latency 上界没有公开实测，需要 1.2-1.5s 的安全 budget
- **SPANN @ 768d 的实测延迟曲线**：原 paper 仅测到 128d byte / 96d float
- **Vespa SPANN OSS 实现在 16 节点 + 1 TB RAM 私有云下的实测**：docs 描述机制但无具体表
- **与 [index-architecture-global-vs-routed.md](./index-architecture-global-vs-routed.md) 的关系**：该 query 锁定"(a) global vs (c) routed"逻辑架构二选一；本 query 锁定 (c) 路径下"16 节点 × 1 TB RAM 的物理排布 + 分片策略 + P99 budget 分摊"——前者是 *topology choice*，本 query 是 *hardware allocation*

## 演化说明

本 query 是 [evolution/tracking-queries.md](../evolution/tracking-queries.md) 的 #1 主追踪 query；后续 ingest 会通过 `evolution/query-snapshots/<date>-<ingest-context>-giga-scale-sharding-pre|post.md` 留下 pre/post 快照（reads-only history），而本归档文件作为**活的最佳答案**会被更新（每次有实质性新 source 后）。
