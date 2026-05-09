---
title: LIRE（Lightweight Incremental REbalancing）
type: concept
sources: [xu-2023-spfresh, singh-2021-freshdiskann]
related: [../systems/spfresh.md, ../systems/spann.md, ../systems/freshdiskann.md, freshvamana.md, ../topics/in-place-vs-out-of-place-updates.md]
created: 2026-05-07
updated: 2026-05-09 (FreshVamana graph-path counterpart)
---

# LIRE

**TL;DR**: 用于 **cluster-based on-disk 向量索引**的轻量增量再平衡协议。核心是当 posting（簇）超长 split 时，**仅检查附近少量 posting 的 NPA（Nearest Posting Assignment）违反并 reassign**，而不全局重建。给出 2 个 reassign 必要条件 + cascading split-reassign 收敛形式化证明。**仅 0.4% 的插入触发 rebalancing**，平均 split 数 2，max cascading length 3。是 [SPFresh](../systems/spfresh.md) 系统的核心算法。[xu-2023-spfresh §3]

## 提出背景

Yuming Xu, Hengyu Liang, Jin Li 等（USTC + Microsoft Research Asia + Harvard），SOSP 2023。

针对的核心矛盾：现有 billion-scale ANN 系统都依赖**周期性全局 rebuild** 应对数据漂移：
- [DiskANN](../systems/diskann.md) `streamingMerge`：1B SIFT 需 1100 GB RAM × 2 天，rebuild 期间 P99.9 飙到 >20ms
- [SPANN](../systems/spann.md)：closure clustering 一旦训完冻结，update 累积导致 partition 不均
- Milvus / Vearch / ADBV：累积 delta + 周期 rebuild

LIRE 的洞察：**well-partitioned 向量索引上的小批量 update 通常只引发 local 范围的连锁修改**（"single vector update to a high-quality vector partition may only incur changes in itself and its neighboring partitions"）。如果能把 rebalance 限制在 local region，整个再平衡过程就轻量可承担。

## 三大设计目标

[xu-2023-spfresh §1 末]

1. **Search latency 短**：维持 partition size 均匀分布（split + merge 主动控制）
2. **Search accuracy 高**：识别**最小**的需 reassign 向量集（NPA 违反者）
3. **Negligible 前台干扰**：implementation 与 foreground search 解耦

## NPA（Nearest Posting Assignment）规则

[xu-2023-spfresh §2.2, §3.2]

cluster-based 索引的核心质量约束：**每个向量应分到其最近的 posting（簇）**——这是 "well-partitioned" 的定义性质。

```
∀ v ∈ X, posting(v) = argmin_{c ∈ Centroids} D(v, c)
```

NPA 保证 SPANN 类查询能用 centroid 距离剪枝（`Dist(q, c_ij) ≤ (1+ε) × Dist(q, c_i1)`）。

**NPA 违反**发生在 split 后：原 posting A 分裂为 A1、A2 后，邻近 posting B 中某些向量到 A1/A2 的距离可能比到 B 更近——这些向量需要 reassign。

## 5 基本操作

[xu-2023-spfresh §3.2]

### Insert / Delete（外部接口）

- **Insert**：把新向量直接加到 NPA-nearest posting（沿用 [SPANN](../systems/spann.md) 设计）
- **Delete**：tombstone 标记，最终 garbage collect 时清除

### Split / Merge / Reassign（内部接口）

- **Split**：当 posting 超过 length limit → 用 balanced clustering [SPANN §3.1] 切两 posting → 触发 reassign（在新 posting + 附近 posting 检查）
- **Merge**：当 posting 小于下界 → 与最近 posting 合并 → 触发 reassign（仅检查被合并 posting 的向量）
- **Reassign**：基于 2 必要条件移动 NPA 违反者

## 2 个 Reassign 必要条件

[xu-2023-spfresh §3.3, Eq 1 & 2]

split 触发的 reassign 检查范围分为**原 posting**和**附近 posting**两类：

### 条件 1（原 posting A_o 中的向量 v）

```
v 需 reassign 当 D(v, A_o) ≤ D(v, A_i)，∀ i ∈ {1, 2}
```

直觉：如果 v 到旧中心 A_o 比到两个新中心都更近，旧中心是更好的 representative——但 A_o 已被删除，不能用。**附近 posting 可能比新 posting 更接近**，所以需要检查。这是必要条件（而非充分条件）。

### 条件 2（附近 posting B 中的向量 v）

```
v 需 reassign 当 D(v, A_i) ≤ D(v, A_o)，∃ i ∈ {1, 2}
```

