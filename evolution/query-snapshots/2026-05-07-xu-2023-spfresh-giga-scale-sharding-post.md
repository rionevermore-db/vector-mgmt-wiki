---
query-key: giga-scale-sharding
query: "千亿/万亿向量在私有云（16 × 128U + 1TB RAM 节点，768-d，read-heavy，P99<50ms，周级别全量更新）下的分片策略？"
date: 2026-05-07
phase: post
ingest-context: xu-2023-spfresh
wiki-pages-total: 33
cited-pages: [systems/spfresh.md, concepts/lire.md, topics/in-place-vs-out-of-place-updates.md, benchmarks/spfresh-vs-diskann-spann-update.md, systems/spann.md, systems/diskann.md, systems/milvus.md, benchmarks/faiss-trillion-scale.md, topics/disk-vs-memory-ann.md]
cited-count: 9
---

# Post-snapshot (xu-2023-spfresh): giga-scale-sharding

## TL;DR (delta from milvus-docs post)

**周级全量更新假设**被 [SPFresh](../../systems/spfresh.md) **直接攻破**：原假设需要"双索引切换 + CDC + cutover"模式，rebuild 期间双倍内存/SSD。SPFresh 证明**完全不需要 rebuild** —— LIRE 协议持续 in-place 更新，1B 索引 100 days × 1% daily update 仅持续 10 GB memory + 2 cores（vs DiskANN rebuild 1100 GB + 32 cores × 2 天）。给定 16 × 1TB 私有云：**SPFresh-style in-place 更新让"周级全量更新"约束直接消失**——可以连续接收增量 update 不停服。

## Answer

### 周级别更新约束的消解（NEW，本次 ingest 关键发现）

[per benchmarks/spfresh-vs-diskann-spann-update.md, systems/spfresh.md]

之前的假设链：
1. 周级别全量更新 → 必须 rebuild
2. Rebuild 资源峰值 1100 GB + 32 cores × 2 天 → 16 节点全集群被占用
3. 必须双索引切换 + CDC + cutover → 内存/SSD 翻倍

**SPFresh 论证打破第 1 步**：
- LIRE [per concepts/lire.md] 仅 0.4% 的插入触发 rebalance
- 100 days 累计仅平均 split 数 2、max cascading length 3
- 持续 10 GB + 2 cores 维护，无 rebuild 高峰
- P99.9 全程稳定 ~4ms（vs DiskANN rebuild 时飙到 >20ms）

→ **周级全量更新约束完全可以重新表述为"周级 1% 增量"**——这正是 SPFresh 100 days 实测的 workload。

### 16 × 1TB 私有云的 SPFresh-based 部署（NEW）

如果不强制周级 rebuild：

**架构**：
- 16 节点跑 SPFresh 实例（每节点 1 SPFresh + NVMe SSD）
- Single-node SPFresh 已支持 SIFT1B 单节点 IOPS-bound（不是 memory/CPU bound）
- 千亿规模 = 100× SIFT1B → 需要 100 NVMe SSD，跨 16 节点 = 6+ NVMe/节点（合理 enterprise 配置）

**资源预算**：
- Memory：每节点 ~10 GB SPFresh（centroids + version map + Block Mapping） + 系统其他 → 远小于 1 TB 上限
- CPU：每节点 15 cores 饱和 IOPS（论文实测）；128 vCPU 节点跑 8× SPFresh shard 仍轻松
- SSD：每节点 6-10 NVMe，通过 K8s storage class 暴露

**未实测**：
- SPFresh **官方明示是单机系统** [per xu-2023-spfresh §6]
- 多节点跨 SSD 的 LIRE 协议（cross-shard NPA 检查）**尚未设计**
- 上述 16 节点部署需要在每节点跑独立 SPFresh + 上层 routing 层（[Milvus](../../systems/milvus.md) shard 模型类似）

### Read-heavy + P99 < 50ms 目标命中

SPFresh 实测：
- **P99.9 ~4-6 ms** during 100-day update [per benchmarks/spfresh-vs-diskann-spann-update.md]
- 1B stress test P99.9 ~5ms (uniform) / ~6.5ms (skew)
- 单节点 4K QPS search + 2K QPS update

**横向扩展到千亿**：if shard-level SPFresh 按 IOPS 平均，千亿延迟应保持 ms 级——远低于 50 ms 目标。

### 与之前两轮 ingest 的演进

| | wang-2021-milvus post | milvus-docs post | **xu-2023-spfresh post (NEW)** |
|---|---|---|---|
| 周级更新方案 | 双索引 + CDC + cutover | + LSM + function field（更易但仍 rebuild segment） | **完全不 rebuild**（LIRE in-place） |
| 1B index rebuild 资源 | 1100 GB + 32c × 2d | 同 | **不需要**（持续 10 GB + 2c） |
| P99 在 update 期 | rebuild 时 latency 跳 | rebuild 时 latency 跳 | **稳定** |
| 16 节点是否够 | 千亿可行但非常紧 | 同 | 千亿宽松，万亿仍未实测 |
| 万亿可行性 | 仅 Meta 1.5T 实测 ~1s | 同 | SPFresh 仍未测万亿 |

### 已知盲区

- **SPFresh 单机限制**：千亿/万亿需多机 SPFresh + routing；论文 §6 明示 future work
- **Pinecone pod-based 架构**：仍未覆盖
- **SPFresh + Milvus 集成**：理论可行（Milvus segment 内用 SPFresh），实际无人做
- **Burst 写场景（>1% daily）**：SPFresh 实验温和 1% daily；burst workload 下 LIRE 行为未量化
- **768-d 实测**：仍为 wiki 整体盲区
- **Embedding 升级触发的"假"全量更新**：不在 SPFresh 解决范围

## Cited Pages

- [systems/spfresh.md](../../systems/spfresh.md)
- [concepts/lire.md](../../concepts/lire.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
- [benchmarks/spfresh-vs-diskann-spann-update.md](../../benchmarks/spfresh-vs-diskann-spann-update.md)
- [systems/spann.md](../../systems/spann.md)
- [systems/diskann.md](../../systems/diskann.md)
- [systems/milvus.md](../../systems/milvus.md)
- [benchmarks/faiss-trillion-scale.md](../../benchmarks/faiss-trillion-scale.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