直觉：split 创了新中心 A_i；如果 v 到某个新中心比旧中心 A_o 更近，新中心可能是 v 的更好表示。需要进一步检查 v 当前 centroid B 是否仍优于 A_i。

### 为什么两个都是必要而非充分

避免每次更新都全数据集扫描。LIRE 仅扫"邻近 several A_o's nearest postings"——通常 ≤64 个——做 NPA-check。实测：5094 个候选 evaluated，仅 79 个真正 reassign。检查 + 真实修改的开销极低。

## Cascading Split-Reassign 收敛证明

[xu-2023-spfresh §3.4]

reassign 把向量 append 到新 posting → 新 posting 可能超长 → 触发新 split → 触发新 reassign → 可能级联无穷？

**形式化**：定义 index state = (C, M)，其中 C 是 centroid 集、M 是 vector→centroid 映射。给定 C，M 唯一决定。证明 split 引发的 |C| 变化序列收敛：

- 每次 split 删除 1 个旧 centroid + 加 2 个新 centroid → |C_{i+1}| = |C_i| + 1
- 经 N 次 split 后 |C_{i+N}| = |C_i| + N
- 由 NPA 性质 N ≤ |V| − |C|（vector 总数有限）
- 故 N 有限，cascading 必然终止

经验：max cascading length = 3（[xu-2023-spfresh §5.2]）。

## 关键性质

| 维度 | LIRE |
|---|---|
| 时间复杂度（每次 insert） | O(check K nearby postings × NPA 距离计算) ≈ 常数 |
| 空间复杂度 | + version map（1 byte/vector：7-bit version + 1-bit tombstone） |
| 收敛保证 | 形式化证明 |
| Reassign 检查范围 | nearest 64 postings（[xu-2023-spfresh Fig 11]） |
| 实测 rebalance 频率 | **0.4% of insertions** |
| 平均 split 数 / 操作 | 2 |
| Max cascading length | 3 |
| Merge 频率 | 0.1% of total updates |

## 与同类对比

| | LIRE | DiskANN streamingMerge | Vearch tombstone | Milvus LSM |
|---|---|---|---|---|
| 更新策略 | **In-place 增量** | **Out-of-place 全局 rebuild** | In-place + tombstone | LSM segment merge |
| 时机 | per-update 触发 | 周期性 | 周期 GC | 周期 merge |
| 资源峰值 | **极低**（10 GB + 2 cores） | 极高（1100 GB + 32 cores） | 中（rebuild 需要） | 中（merge 需要） |
| 数据 skew 适应 | **✓**（NPA 主动维持） | rebuild 后才修复 | ✗（只 GC，不 rebalance） | merge 仅按大小 |
| 形式化证明 | **✓** | — | — | — |
| 适用索引类型 | cluster-based（SPANN-style） | graph-based | cluster-based | DBMS 通用 |
| 详见 | 本 page | [systems/diskann.md](../systems/diskann.md) | [topics/in-place-vs-out-of-place-updates.md](../topics/in-place-vs-out-of-place-updates.md) | [systems/milvus.md](../systems/milvus.md) |

## 典型实现

- 作者实现：见 [systems/spfresh.md](../systems/spfresh.md)；构建在 [Microsoft SPTAG](https://github.com/microsoft/SPTAG) 之上
- LIRE 是 SPFresh 的核心算法层；工程包装是 Updater + Local Rebuilder + Block Controller 三组件 feed-forward pipeline

## Open Questions

- **适用范围限制**：LIRE 仅适用 cluster-based on-disk 索引（SPANN-style）。**graph-based** 索引（HNSW / Vamana / [DiskANN](../systems/diskann.md) 内存层）不适用——graph 边的高维更新成本根本不同。论文 §2 显式承认。
- **Reassign 范围 64 是经验值**：参数研究（Fig 11）显示 ≥64 收益 marginal；不同数据分布下最优范围未给理论指导
- **极端 skew 下的退化**：实测 0.4% 触发率假设 update 大致均匀；冷热极端 skew workload 下触发率与 cascading length 可能炸
- **MIPS 任务下未验证**：LIRE 的距离计算依赖 L2-style metric；MIPS（[topics/mips-vs-l2-nn.md](../topics/mips-vs-l2-nn.md)）下 NPA 是否仍合理？
- **分布式扩展**：[xu-2023-spfresh §6 Conclusion] 明示 "future distributed version"——多机场景下 cross-shard NPA 检查、reassign 跨网络成本未涵盖
- **与 graph-based 系统的混合**：LIRE 思路（"only nearby region needs check"）是否能迁移到 [Vamana](./vamana.md) 这类 graph？开放方向

Cited by: 待 query 引用
